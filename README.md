# Prompt Optimizer

A battle-tested methodology for autonomous, metric-driven LLM prompt optimization. Uses iterative refinement with tournament selection to systematically improve any prompt — extraction, generation, classification, code review, or anything else that takes text in and produces structured output.

Developed from a real optimization campaign that improved a knowledge graph extraction prompt from **0.42 to 0.90 composite score** (a 114% improvement) over ~50 iterations.

## The Problem

Manual prompt engineering is slow, inconsistent, and doesn't scale. You tweak one thing, break another. You can't tell if you're making progress because you're not measuring. You end up in a loop of "try something, eyeball it, try something else."

## The Solution

This skill provides a complete framework for **autonomous prompt optimization** — define metrics, build a scoring function, and let a tournament-based optimizer iterate on your prompt while you sleep.

### Key Ideas

- **Metric-driven**: Define what "good" means as computable scores. Composite scoring with weighted metrics tells the optimizer exactly what to improve.
- **Tournament selection**: Each iteration generates 5 candidate edits with different strategic lenses (examples, counter-examples, simplification, restructuring, etc.), screens them cheaply on 1 test case, and validates the winner on 3 more.
- **Structured edits**: The optimizer proposes surgical find/replace edits — not full prompt regeneration. LLMs can't faithfully reproduce large prompts, but they can make 1-3 targeted changes.
- **Two-phase optimization**: Phase 1 maximizes quality. Phase 2 reduces cost while maintaining a quality floor.

## Architecture

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) — three files drive the loop:

```
project/
├── optimize/
│   ├── program.md           # Optimizer agent instructions (the "brain")
│   ├── scoring.py           # Composite scoring function (pure, no I/O)
│   ├── error_analyzer.py    # Systematic error pattern detection
│   └── experiment_runner.py # I/O layer — runs the prompt on test cases
├── optimize.py              # Main CLI — orchestrates the loop
├── prompt_versions/         # Auto-versioned prompt snapshots
├── test_inputs/             # Your test cases
└── your_prompt.md           # The prompt being optimized
```

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                    OPTIMIZATION LOOP                         │
│                                                              │
│  1. Sample test cases (random subset per iteration)          │
│  2. Analyze errors from last round's outputs                 │
│  3. Run optimizer tournament:                                │
│     ├── 5 candidates × different diversity lenses (parallel) │
│     ├── Screen all 5 on 1 test case (cheap)                  │
│     └── Validate winner on 3 test cases (expensive)          │
│  4. Accept if composite score improves                       │
│  5. Every 5 accepts → full evaluation on ALL test cases      │
│  6. Check convergence / phase transition                     │
│                                                              │
│  Phase 1: Maximize quality → target composite > 0.95         │
│  Phase 2: Minimize cost   → maintain quality > 0.93          │
└─────────────────────────────────────────────────────────────┘
```

## Workflow

### Phase 0: Setup

Define what "good" means for your prompt:

1. **What does the prompt do?** (extraction, generation, classification, etc.)
2. **What are your quality criteria?** Push for specifics — not "accurate" but "extracts all entities with correct types."
3. **What test cases do you have?** 5-25 representative inputs. More is better.
4. **What model runs the prompt?** Determines cost estimation.
5. **Is there post-processing?** If yes, apply it before scoring — don't optimize for issues already handled downstream.

### Phase 1: Define Metrics

Design a composite scoring function with weighted metrics grouped by priority:

| Category | Weight | Examples |
|----------|--------|---------|
| Primary quality | 50-60% | Density, field coverage, predicate diversity |
| Secondary quality | 20-30% | Evidence linkage, completeness |
| Connectivity | 10-15% | Cross-references, source attribution |
| Format compliance | 5-10% | Schema validity (lowest if coercion exists) |

See [`references/scoring-design.md`](references/scoring-design.md) for the full guide with code examples for 8 metric types (density, coverage, validity, diversity, completeness, linkage, consistency, quantitative context).

### Phase 2: Build Supporting Modules

- **`scoring.py`** — Pure function: takes output dict, returns `(composite_score, per_metric_dict)`. Both bounded [0, 1].
- **`error_analyzer.py`** — Detects systematic patterns (vocabulary violations, missing fields, distribution skew) and returns them ranked by `severity × frequency`.
- **`experiment_runner.py`** — Thin I/O wrapper: runs the prompt on test cases in parallel, handles failures gracefully, tracks cost.

### Phase 3: Write the Optimizer Agent

The `program.md` file is the system prompt for the LLM that proposes edits. It receives the current prompt, scores, error patterns, and edit history — and outputs 1-3 surgical find/replace edits as JSON.

See [`references/program-template.md`](references/program-template.md) for the full template with customization guide.

### Phase 4: Run the Loop

```bash
python optimize.py \
    --max-iterations 200 \
    --sample-size 3 \
    --candidates 5 \
    --version-prefix v2
