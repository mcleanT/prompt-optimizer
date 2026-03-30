# Optimization Loop Template (optimize.py)

Template for the main CLI entrypoint that orchestrates the iterative prompt optimization loop. Customize the sections marked with `{{placeholders}}`.

## Architecture Overview

```
optimize.py (this file)
    ├── Loads prompt + test inputs
    ├── Runs baseline evaluation
    └── For each iteration:
        ├── analyze_errors() → error patterns
        ├── run_optimizer_tournament() → N candidate prompts
        ├── Screen candidates on 1 test input (parallel)
        ├── Validate best candidate on 3 test inputs
        ├── Accept/reject based on composite score
        ├── Every 5 accepts: full evaluation on ALL inputs
        └── Check convergence / phase transition
```

## Template Script Structure

### Constants

```python
# Paths
PROMPT_PATH = Path("path/to/your/prompt.md")
VERSIONS_DIR = Path("prompt_versions/")
LOG_PATH = Path("optimize/optimization_log.json")
PROGRAM_PATH = Path("optimize/program.md")

# Optimization parameters
CONVERGENCE_THRESHOLD = 0.002   # Stop when improvement < this
MAX_CONSECUTIVE_REJECTS = 25    # Stop after this many rejects in a row
PHASE1_TARGET = 0.95            # Quality target to trigger Phase 2
PHASE2_QUALITY_FLOOR = 0.93     # Minimum quality during cost optimization
MAX_CHANGE_RATIO = 1.0          # Max fraction of lines that can change (1.0 = disabled)
```

### score_all — Batch Scoring

```python
def score_all(outputs: list[dict], rapid: bool = False) -> tuple[float, dict[str, float]]:
    """Average composite and per-metric across successful outputs.

    Outputs with _error key are excluded so transient failures
    don't poison the score.
    """
    successful = [o for o in outputs if not o.get("_error")]
    if not successful:
        return 0.0, {name: 0.0 for name in METRIC_WEIGHTS}

    composites, per_metric_accum = [], {n: [] for n in METRIC_WEIGHTS}
    for output in successful:
        composite, metrics = score_output(output, rapid=rapid)
        composites.append(composite)
        for name, val in metrics.items():
            per_metric_accum[name].append(val)

    return (
        sum(composites) / len(composites),
        {n: sum(v) / len(v) for n, v in per_metric_accum.items()},
    )
```

### call_optimizer — Single Optimizer Call

```python
def call_optimizer(
    current_prompt: str,
    metrics: dict[str, float],
    composite: float,
    errors: list,
    history: list[dict],
    phase: str = "quality",
    iteration: int = 0,
    consecutive_rejects: int = 0,
    strategy_index: int | None = None,
) -> tuple[str, str]:
    """Call the optimizer agent to propose structured edits.

    Returns (new_prompt, summary).
    """
    system_prompt = PROGRAM_PATH.read_text()

    # Serialize top 8 error patterns
    error_dicts = [
        {"category": e.category, "description": e.description,
         "severity": round(e.severity, 3), "frequency": e.frequency,
         "examples": e.examples[:2]}
        for e in errors[:8]
    ]

    # Build context
    context = {
        "composite_score": round(composite, 4),
        "per_metric_scores": {k: round(v, 4) for k, v in metrics.items()},
        "metric_weights": METRIC_WEIGHTS,
        "error_patterns": error_dicts,
        "edit_history": history[-5:],  # Last 5 iterations only
        "phase": phase,
    }

    # Select diversity lens
    strat_idx = strategy_index if strategy_index is not None else iteration
    strategy = DIVERSITY_STRATEGIES[strat_idx % len(DIVERSITY_STRATEGIES)]
    diversity_block = f"\n## Optimization Lens: {strategy['lens']}\n\n{strategy['hint']}\n"

    # Escalation on consecutive rejects
    if consecutive_rejects >= 5:
        recent_failures = [
            h.get("optimizer_summary", "unknown")
            for h in history[-consecutive_rejects:]
            if h.get("outcome", "").startswith("rejected")
        ]
        diversity_block += (
            f"\n## IMPORTANT: {consecutive_rejects} consecutive rejects\n\n"
            "The following approaches ALL FAILED — do NOT repeat them:\n"
            + "\n".join(f"  - {s}" for s in recent_failures[-5:])
            + "\n\nTry a fundamentally different strategy.\n"
        )

    # Compose user message
    user_message = (
        "## Current Prompt\n\n```markdown\n" + current_prompt + "\n```\n\n"
        "## Current Scores and Error Analysis\n\n```json\n"
        + json.dumps(context, indent=2) + "\n```\n"
        + diversity_block + "\n"
        "Propose 1-3 targeted find/replace edits. Output ONLY the JSON object."
    )

    # Call optimizer model (Sonnet-class for reasoning)
    # Adapt this to your LLM calling pattern
    result = call_llm(
        system_prompt=system_prompt,
        user_message=user_message,
        model="sonnet",  # Use a strong model for optimization reasoning
    )

    # Parse JSON response (with brace-slice fallback)
    edits_obj = parse_json_with_fallback(result)
    edits = edits_obj.get("edits", [])
    summary = edits_obj.get("summary", "")

    if not edits:
        raise ValueError("Optimizer returned no edits.")

    # Apply edits
    new_prompt = current_prompt
    applied = 0
    for edit in edits:
        find_str = edit.get("find", "")
        replace_str = edit.get("replace", "")
        if find_str and find_str in new_prompt:
            new_prompt = new_prompt.replace(find_str, replace_str, 1)
            applied += 1

    if applied == 0:
        raise ValueError("No edits could be applied (find strings not found).")

    return new_prompt, summary
