# Execute implementation plan

Execute the approved implementation plan task by task, one service at a time,
with a human checkpoint between services and before any high-risk task.

This command is stack-agnostic: it resolves each service's skills via
`.claude/skills/skill-registry.yml` and applies those skills' rules, discovering existing
code patterns before implementing anything.

## Arguments
Optional: `$ARGUMENTS`
- No arguments: auto-locate the most recent approved plan in the current session
- A plan file path: `plans/<feature-slug>-plan.md`
- A task ID to resume from: `S1-2` (resumes mid-plan after interruption)

---

## Pre-flight checks

Before touching any code, complete every check below. Stop on any failure.

### Check 1 — Locate the plan file

**If running in the same session as /plan:**
Read the plan file path from session context (`SESSION.prd_summary_path` sibling).

**If argument is a file path:**
Read that file directly.

**If no session context and no argument:**
Search for plan files matching `plans/*-plan.md` across the workspace.
If exactly one found, use it and confirm:
```
# Example output — your actual values will differ
Found plan: plans/<feature-slug>-plan.md
Using this plan. Press enter to confirm or provide a different path.
```
If multiple found, list them and ask which to use:
```
# Example output — your actual values will differ
Multiple plan files found:
  1. plans/<feature-slug>-plan.md
  2. plans/<other-feature>-plan.md

Which plan should I execute? (enter number)
```
If none found, stop:
```
✗ No approved plan file found.
Run /plan first, then APPROVE it before running /implement.
```

### Check 2 — Verify plan is approved

Read the plan file. Check for the approval marker.
An approved plan contains the line: `> Status: APPROVED` near the top.

If not approved, stop:
```
✗ Plan is not approved.
Run /plan, review the output, and type APPROVE before running /implement.
```

### Check 3 — Read all context files

Read in this order — do not skip any:
1. `architecture.yml` — service relationships, guardrails
2. `architecture.yml` `workspace.boundary_rules` and `external:` contracts
3. Each affected service's `CLAUDE.md` — coding conventions, forbidden patterns
4. Root `CLAUDE.md` — system-wide conventions

If `architecture.yml` is missing, stop:
```
✗ architecture.yml not found. System context is required before implementation.
```

If a service `CLAUDE.md` is missing, warn but continue:
```
⚠ [service]/CLAUDE.md not found — will rely on code pattern discovery only.
Consider running /update-service-context after implementation.
```

### Check 4 — Parse the plan

Read the plan file and extract:
- Feature name and slug
- Branch name (from Identifiers section)
- All tasks in order: prerequisites → per-service → cross-service → deployment
- For each task: ID, title, type, service, skills, files, depends-on, risk, guardrail
- Already-completed tasks: marked `[x]` in the plan file
- Pending tasks: marked `[ ]`

**Resolve skills per service** using `.claude/skills/skill-registry.yml`:
1. If `architecture.yml` declares `skills:` for the service, use exactly those
2. Otherwise detect: match the registry's signals (config files, dependencies,
   file extensions) against the service folder
3. Load everything in each skill's `Requires:` transitively
4. Order per the registry's `load_order` — on conflicting rules, the later
   (more specific) skill wins
5. Read every resolved skill file from `.claude/skills/` — these provide the pattern
   discovery checklists, implementation rules, test rules, and verification
   commands used throughout this command

If a service matches no skill in the registry, stop and ask:
```
⚠ No skill matched for [service]. Detected files suggest: [observations]
  A. Name the skills to use (must exist in .claude/skills/)
  B. Add a new skill file for this stack first, then re-run
```

Print the execution summary:
```
# Example output — your actual values will differ
Plan loaded: <feature-slug>
Branch:      feature/<ticket-id>-<feature-slug>

Services and resolved skills:
  <project-a>   [python, pytorch, mlflow] — [N] tasks
  <project-b>   [python, fastapi]         — [N] tasks

Tasks:
  ✓ Completed:  [N] (already marked done)
  ○ Pending:    [N]
  ◉ Starting:   [first pending task ID and title]

High-risk tasks requiring checkpoint:
  [task ID] [title] — [risk reason]
```