```

See [`references/runner-template.md`](references/runner-template.md) for the complete loop implementation.

## Diversity Lenses

Tournament selection uses 10 strategic lenses to prevent the optimizer from getting stuck proposing the same edit repeatedly:

| Lens | Strategy |
|------|----------|
| `structural_rewrite` | Reorganize prompt sections for clarity |
| `example_driven` | Add concrete input→output examples |
| `counter_example` | Show common mistakes and correct versions |
| `constraint_tightening` | Clarify ambiguous language with examples (not "MUST" mandates) |
| `checklist_approach` | Describe desired output properties (not self-verification scans) |
| `negative_space` | Make implicit assumptions explicit |
| `cross_metric_synergy` | One edit that improves multiple metrics |
| `simplification` | Remove contradictory or redundant rules |
| `workflow_reframing` | Guide the model through a specific process |
| `weakest_link_focus` | Target the lowest-scoring neglected metric |

See [`references/diversity-lenses.md`](references/diversity-lenses.md) for detailed descriptions, win conditions, and escalation behavior.

## Best Practices

These are hard-won lessons from production optimization campaigns:

### Metric Design
- **Metric choice dominates everything.** Invest time here before writing any optimization code.
- **Weight by downstream impact.** If 55% of value comes from claim quality, weight it 55%.
- **Don't score what coercion handles.** If post-processing fixes invalid values, don't penalize the prompt.

### Prompt Editing
- **Structured edits > full regeneration.** LLMs hallucinate when reproducing large prompts.
- **1-3 edits per iteration.** More makes attribution impossible.
- **Examples beat rules.** One concrete example is worth three abstract instructions.

### Optimization Loop
- **Tournament selection is essential.** Single-candidate hill climbing has >80% rejection rate.
- **Random sampling per iteration.** Don't run on ALL test cases every time — sample 3-4. Full evaluation every 5 accepts catches overfitting.
- **High-water mark tracking.** The best prompt may come from a rejected iteration due to sampling noise.

### Cost
- **Use cheap models for extraction, expensive for optimization.** Prompt runs on Haiku; optimizer runs on Sonnet.
- **Track cost at every step.** Know your per-iteration cost before starting an overnight run.
- **Version everything.** Save prompt + scores for every iteration.

## Battle-Tested Lessons

These patterns were discovered during optimization campaigns and are now encoded into the templates. Ignoring them leads to wasted iterations.

### What Works (Highest to Lowest Impact)

| Technique | Impact | Example |
|-----------|--------|---------|
| **WRONG/RIGHT counter-examples** | +0.15 composite in 1 iteration | `WRONG: "certainty": "high" for single-condition observation` / `RIGHT: "medium"` |
| **Soft nudges + inline examples** | +0.05-0.10 per metric | `"Provide whenever possible — e.g., {"name": "BMP4", "ontology_id": "UniProt:P21275"}"` |
| **Target distribution statements** | +0.05-0.15 for diversity metrics | `"A typical output should contain 45-60% high, 25-35% medium, 10-20% low"` |
| **Extending vocabulary tables** | +0.03-0.05 per metric | Adding off-vocabulary predicates to a REMOVED predicates mapping table |

### What Fails (Confirmed Harmful)

| Technique | Observed Impact | Why It Fails |
|-----------|----------------|-------------|
| **"REQUIRED" language for uncertain fields** | -0.13 composite regression | Model fabricates values or gets confused trying to comply |
| **Self-verification checklists** | -0.53 composite regression | Model outputs prose commentary about the scan instead of structured output |
| **Hard multi-criterion gates** | -0.10 composite regression | Too restrictive; model ignores the gate or over-applies it |
| **`--json-schema` CLI flag** | Silent data loss | Strips response entirely when validation fails, returning empty string |

### JSON Enforcement for CLI Runners

When using `claude -p` or similar CLI wrappers, the CLI injects its own system prompt (~31K tokens), diluting your JSON output instructions. Fix: append a JSON enforcement suffix to the END of the user message (after input text). See [`references/runner-template.md`](references/runner-template.md) for the full pattern with retry.

## Installation

### As a Claude Code Skill

Copy the skill directory to your Claude Code skills path:

```bash
cp -r prompt-optimizer ~/.claude/skills/prompt-optimizer
```

The skill will trigger when you say things like "optimize my prompt", "tune this prompt", "my prompt isn't working well", or "set up an autoresearch loop."

### Standalone

The reference files in `references/` contain complete templates you can use independently:

- [`references/scoring-design.md`](references/scoring-design.md) — How to design metrics
- [`references/program-template.md`](references/program-template.md) — Optimizer agent prompt template
- [`references/diversity-lenses.md`](references/diversity-lenses.md) — Strategic diversity catalog
- [`references/runner-template.md`](references/runner-template.md) — Full optimization loop template
- [`references/dispatch-template.md`](references/dispatch-template.md) — Subagent dispatch prompt template (for multi-agent architectures)

## Origin

This methodology was developed during the [AutoReview](https://github.com/mcleanT/AutoReview) project — an autonomous pipeline for generating scientific review papers. The knowledge graph extraction prompt was optimized from a 0.42 composite score to 0.90 over multiple campaigns using this exact approach:

- **19 weighted metrics** across 4 categories (claim quality, evidence quality, cross-paper connectivity, format validity)
- **Tournament selection** with 5 parallel candidates and 10 diversity lenses
- **Two-phase optimization** (quality maximization → cost reduction)
- **25-paper test corpus** with section-aware truncation and production coercion

The methodology generalizes to any prompt-based task with measurable quality criteria.

## License

MIT
