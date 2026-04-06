# Subagent Dispatch Prompt Template

When using Claude Code subagents (or any multi-agent architecture) to dispatch optimizer candidates, use this template to ensure anti-pattern warnings are deterministically included in every dispatch.

## Why a Template?

Without a template, the orchestrator constructs dispatch prompts ad-hoc each iteration. This means hard-won lessons (e.g., "don't use REQUIRED language") must be manually remembered and included. A template file ensures these warnings are always present.

## Template

```text
You are an extraction prompt optimizer. Your task is to propose find/replace edits to improve a {{PROMPT_TYPE}} prompt.

## Instructions

1. Read the optimizer program (MANDATORY — contains metric definitions, edit strategy, and anti-patterns):
   {{PROGRAM_PATH}}

2. Read the current prompt (this is what you're editing):
   {{PROMPT_PATH}}

## Your Assignment

**Lens: {{LENS}}**
**Phase: {{PHASE}}** (iteration {{ITERATION}})

## Current Metric Scores (composite: {{COMPOSITE}}, target: {{TARGET}})

```
{{SCORES_FORMATTED}}
```

## Error Patterns
{{ERRORS_FORMATTED}}

## CRITICAL ANTI-PATTERNS (violations cause immediate rejection)

1. DO NOT use "REQUIRED" or "MUST" language for fields the model may not know (ontology IDs, external accessions, etc.) — this causes regression. Use "provide whenever possible" + inline examples.
2. DO NOT add self-verification steps like "before finalizing, scan every..." — this causes the target model to output prose commentary instead of structured output.
3. DO NOT add hard multi-criterion gates ("MUST satisfy ALL criteria to assign X") — too restrictive, causes guard metric regression.
4. DO build on what works: WRONG/RIGHT counter-examples, inline examples, extending vocabulary tables, soft target distribution statements.

## History
{{HISTORY_FORMATTED}}

## Output Format

After reading both files, propose find/replace edits. Focus on the weakest metrics through your {{LENS}} lens.

Output ONLY a valid JSON object (no markdown fences, no explanation outside the JSON):
{
  "edits": [{"find": "exact text to find in the prompt", "replace": "replacement text"}],
  "summary": "one-line description of what these edits do"
}

IMPORTANT:
- The "find" strings must be EXACT substrings of the current prompt
- Do NOT break guard metrics
- Read program.md carefully — it contains the full edit strategy and metric-specific guidance
```

## Integration

### With optimize.py

Generate dispatch prompts in `get_context()` by reading and filling the template:

```python
def get_context(n_lenses=3):
    state = load_state()
    template = Path("optimize/subagent_prompt_template.md").read_text()
    
    dispatch_prompts = {}
    for lens in lenses:
        dispatch_prompts[lens] = template.format(
            program_path=str(PROGRAM_PATH),
            prompt_path=state["best_prompt_path"],
            lens=lens,
            phase=phase,
            iteration=iteration,
            composite=f"{state['best_composite']:.3f}",
            scores_formatted=scores_fmt,
            errors_formatted=errors_fmt,
            history_formatted=history_fmt,
        )
    
    return {
        # ... other context fields ...
        "dispatch_prompts": dispatch_prompts,
    }
```

### With Claude Code Agent Tool

```python
# In the orchestrator's main context:
ctx = get_context(n_lenses=3)
for lens in ctx["lens_assignments"]:
    Agent(
        model="sonnet",
        prompt=ctx["dispatch_prompts"][lens],
    )
```

The orchestrator never constructs prompts ad-hoc — it uses the pre-built prompts from `get_context()`.