### Check 5 — Verify or create feature branches (per repo)

Each service is its own git repo. Create the feature branch in every affected
service repo, not just one.

For each affected service folder:

**Step 5a — Check for dirty working tree first**
```bash
cd [service-folder]
git status --porcelain
```

If output is non-empty (unstaged or uncommitted changes exist), stop immediately:
```
✗ [service-folder] has uncommitted changes:
  [git status output]

Branching over uncommitted changes can carry unrelated work onto the feature branch.
Options:
  A. Stash them: git stash — I will continue and you can restore with git stash pop later
  B. Commit them manually, then re-run /implement
  C. Discard them: git checkout . — WARNING: changes will be lost
```
Wait for response. Do not create a branch until the working tree is clean.

**Step 5b — Create or check out the branch**
```bash
git branch --list [branch-name]
```

If branch does not exist:
```bash
git checkout main          # or master — check git branch -r to confirm default
git pull origin main
git checkout -b [branch-name]
```

If branch already exists (resuming):
```bash
git checkout [branch-name]
```

Print a status line per repo:
```
# Example output — your actual values will differ
Branches:
  ✓ <service-a>   feature/<ticket-id>-<feature-slug>  (new)
  ✓ <service-b>   feature/<ticket-id>-<feature-slug>  (new)
  ✓ <service-c>   feature/<ticket-id>-<feature-slug>  (existing — resuming)
```

---

## Pattern discovery (skill-driven)

**Run pattern discovery once per service before implementing its first task.**
This is mandatory — do not skip even if CLAUDE.md exists. CLAUDE.md describes
intent; the codebase shows reality. Discrepancies between them are common.

For each service, work through the **Pattern discovery checklist of every resolved
skill**, in load order (language → runtime → markup → styling → ui-library → framework).
Each skill's checklist says what to scan and record for its layer — e.g. the nodejs
skill detects the package manager, the tailwind skill extracts the design vocabulary,
the framework skill maps module/routing structure.

**Use CodeGraph for breadth where available** (per `.claude/context-policy.md`, type B):
instead of sampling 2–3 files and generalising, `codegraph explore "<area>"` and
`codegraph files [path]` survey the whole service's real structure, reducing the
"sampled an unrepresentative example" drift. Fall back to reading sample files when
CodeGraph is unavailable. Skill checklists that describe *conventions* (naming, style)
still come from reading actual files — CodeGraph gives breadth, not taste.

Consolidate the findings into one pattern summary per service and print it before the
first task:

```
# Example output — structure comes from the resolved skills; values will differ
Patterns — <project-name>  [skills: python, pytorch, mlflow]
=================================================================
[python]     Environment: uv · configured lint and test commands
[pytorch]    Model code: src/models/ · configs: configs/ · tests: tests/models/
[mlflow]     Experiment tracking and artifact storage are configured
```

If a skill's checklist finds nothing (e.g. no existing analogue component for the
tailwind vocabulary), record the gap explicitly — it becomes a question, not a guess.

If mid-task an unfamiliar pattern appears that discovery didn't cover, scan 1–2
existing examples of that pattern before writing — never guess.


---

## Execution loop

Work through pending tasks in plan order. For each task:

### Step A — Announce the task

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Task [ID]: [title]
  Component: [component-name] ([role from architecture.yml])
Type:      [type]
Risk:      [low / medium / high]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step B — Pre-task checkpoint for HIGH risk tasks

If the task is marked `Risk: high`, stop and ask:
```
⚠ HIGH RISK TASK — manual review required before I proceed

Task:     [ID] [title]
Risk:     [risk reason from plan]
Guardrail:[relevant rule if any]

What I'm about to do:
  [precise description — files, what changes, why]

Proceed? (yes / skip / abort)
  yes   — proceed with this task
  skip  — mark as TODO and continue to next task
  abort — stop the entire implementation session
```

Wait for explicit response. Never auto-proceed on a high-risk task.

### Step C — Check dependencies

Verify all tasks in `Depends on:` are marked `[x]` in the plan file.
If any dependency is incomplete, stop:
```
✗ Task [ID] depends on [dep-ID] which is not yet complete.
Complete [dep-ID] first or re-order the plan manually.
```

