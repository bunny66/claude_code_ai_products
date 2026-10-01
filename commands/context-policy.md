# Context resolution policy

Shared policy referenced by all workflow commands. It defines which source of code
context to trust for which kind of question. Commands cite this file rather than
re-deriving the rules.

## The four sources

| Source | Carries | Freshness | Use for |
|--------|---------|-----------|---------|
| `architecture.yml` | Declared relationships, events, guardrail *rules* | Low (hand-maintained) | Intent about structure |
| service `CLAUDE.md` | Conventions, deviations, guardrail *reasoning* | Low (hand-maintained) | Intent about behaviour |
| CodeGraph | Actual symbols, calls, imports, impact radius | High (auto-synced) | Structural ground truth |
| grep / read | Raw text matches, file contents | High | Fallback structural lookup |

## Two axes that decide the order

- **Intent** (why / should / must-not) lives ONLY in the artifacts. No amount of code
  analysis reveals "other workspaces must never write this workspace's DB." CodeGraph and
  grep cannot answer intent questions.
- **Structural truth** (what / where / who-calls) is answered best by CodeGraph, then
  grep. The artifacts' structural claims may be stale and must not be trusted for this.

## Resolution order by question type

### A. "What is intended / required / forbidden?"
Boundary rules, guardrails, conventions, deviations, why something exists.
```
1. service CLAUDE.md  (behaviour intent, deviations)
2. architecture.yml   (relationship + guardrail rules)
— CodeGraph / grep cannot answer this; do not consult them for intent
```

### B. "What does the code actually do?"
Callers, callees, impact radius, dependency edges, where a symbol is defined,
who consumes a type, cross-file/dynamic-dispatch flows.
```
1. CodeGraph          (freshest + structurally accurate)
   └─ if its staleness banner flags a file mid-sync → read/grep THAT file directly, then resume
2. grep / read        (fallback when CodeGraph is absent, not initialised, or daemon down)
— architecture.yml is NOT authoritative for this; it is declared intent
```

### C. "Does intent still match reality?"  (validation / drift)
```
Cross-check CodeGraph (reality) against architecture.yml / CLAUDE.md (declared).
Discrepancy ⇒ the artifact is stale ⇒ trust CodeGraph, flag the artifact for update.
```

## CodeGraph availability — graceful degradation (mandatory)

CodeGraph is the workflow's only runtime dependency. Every use MUST degrade, never fail:

Detect availability once per command run:
```bash
codegraph status 2>/dev/null    # non-zero / not found ⇒ unavailable
```
Also treat as unavailable if the target service has no `.codegraph/` index
(`codegraph init` never run there).

- **Available** → use per question-type order above.
- **Unavailable** → fall back to grep/read for structural questions; print one line so
  the developer knows precision is reduced:
  `ℹ CodeGraph not available for [service] — using grep fallback (lower precision on cross-file/dynamic refs).`
- Never block, never abort solely because CodeGraph is missing. Intent questions (type A)
  are unaffected — they never used CodeGraph.

## Per-file staleness rule

When CodeGraph returns a result with a `⚠️` staleness banner naming a file (edited within
its debounce window), read that specific file directly for live content, then continue
using the graph for everything else. Do not discard the whole graph result over one
pending file.

## Precision note for structural queries

When available, prefer CodeGraph's structural commands over grep for:
- consumer/impact search (`codegraph impact`, `codegraph callers`) — catches re-exports,
  aliased imports, and dynamic-dispatch hops that `grep -r "Name"` silently misses
- affected-test detection (`codegraph affected`)
- breadth in pattern discovery (`codegraph explore`, `codegraph files`)

grep remains correct for literal-string needs (finding a config key, an env var name,
a log message) where structural resolution adds nothing.