```

### run_optimizer_tournament — Parallel Candidates

```python
def run_optimizer_tournament(
    current_prompt, metrics, composite, errors, history,
    phase="quality", iteration=0, consecutive_rejects=0, n_candidates=5,
) -> list[dict]:
    """Run N optimizer calls in parallel with different lenses.

    Returns list of {new_prompt, summary, lens, strategy_index}.
    """
    n_strats = len(DIVERSITY_STRATEGIES)
    if n_candidates <= n_strats:
        indices = random.sample(range(n_strats), n_candidates)
    else:
        indices = list(range(n_strats)) + random.choices(
            range(n_strats), k=n_candidates - n_strats
        )

    candidates = []
    with ThreadPoolExecutor(max_workers=n_candidates) as executor:
        futures = {
            executor.submit(
                call_optimizer, current_prompt, metrics, composite,
                errors, history, phase, iteration, consecutive_rejects, idx
            ): idx
            for idx in indices
        }
        for future in as_completed(futures):
            idx = futures[future]
            try:
                new_prompt, summary = future.result()
                candidates.append({
                    "new_prompt": new_prompt,
                    "summary": summary,
                    "lens": DIVERSITY_STRATEGIES[idx % n_strats]["lens"],
                    "strategy_index": idx,
                })
            except Exception:
                pass  # Log and skip failed candidates

    return candidates
