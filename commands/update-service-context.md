# Create or update a service CLAUDE.md

Scaffold or refresh a service's `CLAUDE.md` from the single stack-agnostic template
(`.claude/templates/CLAUDE.service.md`), driven by the service's resolved skills.

Division of responsibility (enforced by this command):
- **.claude/skills/** own generic technology rules — never copied into a service CLAUDE.md
- **architecture.yml** owns relationships, endpoints, events, env declarations, guardrail rules
- **The service CLAUDE.md** owns only service-specific facts: responsibility, structure
  snapshot, deviations from skill defaults, local conventions, env-var operational notes,
  guardrail reasoning, local dev setup, deployment notes, known quirks

## Arguments
`$ARGUMENTS` — one of:
- A service folder path: `policy-service`
- A service name matching a folder in this workspace
- `--all` — every service with no `CLAUDE.md`, a `TODO:` placeholder, or an old
  two-template header (`> Layer: backend` / `> Layer: frontend`) needing migration

If empty, stop:
```
✗ Provide a service folder path or name.
  /update-service-context policy-service
  /update-service-context --all
```

---

## Instructions

### Step 1 — Locate the service folder

Use the path directly, or search this workspace's folders for one with the
given name. If not found, stop and suggest `/update-architecture --validate` to list
registered services.

### Step 2 — Resolve skills

Resolve this service's skills via `.claude/skills/skill-registry.yml`:
1. `architecture.yml` `skills:` field wins if declared
2. Otherwise detect via the registry's signals (files, dependencies, extensions)
3. Load every skill's `Requires:` transitively; order per registry `load_order`

If nothing matches:
```
⚠ No skill matched for [service]. Observed: [key files/deps found]
  A. Name the skills to use (must exist in .claude/skills/)
  B. Add a skill file for this stack first (see .claude/skills/_template.md), then re-run
```
Wait for resolution. Read every resolved skill file — their Pattern discovery
checklists drive Step 5, and their rules define the *defaults* that §3 Deviations
is measured against.

### Step 3 — Handle an existing CLAUDE.md

If `CLAUDE.md` exists in the service folder:
- If its header is the old two-template format (`> Layer: backend|frontend`), say so —
  this is a migration. Offer:
  ```
  Found old-format CLAUDE.md (backend/frontend template).
    A. Migrate — extract service-specific content into the new template; DROP the
       generic pattern sections (controller/DTO/hook examples) now owned by .claude/skills/.
       Anything dropped that is NOT covered by a skill gets flagged for review.
    B. Show what would be dropped vs kept first
    C. Skip this service
  ```
- If it's already new-format: offer update (fill gaps/refresh snapshot, keep manual
  edits) / replace / diff.
Wait for the choice.

### Step 4 — Read architecture.yml entry

Extract this service's `description`, `guardrails`, `env_vars`, plus
`workspace.boundary_rules`. These pre-fill
the template — never re-ask what architecture.yml already answers. Guardrail *rules*
are referenced, not copied; only their reasoning column is authored here.

### Step 5 — Scan the codebase for service-specific facts

Run each resolved skill's Pattern discovery checklist, but with a different goal than
/implement uses it for: you are hunting **deviations and local facts**, not documenting
generic patterns.

For each discovered pattern, ask: does it match the skill's default rule?
- Matches → record nothing (the skill already says it)
- Differs → candidate row for §3 Deviations (confirm the "why" with the developer
  in Step 6 — never invent a justification)
- Local-only convention with no skill equivalent → candidate for §4

Also capture:
- Top-level structure snapshot (one nesting level) and any non-obvious locations
- Local dev facts: install/run/test commands from the manifest, dev port, docker deps
- Anything smelling like intentional weirdness (comments saying "do not change",
  odd patterns repeated deliberately) → candidate for §9 Known quirks

### Step 6 — Ask batched questions

One message, numbered, covering only what scanning couldn't settle:
- The "why" for each deviation candidate (a deviation without a reason is recorded as
  `⚠ UNEXPLAINED — confirm or fix`, which /implement treats as "ask before relying on it")
- Guardrail reasoning for each rule in architecture.yml
- Deployment notes, minimum env vars for local boot
Unanswered items become explicit `TODO:` entries.

### Step 7 — Write the file

Fill `.claude/templates/CLAUDE.service.md` and write to `[service-folder]/CLAUDE.md`:
- Header `> Skills:` = the resolved list from Step 2 (lets /implement skip re-detection
  and shows humans which skill defaults apply)
- §3 Deviations: table rows, or the explicit word "None" — never leave it blank
- Do not copy generic skill content anywhere; if you catch yourself writing framework
  rules, stop — that content belongs in .claude/skills/

If updating (not replacing): keep manually-edited section content, add missing
sections, preserve unknown sections verbatim at the end.

### Step 8 — Stamp the context baseline

```bash
cd [service-folder] && git rev-parse HEAD
```
Update only this service's entry in `architecture.yml`:
```yaml
[service-name]:
  context_built_commit: [HEAD-sha]
  context_built_date: [today]
```
(This is the baseline /sync compares against for staleness.) Skip with a note if the
folder isn't a git repo.

### Step 9 — Completion report

```
# Example output — your actual values will differ
✓ CLAUDE.md written: [service-folder]/CLAUDE.md
  Skills: [list]                       (header — read by /implement)

Auto-filled: structure snapshot, env table, local dev commands
Deviations recorded: [N] ([N] explained, [N] ⚠ UNEXPLAINED)
Pre-filled from architecture.yml: guardrails ([N] — [N] reasons still TODO)

Needs your input:
  ⚠ §3 — unexplained deviation: [summary]
  ⚠ §6 — guardrail reasoning: [rule]

Next: fill TODOs → /update-context → commit
```

---

## `--all` mode

1. List services needing work: no CLAUDE.md / TODO placeholders / old-format header
2. Confirm the list, then process sequentially, batching questions per service
3. Final summary: per-service status + remaining TODOs/UNEXPLAINED items across all
