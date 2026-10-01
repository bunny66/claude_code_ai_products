# [Service name] — service context

> Workspace: [workspace-name]
> Skills: [resolved skill list, e.g. python, pytorch, mlflow]
> Last updated: [date]
> Maintainer: [team or person]
>
> SCOPE OF THIS FILE — read before editing:
> - Generic technology rules live in `skills/` (one file per technology). Do NOT
>   repeat them here — this file records only what is specific to THIS service.
> - Service relationships, endpoints, events, env var declarations, guardrail rules,
>   and workspace boundary rules live in `architecture.yml`. This file explains and localises them; it
>   does not redeclare them.
> - The most valuable section is §3 Deviations: it is the only legitimate way for
>   a service to override a skill's default rule.

---

## 1. Responsibility

[One paragraph. What responsibility does this component own? What is it the
authoritative source of truth for? What does it explicitly NOT own?]

Example:
> Owns feature generation and training data publication for the demand-forecasting
> workflow. It is authoritative for the versioned feature dataset, but does not own
> model training, evaluation, or inference serving.

---

## 2. Structure snapshot

[The actual top-level layout of this service — brief, one level of nesting max.
This is a map for orientation, not documentation of generic framework structure
(the framework skill covers that). Regenerate via /update-service-context when it
changes materially.]

```
src/
├── ingestion/        # source loading and validation
├── features/         # transformations and feature definitions
├── pipelines/        # runnable workflows
└── tests/            # unit and data-contract checks
```

Notable locations:
- [anything a newcomer wouldn't guess — e.g. "cron jobs live in src/schedulers/,
  not inside feature modules"]

---

## 3. Deviations from skill defaults

[THE key section. Each resolved skill defines default rules; list here every place
this service intentionally differs, with the reason. /implement applies skill rules
EXCEPT where overridden here. If there are no deviations, state "None" explicitly —
an empty section is ambiguous, "None" is a claim.]

| Skill rule (default) | This service does | Why |
|----------------------|-------------------|-----|
| [e.g. pytorch: checkpoint format] | Uses the project's versioned checkpoint wrapper | Shared inference runtime contract |
| [e.g. python: dependency management] | Uses the workspace lockfile and pinned environment | Reproducible training runs |

---

## 4. Service-specific conventions

[Conventions that are real in this service but too local for a skill file —
naming quirks, folder rules, patterns unique to this codebase. Keep short; if a
convention applies to every service using a technology, move it to the skill.]

- [e.g. "All timestamps are normalized to UTC before feature generation"]
- [e.g. "Training and evaluation datasets are selected by immutable version ID"]

---

## 5. Environment variables

[Purpose and quirks of this service's env vars. Declarations live in
architecture.yml; this table adds the operational knowledge.]

| Variable | Purpose | Notes |
|----------|---------|-------|
| [VAR] | [what it does] | [gotchas — e.g. "must match auth-service's value"] |

---

## 6. Guardrail explanations

[architecture.yml declares the guardrail rules for this service. Record the
REASONING here — Claude enforces rules better when it knows why they exist, and
reviewers can judge exception requests against the original intent.]

| Guardrail (from architecture.yml) | Why it exists |
|-----------------------------------|---------------|
| [rule] | [consequence it prevents; incident/requirement behind it if any] |

---

## 7. Local development setup

```bash
# Prerequisites: [runtime version, docker services needed]

# First time
cp .env.example .env.local        # fill in: [minimum required vars]
[install command per this service's package/environment manager]
[migration / seed commands if any]

# Run
[dev command]                     # serves on [port]

# Tests
[test command]
```

Common local issues specific to this service:
- [symptom → cause → fix; only service-specific ones — generic framework failures
  are in the skill's Common failure signatures]

---

## 8. Deployment notes

[Only what ops must know that isn't obvious: migration-before-deploy, statefulness,
scaling constraints, config that must exist before first boot. "Nothing unusual"
is a valid entry.]

---

## 9. Known quirks and debt

[Optional. Landmines a developer or Claude should not "fix" in passing, half-done
refactors, intentionally weird code with a reason. Prevents well-meaning cleanup
from breaking things.]
