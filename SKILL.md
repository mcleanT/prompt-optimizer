---
name: prompt-optimizer
description: Systematically optimize any LLM prompt through autonomous iterative improvement. Use when the user wants to improve prompt quality, set up an optimization loop, tune a prompt for better outputs, benchmark prompt variants, or says "optimize my prompt", "improve this prompt", "my prompt isn't working well", "tune this prompt", "autoresearch loop", "prompt engineering". Also use when the user has a prompt producing inconsistent or low-quality outputs and wants a disciplined, metric-driven approach rather than ad-hoc tweaking.
---

# Prompt Optimizer

A methodology for autonomous, metric-driven prompt optimization using iterative refinement with tournament selection. Based on the Karpathy autoresearch pattern — proven to improve prompt quality by 30-50% in production systems.

## When to Use This

- A prompt produces inconsistent or low-quality outputs
- You need measurable improvement, not vibes-based tweaking
- The prompt is complex enough that manual iteration is slow
- You want to optimize for cost while maintaining quality
- You have (or can create) test cases to evaluate against

## Core Architecture: The Three-File Pattern

Every prompt optimization project uses three files:

| File | Role | Who writes it |
|------|------|---------------|
| `program.md` | Optimizer agent instructions — tells the LLM how to propose improvements | You (with this skill's template) |
| The artifact | The prompt being optimized | Exists already (user's prompt) |
| `optimize.py` | Runner script — orchestrates the loop, scoring, and version control | You (with this skill's template) |

**Supporting modules** (created alongside the runner):
- `scoring.py` — Composite scoring function (pure, no I/O)
- `error_analyzer.py` — Systematic error pattern detection (pure, no I/O)
- `experiment_runner.py` — I/O layer that executes the prompt on test cases

## Workflow

### Phase 0: Setup Interview

Before writing any code, establish these with the user:

1. **What is the prompt doing?** (extraction, generation, classification, summarization, etc.)
2. **What does "good" look like?** This determines your metrics. Push for specifics:
   - Not "accurate" → "extracts all entities with correct types"
   - Not "well-written" → "uses active voice, <200 words, includes specific examples"
3. **What test cases exist?** You need 5-25 representative inputs. More is better for statistical power, but even 3 can bootstrap the process.
4. **What model runs the prompt?** (determines cost estimation and context limits)
5. **Is there a production post-processing step?** If so, apply it before scoring — don't optimize for issues already handled downstream.
6. **What's the budget?** Each iteration costs N extractions × per-call cost. Tournament selection with 5 candidates on 4 test cases = ~20 LLM calls per iteration.

### Phase 1: Define Metrics

This is the most important step. Bad metrics waste every subsequent iteration.

Read `references/scoring-design.md` for the full guide. Key principles:

**Score what matters for the downstream use case, not format compliance.**
If a post-processing step fixes formatting issues, don't waste optimization budget on format metrics. Weight claim quality > evidence quality > connectivity > format validity.

**Every metric must be:**
- Computable from the output alone (no human judgment in the loop)
- Bounded [0, 1] for composability
- Weighted by importance to the use case

**Metric design template:**
```python
METRIC_WEIGHTS: dict[str, float] = {
    # --- Primary quality (50-60% of composite) ---
    "metric_a": 0.15,  # Most important aspect
    "metric_b": 0.12,  # Second most important
    # --- Secondary quality (20-30%) ---
    "metric_c": 0.08,
    # --- Format/structure (10-20%) ---
    "metric_d": 0.05,
}
# Weights MUST sum to 1.0
```

**Common metric patterns:**
- **Density**: Output count in target range (penalize both too few and too many)
- **Coverage**: Fraction of items with required field populated
- **Diversity**: Shannon entropy of categorical distribution (normalized to [0,1])
- **Validity**: Fraction matching controlled vocabulary
- **Completeness**: Average fraction of required sub-fields populated per item

### Phase 2: Build the Scoring Function

Create `scoring.py` as a pure function — no I/O, no side effects:

```python
def score_output(data: dict) -> tuple[float, dict[str, float]]:
    """Score a single prompt output against quality metrics.

    Returns:
        (composite_score, per_metric_dict) — both values in [0, 1].
    """
```

Key design rules:
- Return both composite AND per-metric breakdown (the optimizer needs per-metric to know what to fix)
- Support a `rapid` mode that excludes metrics requiring expensive computation
- Use target ranges with linear ramp-up and penalty for overshooting (not just "more is better")

### Phase 3: Build the Error Analyzer

Create `error_analyzer.py` — detects systematic patterns across a batch of outputs:

```python
@dataclass
class ErrorPattern:
    category: str       # e.g., "missing_field", "invalid_value", "low_density"
    description: str    # Human-readable description with statistics
    severity: float     # 0-1, how bad this pattern is
    frequency: int      # How many times it occurred
    examples: list[str] # 2-3 concrete examples from the outputs

def analyze_errors(outputs: list[dict]) -> list[ErrorPattern]:
    """Detect systematic errors, sorted by severity × frequency descending."""
```

The error analyzer bridges the gap between metrics (which say "what's wrong") and the optimizer (which needs to know "why it's wrong" to propose fixes). Include:
- Vocabulary violations (using invalid values)
- Missing required fields
- Distribution skew (one category dominating)
- Structural problems (missing linkages, incomplete records)
- Domain-specific anti-patterns

### Phase 4: Write the Optimizer Agent Prompt (program.md)

Read `references/program-template.md` for the full template. The optimizer agent receives:
1. The current prompt text
2. Composite score + per-metric breakdown
3. Ranked error patterns with examples
4. History of past edit attempts and outcomes

And outputs: **1-3 surgical find/replace edits** as JSON.

Critical design decisions baked into the template:
- **Structured edits, not regeneration.** LLMs cannot faithfully reproduce large prompts. Find/replace edits are verifiable and bounded.
- **Priority ordering.** The optimizer attacks the highest-impact metrics first.
- **History awareness.** Failed approaches are explicitly listed so the optimizer doesn't repeat them.
- **Diversity lenses.** Each iteration gets a different strategic lens to prevent stagnation. See `references/diversity-lenses.md`.

### Phase 5: Build the Experiment Runner

Create `experiment_runner.py` — thin I/O wrapper:

```python
def run_single(prompt_text: str, input_text: str, timeout: int = 600) -> dict:
    """Execute the prompt on one input and return parsed output."""

def run_batch(prompt_text: str, inputs: list[dict], max_workers: int = 5) -> list[dict]:
    """Run on all inputs in parallel via ThreadPoolExecutor."""
```

Key patterns:
- **Parallel execution** with `ThreadPoolExecutor` — each call is independent I/O
- **Graceful failure handling** — failed calls return empty results with `_error` key, never crash the batch
- **Cost estimation** — track input/output tokens and estimated cost per call
- **JSON parsing with fallback** — direct parse → brace slice → truncation repair
- **Production coercion** — apply any post-processing pipeline before returning results
- **Timeout per call** — prevent hung calls from blocking the batch

### Phase 6: Wire the Optimization Loop

Read `references/runner-template.md` for the full template. The main loop:

```
for each iteration:
    1. Sample test cases (random subset per iteration to reduce cost)
    2. Analyze errors from last round's outputs
    3. Run optimizer tournament (N candidates, different lenses, parallel)
    4. Screen: test each candidate on 1 input (parallel)
    5. Pick best screened candidate
    6. Validate: test winner on 3 additional inputs
    7. Accept/reject based on composite improvement
    8. If accepted: update best prompt, reset reject counter
    9. Every 5 accepts: full evaluation on ALL test cases
    10. Check convergence / phase transition
```

**Tournament selection** is the key innovation over naive hill climbing:
- Launch N=5 optimizer candidates in parallel, each with a different diversity lens
- Screen all 5 on 1 test case (cheap — 5 LLM calls)
- Only validate the winner on 3 more test cases (expensive — 3 calls)
- Per-iteration cost: ~5 optimizer + 5 screen + 3 validation = 13 calls
- Solves the 80%+ rejection rate of single-candidate approaches

**Two-phase optimization:**
- **Phase 1 (Quality):** Maximize composite score until target reached (e.g., > 0.95)
- **Phase 2 (Cost):** Minimize per-call cost while maintaining quality floor (e.g., > 0.93)

### Phase 7: Run and Monitor

```bash
python optimize.py \
    --max-iterations 200 \
    --sample-size 3 \
    --candidates 5 \
    --version-prefix v2
```

Monitor for:
- **Consecutive rejects > 10:** The optimizer is stuck. The loop auto-escalates diversity pressure, but if rejects hit 25, it stops.
- **Phase transition:** Quality target reached → switches to cost reduction automatically.
- **High-water mark:** The globally best prompt is tracked even across rejected iterations. The final output uses the HWM prompt, not just the last accepted one.
- **Full evaluation checkpoints:** Every 5 accepted iterations, runs on ALL test cases (not just the sample) to catch overfitting to the sample.

### Phase 8: Analyze Results

After the loop completes:
1. Compare baseline vs final composite score
2. Review per-metric improvements (which metrics improved most?)
3. Check the optimization log for patterns (which lenses worked best?)
4. Run a final full evaluation on the HWM prompt
5. If quality target not reached, consider: expanding test corpus, redesigning metrics, or restructuring the prompt architecture

## Best Practices (Hard-Won Lessons)

### Metric Design
- **Metric choice dominates.** Switching from word-overlap to embedding similarity changed scores 5x without changing the system. Invest time here.
- **Weight by downstream impact.** If 55% of value comes from claim quality, weight it 55%.
- **Don't score what coercion handles.** If a post-processing pipeline fixes invalid values, don't penalize the prompt for producing them.

### Prompt Editing
- **Structured edits > full regeneration.** LLMs hallucinate when asked to reproduce large prompts. Find/replace is verifiable.
- **1-3 edits per iteration.** More than that makes it impossible to attribute improvement to specific changes.
- **Add, don't remove.** Prefer inserting clarifications after existing text. Only modify text that's actively causing confusion.
- **Examples beat rules.** One concrete input→output example is worth three abstract instructions.

### Optimization Loop
- **Tournament selection is essential.** Single-candidate hill climbing has >80% rejection rate. N=5 candidates with different lenses solves this.
- **Random sampling per iteration.** Don't run on ALL test cases every iteration — sample 3-4 to reduce cost. Full evaluation every 5 accepts catches overfitting.
- **High-water mark tracking.** The best prompt may come from a rejected iteration (screen passed, validation failed due to sampling noise). Track it.
- **Consecutive reject escalation.** After 5 rejects, feed failed approaches back to the optimizer with "DO NOT repeat these." After 25, stop — the prompt needs structural changes, not more tweaking.

### Test Corpus
- **Filter to representative inputs.** Exclude edge cases that produce unrepresentative outputs (e.g., review papers when optimizing for primary research extraction).
- **Minimum 5 test cases.** Fewer than 5 and per-iteration scores are too noisy.
- **Include hard cases.** The optimizer should be tested against difficult inputs, not just easy ones.

### Cost Management
- **Track cost at every step.** Know your per-iteration cost before starting an overnight run.
- **Use cheap models for extraction, expensive models for optimization.** The prompt runs on Haiku; the optimizer that proposes edits runs on Sonnet.
- **Rapid mode for early iterations.** Truncate inputs aggressively during exploration; switch to full inputs for final validation.
- **Batch API for production.** Optimizer iterations use real-time API; production deployments use batch API at 50% discount.

### Infrastructure
- **Version every iteration.** Save prompt + scores as `v{N}.md` + `v{N}_scores.json`. You will want to diff versions later.
- **Append-only optimization log.** JSON log of every iteration's outcome, metrics, cost, and optimizer summary. Essential for post-hoc analysis.
- **Unbuffered stdout.** Set `PYTHONUNBUFFERED=1` — overnight runs need real-time log visibility.
- **JSON output format from CLI.** Text output gets truncated on large responses; JSON output preserves full content.

## Scaffolding Generation

When the user is ready to start, generate the full file structure:

```
project/
├── optimize/
│   ├── __init__.py
│   ├── scoring.py          # Composite scoring (from Phase 2)
│   ├── error_analyzer.py   # Error pattern detection (from Phase 3)
│   ├── experiment_runner.py # I/O layer (from Phase 5)
│   └── program.md          # Optimizer agent prompt (from Phase 4)
├── optimize.py              # Main CLI entrypoint (from Phase 6)
├── prompt_versions/         # Auto-created: version snapshots
├── test_inputs/             # User's test cases
└── tests/
    └── test_optimizer/
        ├── test_scoring.py
        └── test_error_analyzer.py
```

Generate scoring.py and error_analyzer.py with domain-specific metrics based on the setup interview. Generate program.md from the template in `references/program-template.md`. Generate optimize.py from the template in `references/runner-template.md`.

## Reference Files

- `references/scoring-design.md` — Deep guide to metric design with examples for common prompt types
- `references/program-template.md` — Full template for the optimizer agent prompt
- `references/diversity-lenses.md` — Catalog of 10 diversity strategies with usage guidance
- `references/runner-template.md` — Full template for the optimization loop script
