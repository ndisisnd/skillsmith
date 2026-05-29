# Sub-agent contracts for agent-evaluate

Defines the invocation interface, args, and expected output shape for the two sub-agents
that agent-evaluate dispatches.

---

## agent-audit

### Invocation

Use the **Skill tool** (not the Agent tool):

```
Skill({ skill: "agent-audit", args: "<skill_path>" })
```

`skill_path` must be the full relative path including trailing slash, e.g. `.claude/skills/cook/`.

### Args

| Arg | Format | Required |
|-----|--------|----------|
| skill_path | `.claude/skills/<name>/` | Yes |

`agent-audit` runs its own check-selection step (Step 2/5) during execution. In **flash mode**,
pre-answer that step by prefixing args with `mode=flash` — the agent will run only `lint` and
skip test/grade/benchmark to keep the run short. In **comprehensive mode**, let the agent ask
the user normally or pass `mode=comprehensive` to run all checks.

### Output shape

agent-audit writes its artefacts to `<skill_path>/run/run-[n]/` and renders an inline report:

```
## Audit report — <skill_name> — run-[n]
### Evals       — table: id | prompt excerpt | verdict | assertions passed/total
### Grading     — pass_rate, total_assertions, human_review_pending
### Lint        — table: severity | rule | location | finding
### Optimise    — improved yes/no, val_pass_rate, iterations
### Benchmark   — tokens mean, duration mean, partial flag
### Errors      — any run_errors
## Summary      — 3 bullets: overall verdict, highest-priority fix, cost signal
```

Collect the rendered inline report as `audit_output`. The primary structured file is
`<skill_path>/run/run-[n]/feedback.json`.

---

## agent-quality

### Invocation

Use the **Skill tool** (not the Agent tool):

```
Skill({ skill: "agent-quality", args: "<skill_name>" })
```

Pass the bare skill **name** (not the full path), e.g. `cook`. agent-quality constructs
`skill_path = .claude/skills/<skill_name>/` internally.

### Args

| Arg | Format | Required |
|-----|--------|----------|
| skill_name | directory name under `.claude/skills/` | Yes |
| test_input | free-text string | No (auto-generated if absent) |

In **flash mode**, omit `test_input` — the agent uses a short auto-generated input.
In **comprehensive mode**, consider passing a representative `test_input` that exercises
the skill's primary claim.

### Output shape

agent-quality renders an inline side-by-side comparison and writes
`<skill_path>/run/run-[n]/quality-[n].json`:

```json
{
  "test_input": "...",
  "vanilla_prompt": "...",
  "vanilla_output": "...",
  "skill_output": "...",
  "scoring": {
    "completeness": { "vanilla": 0, "skill": 0 },
    "structure":    { "vanilla": 0, "skill": 0 },
    "actionability":{ "vanilla": 0, "skill": 0 }
  },
  "grade": "skill_better | marginal | opus_alone_better",
  "recommendation": "skill_better | marginal | opus_alone_better",
  "recommendation_rationale": "..."
}
```

Collect the rendered inline report as `quality_output`. The grade emoji mapping:
- `skill_better` → 🟢 Skill is clearly better
- `marginal` → 🟡 Marginal — skill adds limited value over Opus alone
- `opus_alone_better` → 🔴 Opus alone is better

---

## Running both in parallel

When both agents are selected, send **two Skill tool calls in a single message**. Both
agents write to their own `run/run-[n]/` directories and do not share state, so they are
safe to run concurrently.

```
Skill({ skill: "agent-audit",   args: ".claude/skills/<name>/" })
Skill({ skill: "agent-quality", args: "<name>" })
```

Collect `audit_output` and `quality_output` once both complete, then proceed to Step 5/5.