### Step D — Navigate to the correct service folder

```bash
cd [service-folder-path]
```

Confirm you're in the right directory before writing any file.

### Step E — Implement the task following discovered patterns

**Before writing any code:** re-read the pattern summary for this service.
If the task involves a pattern not covered in the summary (e.g. first time seeing
a particular component type), scan 1-2 existing examples of that pattern first.

Apply the discovered patterns exactly — match naming, file location, structure,
and conventions. If you encounter an unfamiliar pattern mid-task, scan for an
example before guessing.

**Read the service CLAUDE.md guardrails before writing the first line of code.**
If implementing would violate a guardrail, stop and ask:
```
⚠ Guardrail conflict
  Task [ID] requires: [what the task asks for]
  Guardrail says:     [the rule]

  Option A: [compliant alternative approach]
  Option B: Override the guardrail — requires explicit confirmation
```

---

#### Task type rules come from the resolved skills

Each resolved skill contributes **Implementation rules** per task type. Apply the
rules of every loaded skill that covers the current task's type; where two skills
both address the same concern, the more specific layer wins (framework over
runtime over language).

Universal rules regardless of skill:
- If the task's type is not contributed by any resolved skill, stop — the plan and
  the service's skills disagree; resolve with the developer before writing code
- Hard constraints in a skill (marked "never") are non-negotiable without explicit
  developer confirmation
- Required follow-ups in a skill (marked "after writing") run before the task is
  considered complete — for example dependency lockfile updates, schema or API
  regeneration, data/model artifact validation, or deployment checks when required
- UI tasks additionally honour the design vocabulary and mockup specs carried in
  the plan (tailwind/reactjs skills define how)

### Step F — Write tests alongside the code

Write tests immediately after implementation — not at the end of all tasks.

Apply the **Test rules** section of every resolved skill covering the task's type.
The framework skill defines coverage minimums per task type; the language/runtime
skills define which runner and mocking approach to use.

Universal rules regardless of skill:
- Match the exact test structure and mock pattern found during pattern discovery —
  never introduce a second test framework or mocking approach to a service
- Every task ships with its tests and applicable data/model validation in the same
  commit; a task without the minimum verification its skill demands is not complete

### Step G — Lint, build, and commit

Run these checks in order. Do not commit if any step fails.

**Frontend design self-check (UI tasks only — before lint):**

For `new component`, `new page`, or `modify component` tasks, before running the
technical checks, verify the built UI against its constraints from the page map:

```
Design self-check — [component/page name]
  Design vocabulary:
    □ Card/container classes match closest analogue? [the tokens from the page map]
    □ Spacing scale matches (space-y, padding, gaps)?
    □ Button size/radius/variant matches app convention?
    □ Typography (size, weight, color) matches?
  Mockup match (if a mockup spec exists for this component):
    □ Layout structure matches the mockup?
    □ Elements present in the order the mockup shows?
    □ All states from the mockup handled (empty/loading/error/populated)?
    □ Copy/labels match the mockup exactly?
```

If any check fails, fix it before proceeding to lint. If a mockup spec and the app's
design vocabulary genuinely conflict and no resolution was recorded in the page map,
stop and ask the developer rather than guessing.

This is a structural self-check, not a pixel comparison — it confirms the component
uses the right tokens and follows the mockup's layout, which is what prevents the
"doesn't match the rest of the app / doesn't match the screenshot" failures.

Run the **Verification commands** of every resolved skill, in load order, deduplicated.
The project's configured commands are authoritative. For AI/ML components, include
relevant data-quality checks, evaluation against the plan's agreed datasets/metrics,
and artifact or lineage validation when applicable. Do not invent metric thresholds.

Never assume script names — each skill's rules say to read the project's scripts and
use the detected package/environment manager.

If lint has auto-fixable errors, fix them silently and include in the commit.
If lint has manual errors, stop and report:
```
⚠ Lint errors require manual fixes in task [ID]:
  [file]:[line] — [error message]
  [file]:[line] — [error message]
How should I resolve these? (fix suggestion / skip lint for now)
```