```

### Main Loop — The Core Algorithm

```python
def main():
    # --- Configuration ---
    # Parse args: --max-iterations, --sample-size, --candidates, --rapid, etc.

    # --- Load test inputs ---
    all_inputs = load_test_inputs()  # Your loading function

    # --- Load current prompt ---
    current_prompt = PROMPT_PATH.read_text()

    # --- Baseline evaluation (on ALL inputs) ---
    baseline_outputs = run_batch(current_prompt, all_inputs)
    baseline_composite, baseline_metrics = score_all(baseline_outputs)
    save_version(current_prompt, "v1.0_baseline", baseline_metrics, baseline_composite)

    # --- Tracking ---
    best_prompt = current_prompt
    best_composite = baseline_composite
    best_metrics = baseline_metrics
    last_outputs = baseline_outputs
    history = []
    accepted_count = 0
    consecutive_rejects = 0
    accepted_since_full_eval = 0

    # High-water mark (captures best prompt even across rejected iterations)
    hwm_composite = best_composite
    hwm_prompt = best_prompt
    hwm_iteration = 0

    phase = "quality"

    # --- Optimization loop ---
    for iteration in range(1, max_iterations + 1):

        # (a) Sample test inputs for this iteration
        random.shuffle(all_inputs)
        screen_input = all_inputs[0]
        validation_inputs = all_inputs[1:1 + sample_size]

        # (b) Analyze errors from last outputs
        errors = analyze_errors(last_outputs)
        if not errors:
            print("Converged — no error patterns detected.")
            break

        # (c) Run optimizer tournament
        candidates = run_optimizer_tournament(
            best_prompt, best_metrics, best_composite, errors,
            history, phase, iteration, consecutive_rejects, n_candidates,
        )
        if not candidates:
            consecutive_rejects += 1
            continue

        # (d) Screen: test each candidate on 1 input (parallel)
        screen_results = []
        with ThreadPoolExecutor(max_workers=len(candidates)) as executor:
            futures = {
                executor.submit(run_batch, c["new_prompt"], [screen_input]): i
                for i, c in enumerate(candidates)
            }
            for future in as_completed(futures):
                idx = futures[future]
                outputs = future.result()
                s_composite, s_metrics = score_all(outputs)
                screen_results.append({
                    "idx": idx, "composite": s_composite,
                    "metrics": s_metrics, "outputs": outputs,
                })

        # (e) Pick best screened candidate
        screen_results.sort(key=lambda r: r["composite"], reverse=True)
        best_screen = screen_results[0]
        best_cand = candidates[best_screen["idx"]]

        # Quick reject: if screen score <= baseline, skip validation
        if best_screen["composite"] <= best_composite:
            consecutive_rejects += 1
            # Update HWM if screen was still the best we've seen
            if best_screen["composite"] > hwm_composite:
                hwm_composite = best_screen["composite"]
                hwm_prompt = best_cand["new_prompt"]
                hwm_iteration = iteration
            history.append({
                "iteration": iteration, "outcome": "rejected_no_improvement",
                "old_composite": best_composite,
                "new_composite": best_screen["composite"],
                "optimizer_summary": best_cand["summary"],
                "lens": best_cand["lens"], "phase": phase,
            })
            continue

        # (f) Validate: test winner on additional inputs
        val_outputs = run_batch(best_cand["new_prompt"], validation_inputs)
        val_composite, val_metrics = score_all(val_outputs)

        # Update HWM
        if val_composite > hwm_composite:
            hwm_composite = val_composite
            hwm_prompt = best_cand["new_prompt"]
            hwm_iteration = iteration

        # (g) Accept/reject (phase-aware)
        accepted = False
        if phase == "quality":
            accepted = val_composite > best_composite
        else:  # cost phase
            if val_composite < PHASE2_QUALITY_FLOOR:
                accepted = False  # Quality dropped below floor
            elif new_cost < best_cost:
                accepted = True   # Cost improved
            elif val_composite > best_composite:
                accepted = True   # Quality improved, cost neutral

        if not accepted:
            consecutive_rejects += 1
            history.append({
                "iteration": iteration,
                "outcome": "rejected_no_improvement",
                "old_composite": best_composite,
                "new_composite": val_composite,
                "optimizer_summary": best_cand["summary"],
                "lens": best_cand["lens"], "phase": phase,
            })
            if consecutive_rejects >= MAX_CONSECUTIVE_REJECTS:
                break
            continue

        # (h) Accept!
        best_prompt = best_cand["new_prompt"]
        best_composite = val_composite
        best_metrics = val_metrics
        last_outputs = val_outputs
        accepted_count += 1
        accepted_since_full_eval += 1
        consecutive_rejects = 0

        save_version(best_prompt, f"v{iteration}", val_metrics, val_composite)

        history.append({
            "iteration": iteration, "outcome": "accepted",
            "old_composite": best_composite - (val_composite - best_composite),
            "new_composite": val_composite,
            "delta": val_composite - best_composite,
            "optimizer_summary": best_cand["summary"],
            "lens": best_cand["lens"], "phase": phase,
        })

        # (i) Full evaluation checkpoint every 5 accepts
        if accepted_since_full_eval >= 5:
            full_outputs = run_batch(best_prompt, all_inputs)
            full_composite, full_metrics = score_all(full_outputs)
            best_composite = full_composite  # Full eval is authoritative
            best_metrics = full_metrics
            accepted_since_full_eval = 0

            # Phase transition check
            if phase == "quality" and full_composite > PHASE1_TARGET:
                phase = "cost"
                print(f"PHASE TRANSITION: Quality {PHASE1_TARGET} reached!")

        # (j) Convergence check
        delta = val_composite - best_composite
        if delta < CONVERGENCE_THRESHOLD and phase == "quality":
            print(f"Converged: delta {delta:.4f} < {CONVERGENCE_THRESHOLD}")
            break

    # --- Final evaluation with HWM prompt ---
    final_outputs = run_batch(hwm_prompt, all_inputs)
    final_composite, final_metrics = score_all(final_outputs)

    if final_composite > baseline_composite:
        PROMPT_PATH.write_text(hwm_prompt)
        print(f"Updated prompt. Improvement: {final_composite - baseline_composite:+.4f}")

    # Save optimization log (append to existing)
    save_log(history, baseline_composite, final_composite)
```

## Key Design Decisions

### Why Tournament Selection?
Single-candidate hill climbing accepts only ~15-20% of iterations. Tournament selection with N=5 candidates raises this to ~40-50% by exploring diverse approaches per iteration. The cost is 5x optimizer calls (cheap — Sonnet) but only 1x validation extractions (expensive — the actual prompt execution).

### Why Random Sampling Per Iteration?
Running on ALL test inputs every iteration is expensive. Random 4-input sampling (1 screen + 3 validation) reduces per-iteration cost by 80%+ while maintaining statistical signal. Full evaluations every 5 accepts catch overfitting.

### Why High-Water Mark?
The best prompt may appear in an iteration where screening passes but validation fails due to sampling noise. HWM captures this — the final evaluation uses the HWM prompt, not just the last accepted one.

### Why Separate Screen and Validation?
Screening on 1 input is a cheap filter — eliminates clearly bad candidates before committing to expensive 3-input validation. This cuts per-iteration validation cost by ~60%.

### Why Append-Only Log?
Every iteration's outcome, metrics, cost, and optimizer summary are logged. This enables post-hoc analysis: which lenses work best, where does the optimizer stagnate, what's the cost curve. Never delete or overwrite log entries.

## CLI Arguments Template

```python
parser = argparse.ArgumentParser()
parser.add_argument("--max-iterations", type=int, default=200)
parser.add_argument("--sample-size", type=int, default=3,
                    help="Validation inputs per iteration")
parser.add_argument("--candidates", type=int, default=5,
                    help="Parallel optimizer candidates per tournament")
parser.add_argument("--rapid", action="store_true",
                    help="Aggressive input truncation for faster iterations")
parser.add_argument("--version-prefix", type=str, default="v2")
parser.add_argument("--skip-baseline", type=str, default=None,
                    help="Path to existing baseline scores JSON")
```