If a configured type-check fails, stop — do not commit with type errors:
```
✗ TypeScript errors in task [ID]:
  [file]:[line] — [error]
Fixing before commit.
```

If a configured build fails, stop and investigate:
```
✗ Build failed after task [ID] (framework skill's build step):
  [error output]
This may indicate a runtime error not caught by TypeScript.
Investigating root cause before proceeding.
```

After all checks pass:
```bash
git add [files for this task + their test files only]
git commit -m "[task-id]: [task title]

[2-3 lines — what changed and why, not what the code does]
Plan task: [task-id]
Feature: [feature-slug]"
```

Commit rules:
- Max 72 chars on first line
- Include `Plan task:` and `Feature:` trailers for traceability
- Only commit files listed in the task `Files:` field + their test files
- Never commit unrelated changes

If tests or build fail and the fix requires scope beyond this task, stop:
```
⚠ Fix required beyond task [ID] scope.

Issue:        [description]
Fix required: [file/service]

Options:
  A. I fix it as a prerequisite within this task (adds scope)
  B. We create a new task P0 and I do it first
  C. You fix it manually and I continue
```

### Step H — Mark task complete in plan file

After a successful commit, change `[ ]` to `[x]` for this task in the plan file
of the plan file at `plans/<feature-slug>-plan.md`.

```
[x] S1-1. Create renewal_schedule config table migration
```

### Step I — Service boundary checkpoint

After completing all tasks for a service, stop before moving to the next:

```
════════════════════════════════════════
SERVICE COMPLETE: [service-name] ([backend/frontend])
════════════════════════════════════════

Tasks completed: [N]
  [x] [task-id] [title]
  [x] [task-id] [title]

Commits: [N]
  [short hash] [task-id]: [title]

Checks passed:
  ✓ Lint
  ✓ Type check       (frontend only)
  ✓ Build            (frontend only)
  ✓ Tests: [N] passing

Next service: [service-name] ([backend/frontend]) — [N] tasks pending

Review commits above before I continue.
Proceed to [next-service-name]? (yes / pause / abort)
  yes   — continue to next service
  pause — stop here, resume later with /implement
  abort — stop, do not continue
```

Wait for explicit confirmation.

---

## Handling interruptions and resuming

The plan file checkbox state (`[x]` vs `[ ]`) is the resume mechanism.
Every completed task is committed and checked before moving on.

To resume:
```
/implement [plan-file-path]
/implement [task-id]          ← resume from a specific task
```

On resume, re-read the plan file, run pattern discovery for any service not yet started,
then continue from the first `[ ]` task. No questions re-asked if context files are present.

Print on resume:
```
# Example output — your actual values will differ
Resuming: <feature-slug>
Branch:   feature/<ticket-id>-<feature-slug>

Progress:
  [x] P1    [title]    ✓
  [x] S1-1  [title]    ✓
  [x] S1-2  [title]    ✓
  [ ] S1-3  [title]    ← resuming here
  [ ] FE1-1 [title]
  [ ] FE1-2 [title]
  [ ] D1    Deployment order

Continuing from S1-3...
```

---

## Completion

When all non-deployment tasks are `[x]`:

```
# Example output — your actual values will differ
════════════════════════════════════════════════════
IMPLEMENTATION COMPLETE
════════════════════════════════════════════════════

Feature:  <feature-slug>
Branch:   feature/<ticket-id>-<feature-slug>
Commits:  [N] across [N] services

Completed:
  Backend
    [x] P1   [title]
    [x] S1-1 [title]
    [x] S1-2 [title]
  Frontend
    [x] FE1-1 [title]
    [x] FE1-2 [title]
  Cross-service
    [x] C1   [title]

Deployment tasks (manual — ops team):
  [ ] D1  Deploy order: [service-a] before [service-b]
  [ ] D2  Run migrations before deploying [service-name]
  [ ] D3  Add [VAR_NAME] to prod config

════════════════════════════════════════════════════

Run /test to execute the testing plan.
After tests pass: run /raise-pr to create the Bitbucket PR.
```

Deployment tasks stay `[ ]` — they are ops reminders, not Claude Code actions.
Do not run `/test` automatically. Stop here.
