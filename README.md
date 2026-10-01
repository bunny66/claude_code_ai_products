# Claude Code Workflow for AI/ML Projects

> A reusable workflow for planning and implementing changes across data pipelines,
> model training and evaluation, inference services, and supporting applications.
> It discovers each project's actual stack and follows its existing tools and conventions.

---

## Overview

The workflow has two phases: **Setup** (done once) and **Feature development** (repeated per feature).

```
SETUP (one time)
  Step 1 → Create workspace folder structure
  Step 2 → /update-architecture      build architecture.yml
  Step 3 → /update-context           generate root CLAUDE.md
  Step 4 → /update-service-context   create per-service CLAUDE.md files
   Step 5 → Configure pr-config.yml   configure available PR and notification integrations

FEATURE DEVELOPMENT (per feature)
  Step 6  → /sync                     update local repos, detect stale context
  Step 7  → /prd                      read PRD (Doc/Jira/file/image), identify services
  Step 8  → /plan                     generate implementation plan, review + APPROVE
  Step 9  → /implement                execute plan task by task, service by service
  Step 9b → /code-feedback (optional) apply review feedback: reclassify, re-plan, implement
  Step 10 → /test                     run all test levels, generate test report
   Step 11 → /raise-pr                 push branches, raise PRs, send configured notifications
```

**Human checkpoints:** Branch-state decisions during sync (Step 6), component mapping
confirmation and clarifying questions (Step 7), plan approval (Step 8), per-service
   review during implementation (Step 9), PR review in the configured Git provider (after Step 11).
Claude stops and waits at every gate.

---

## Phase 1 — Setup

### Step 1 — Create workspace folder structure

Put the related project repositories under a single workspace root so Claude Code can
trace changes across data, model, and application components. The workflow also works
with a single repository organized into multiple components.

```bash
mkdir ~/workspace && cd ~/workspace

# Example AI/ML components; use the repository layout that fits your project
git clone <git-url>/feature-pipeline.git       domain-forecasting/feature-pipeline
git clone <git-url>/model-training.git         domain-forecasting/model-training
git clone <git-url>/inference-service.git       platform/inference-service
git clone <git-url>/evaluation-suite.git        platform/evaluation-suite
# ... repeat for every service
```

The `.claude/` folder lives at the workspace root alongside the domain folders:

```
~/workspace/
├── .claude/                          ← everything the workflow needs lives here
│   ├── pr-config.yml
│   ├── commands/
│   │   ├── sync.md
│   │   ├── prd.md
│   │   ├── plan.md
│   │   ├── implement.md
│   │   ├── code-feedback.md
│   │   ├── test.md
│   │   ├── raise-pr.md
│   │   ├── update-architecture.md
│   │   ├── update-context.md
│   │   └── update-service-context.md
│   ├── skills/
│   │   ├── skill-registry.yml
│   │   ├── _template.md
│   │   ├── python.md       pytorch.md       sklearn.md
│   │   └── [other skills discovered or added for this workspace]
│   └── templates/
│       ├── CLAUDE.service.md
│       └── domain-pr-config.yml
├── architecture.yml              ← generated in Step 2 (workspace root)
├── CLAUDE.md                     ← generated in Step 3 (workspace root)
├── domain-forecasting/
│   ├── README.md                 ← document ownership and data boundaries
│   ├── pr-config.yml             ← optional workspace-group override
│   ├── feature-pipeline/
│   └── model-training/
└── platform/
   ├── inference-service/
   └── evaluation-suite/
```

**Critical rule:** Always launch Claude Code from `~/workspace/` — never from inside
a domain or service folder. The commands and context files must be at the working directory.

```bash
cd ~/workspace
claude
```

---

### Step 2 — Build architecture.yml

`architecture.yml` is the single source of truth for project components and their
relationships. This reusable starter intentionally has `services: {}`; running the
command in your project workspace discovers and adds its components.

```
/update-architecture
```

Claude will:
1. Discover project folders automatically from manifests and ML/data project markers
2. Ask you to confirm the list (add or remove components)
3. Ask focused questions per component — interfaces, events, data/model inputs and
   outputs, shared artifacts, and other components it calls
4. Type `check code` for any question to have Claude scan the source files instead
5. Ask once about shared infrastructure (databases, queues, data platforms, feature
   stores, model registries, and artifact stores)
6. Write `architecture.yml` to the workspace root
7. Create stub `README.md` files for workspace groups when needed

**After it completes:**
- Fill in the `TODO:` sections in `architecture.yml` (typically the service
  interaction map which can't be fully inferred from code)
- Fill in domain boundary rules in each `domain-*/README.md`
- Run validation to catch any broken references:

```
/update-architecture --validate
```

Fix any warnings before proceeding. This is worth the time — every command after
this reads `architecture.yml` as ground truth.

---

### Step 3 — Generate root CLAUDE.md

```
/update-context
```

Claude reads `architecture.yml` + all domain `README.md` files + all service
`CLAUDE.md` files (if they exist yet) and writes a synthesised `CLAUDE.md` at the
workspace root.

**After it completes:**
- Fill in the `TODO:` sections Claude flagged — system overview and local dev steps
  are the most common ones it can't auto-fill
- Re-run `/update-context` any time `architecture.yml` or a service `CLAUDE.md` changes

---

### Step 4 — Create per-service CLAUDE.md files

One `CLAUDE.md` per service. This is what `/implement` reads to understand coding
conventions, module structure, testing patterns, and guardrail reasoning for each
service. Without these files, Claude falls back to code scanning alone.

Run for each project component:
```
/update-service-context domain-forecasting/feature-pipeline
/update-service-context domain-forecasting/model-training
/update-service-context platform/inference-service
# ... repeat for every component
```

Or run for all at once:
```
/update-service-context --all
```

Claude will:
1. Detect the component's workload and stack from its code and project files
2. Scan for actual patterns — data contracts, pipeline stages, model interfaces,
   configuration, evaluation setup, tests, and deployment conventions as applicable
3. Pre-fill guardrail explanations from `architecture.yml`
4. Ask batched questions for anything it couldn't infer
5. Write the `CLAUDE.md` from the single stack-agnostic template
   (`.claude/templates/CLAUDE.service.md`), with resolved skills in the header

**After it completes:**
- Fill in `TODO:` sections — local dev setup and guardrail reasoning are the
  most important ones to complete manually
- Re-run this command whenever a service's patterns change significantly

---

### Step 5 — Configure PR settings

```
/raise-pr
```

On first run with an empty `pr-config.yml`, Claude will ask for the supported
source-control and notification settings. The current `/raise-pr` integration uses:
1. Slack channel ID for PR notifications (workspace default)
2. Bitbucket workspace slug
3. Default target branch

These are written to `.claude/pr-config.yml`.

**For domain-specific overrides** (different Slack channel or target branch per domain):
```bash
cp .claude/templates/domain-pr-config.yml domain-forecasting/pr-config.yml
# Edit domain-forecasting/pr-config.yml — fill in only what differs from workspace default
```

Config resolves per domain: domain config overrides workspace config. Fields left
empty in the domain config inherit from the workspace config.

---

## Phase 2 — Feature Development

Every feature follows these six commands in order. Each hands off to the next via
files written to disk — session restarts between steps lose nothing.

---

### Step 6 — /sync — Update local repos and detect stale context

Run this first, before reading any PRD. It prevents the two most common sources of
bad output: planning against stale code, and planning against stale context files.

```
/sync
```

Scoped variants:
```
/sync domain-forecasting              sync only one workspace group
/sync domain-forecasting/model-training sync only one component
/sync --fetch-only                     report status without pulling or switching
/sync --context-check                  only check context staleness, skip git sync
```

**What Claude does:**

1. Resolves the base branch per domain from `pr-config.yml` (handles teams that
   branch off `develop` instead of `main`)
2. Fetches every repo (read-only, always safe) and classifies each by state:
   clean on base, dirty, on another branch, or detached HEAD
3. Resolves each repo by state — fast-forwards clean ones, stops on dirty trees
   (offers stash), and asks before switching branches
4. Checks whether `architecture.yml` and service `CLAUDE.md` files have drifted —
   compares the recorded context-built SHA against current HEAD, scans changed files
   for endpoint/event/structure changes, and recommends `/update-architecture` or
   `/update-service-context` if context looks stale

**Your job:**
- For any repo on a non-base branch, decide: switch to base (forgotten branch) or
  stay (active work). Claude shows ahead/behind counts and last-commit age to help.
- If context staleness is flagged, run the recommended update commands before `/prd`

**Why this matters:** a feature branched from week-old `main` references files and
signatures that have already changed. Stale `architecture.yml` makes `/prd` and
`/plan` reason from wrong facts. This step closes both gaps before they propagate.

---

### Step 7 — /prd — Read and analyse the PRD

Accepts multiple input types, alone or combined:

```
/prd https://docs.google.com/document/d/<id>/edit      Google Doc
/prd https://yourorg.atlassian.net/browse/PROJ-123     Jira URL
/prd PROJ-123                                           Jira issue ID
/prd ./docs/feature-name.md                             local file (.md .txt .pdf .docx)
/prd ./mockups/renewal-page.png                         mockup image (.png .jpg .webp)
/prd PROJ-123 ./mockups/screen1.png ./docs/notes.md     combined sources
```

**What Claude does:**

1. Loads system context first — reads `architecture.yml`, all domain `README.md`
   files, and root `CLAUDE.md` before touching the PRD
2. Fetches every source: Google Doc (Drive MCP), Jira (Atlassian MCP), local files,
   or inline text — and merges them, flagging any conflicts between sources
3. Extracts and analyses embedded mockup images as images, producing a visual spec
   (layout, components, states, copy) for each — the design reference for the UI.
   If the MCP strips images, it tells you and accepts mockups as local file paths
4. Extracts all requirements, user stories, and acceptance criteria — each labelled
   with its source for traceability
5. Maps requirements to the relevant code, data, pipeline, model, evaluation, API,
   or UI surfaces; uses mockup analysis when visual references are provided
6. Identifies affected components from `architecture.yml` and checks workspace
   boundary and data-handling requirements
7. Captures ticket ID (auto-set if a Jira ID was the input; otherwise optional)
8. Asks all clarifying questions in one batch — answer by number
9. Writes the PRD summary — including affected component maps and any applicable
   data, model, evaluation, or UI requirements — to `<primary-domain>/plans/prd-summary-<slug>.md`

**Your job:**
- Answer clarifying questions — "your call" answers become documented delegated decisions
- Confirm or correct the ticket ID and feature slug
- Confirm the domain and service mapping before Claude proceeds
- Resolve any flagged conflicts between requirements and existing project conventions

**Output files:**
```
<primary-domain>/plans/prd-summary-<feature-slug>.md
```

---

### Step 8 — /plan — Generate implementation plan

```
/plan
```

Run in the same session as `/prd` or in a new session (Claude reads the PRD summary
file automatically).

**What Claude does (8 phases):**

- Phase 0: reads `architecture.yml`, workspace `README.md` files, and relevant
   component maps from the PRD summary — confirms scope before proceeding
- Phase 1: restates the feature in plain English — confirms your understanding
- Phase 2: asks all remaining ambiguous questions in one batch
- Phase 3: produces Areas of Impact — data and schema changes, pipeline stages,
   model/config/artifact changes, APIs, events, infrastructure, and existing behavior at risk
- Phase 4: generates ordered atomic tasks, respecting data, training, evaluation,
   deployment, and cross-component dependencies
- Phase 5: produces a project-native testing and evaluation plan, including unit,
   integration, data-quality, model-quality, contract, regression, and end-to-end checks as applicable
- Phase 6: produces risk register with rollback plans
- Phase 7: lists all delegated decisions for reviewer visibility
- Phase 8: summary + waits for `APPROVE`

**Your job:**
- Review the Areas of Impact carefully — this is where mistakes are cheapest to catch
- Review high-risk tasks — these need the most scrutiny before you approve
- Check the deploy order — correct it if wrong
- Type `APPROVE` when satisfied

**What APPROVE does:**
Writes the plan file with a `> Status: APPROVED` header, branch name, and service
labels. Both domain copies get the plan. This is what `/implement` reads.

**Output files:**
```
<primary-domain>/plans/<feature-slug>-plan.md      ← primary
<secondary-domain>/plans/<feature-slug>-plan.md    ← secondary copy
```

---

### Step 9 — /implement — Execute the plan

```
/implement
```

Or resume from a specific task after interruption:
```
/implement S1-3
```

**What Claude does before writing any code:**

1. Verifies `> Status: APPROVED` in the plan file
2. Reads all context: `architecture.yml`, domain `README.md`, all service `CLAUDE.md`
3. Checks for dirty working tree in every affected repository — stops if uncommitted changes
   exist, offers to stash
4. Creates the feature branch in every affected repository
5. Discovers each component's actual layout, configuration, data/model contracts,
   pipeline and serving patterns, tests, and deployment conventions

**What Claude does per task:**

- Announces the task with type, service, and risk level
- Stops and asks before every `Risk: high` task
- Checks all dependencies are marked `[x]` before starting
- Reads service CLAUDE.md guardrails before the first line of code
- Implements following the exact patterns discovered, not assumed
- Installs or updates dependencies only when the project's established workflow requires it
- Runs applicable data/schema migrations and follows the project's rollback procedure
- Regenerates API schemas or model/interface artifacts when required by project conventions
- Writes tests immediately after each task — not at the end
- Runs the repository's applicable formatting, static checks, build, tests, and
   evaluation commands before each commit
- Commits per task with tracer footers (`Plan task:`, `Feature:`)
- Updates `[ ]` → `[x]` in both plan file copies

**Human checkpoints:**

After every service completes, Claude stops and prints a summary of commits,
passing tests, and what's coming next. You choose: `yes` to continue, `pause`
to stop and resume later with `/implement`, or `abort`.

**Resume after interruption:**
The plan file checkbox state is the resume mechanism. Every completed task is
committed and checked before moving on. Run `/implement` or `/implement S1-3`
from any session.

---

### Step 10 — /test — Execute the testing plan

```
/test
```

Flags:
- `--unit-only` — run unit tests only, skip integration, contract, and E2E
- `--skip-e2e` — run everything except E2E (useful when E2E environment is unavailable)

**What Claude does:**

Verifies implementation is complete (all non-deployment tasks are `[x]`), then
runs tests in strict order:

| Level | What runs | Examples (use project-native tools) |
|-------|-----------|-------------------------------------|
| 1 — Unit | Changed functions and modules | pytest, unittest, language-native test runner |
| 2 — Integration | Pipeline, storage, API, and service boundaries | Integration tests and contract checks |
| 3 — Data | Input/schema validation and data-quality invariants | Great Expectations, Pandera, custom checks |
| 4 — Model | Evaluation against agreed datasets, metrics, and baselines | Project evaluation suite; record limits and variance |
| 5 — Regression | Existing behaviors and quality baselines at risk | Regression suite and tracked metric comparisons |
| 6 — End-to-end | Relevant pipeline-to-artifact or request-to-inference flow | Project integration or deployment checks |
| 7 — Reproducibility | Config, data, code, and artifact lineage are captured | Project tracking/registry tools and documented versions |

**On any failure:** Claude stops at that level, reports the failure, and offers
three options: fix it, skip with a note in the report, or abort.

**Missing regression tests:** If a regression scenario from the plan has no existing
test coverage, Claude writes it now and commits it to the branch.

**Output files:**
```
<primary-domain>/plans/<feature-slug>-test-report.md
```

Test report status:
- `PASSED` — all levels green, ready to merge
- `PASSED WITH NOTES` — passed but some tests skipped or coverage accepted below threshold
- `FAILED` — do not merge until resolved

---

### Step 11 — /raise-pr — Raise a code review and send configured notifications

```
/raise-pr
```

Flags:
- `--force` — raise PR despite FAILED test status (reviewer is warned prominently)
- `--skip-slack` — raise PR without Slack notification (when configured)

**What Claude does:**

1. Resolves config per domain: reads `<domain>/pr-config.yml` first, falls back
   to `.claude/pr-config.yml` for any missing field
2. Displays resolved config (channel, target branch, workspace) — you confirm before proceeding
3. Verifies test report exists and is not FAILED (blocks unless `--force`)
4. Pushes all unpushed commits across every service repo
5. Uses `--force-with-lease` for any branch that was rebased — never force-pushes
   without explicit confirmation
6. Composes one PR description from plan + test report artifacts — no re-asking questions
7. Raises one PR per service repo via `bb` CLI
8. Groups PRs by resolved Slack channel — sends one message per unique channel
9. Updates plan file with `> PR Status: RAISED` and all PR URLs

**PR description includes:**
- What the feature does (from PRD summary)
- Services changed table
- Implementation approach (from plan tasks)
- Breaking changes, new env vars, deploy order (from impact analysis)
- Test results table (from test report)
- Dynamic checklist based on what's relevant to this feature
- Reviewer notes — delegated decisions from the plan

**Your job after `/raise-pr`:**
1. Open the PR links and add reviewers in Bitbucket
2. Add new env vars to staging config if flagged
3. Coordinate deploy order with ops if services have dependencies

---

## Maintenance commands

These run outside the feature development cycle.

### When a service changes its architecture

Update `architecture.yml` first, then regenerate the root `CLAUDE.md`:
```
/update-architecture --validate     ← check for broken references first
# edit architecture.yml manually if needed
/update-context                     ← regenerate root CLAUDE.md
```

### When a new service is added

```
/update-architecture                ← interactive, detects the new folder
/update-service-context <new-service-path>
/update-context
```

### When a service's internal patterns change

```
/update-service-context <service-path>
/update-context
```

### When you're not sure if local repos or context are current

```
/sync                               ← fetches all repos, reports git + context staleness
/sync --context-check               ← skip git, only check if context files have drifted
```

`/sync` compares each service's current HEAD against the SHA recorded when context was
last built, and tells you which context files likely need refreshing.

### To check architecture.yml consistency at any time

```
/update-architecture --validate
```

### To see the architecture.yml format reference

```
/update-architecture --help
```

---

## Quick reference

| Command | When to run | Output |
|---------|-------------|--------|
| `/update-architecture` | Setup, or when services change | `architecture.yml` |
| `/update-context` | After architecture or service context changes | `CLAUDE.md` (root) |
| `/update-service-context` | Setup, or when service patterns change | `<service>/CLAUDE.md` |
| `/sync` | Start of each feature, before `/prd` | Updated repos + staleness report |
| `/sync --fetch-only` | Check repo status without pulling | Status report only |
| `/sync --context-check` | Check only context drift | Staleness report only |
| `/prd <doc/jira/file/image>` | Start of each feature, after `/sync` | `plans/prd-summary-*.md` |
| `/prd PROJ-123 ./mockup.png` | PRD with Jira ticket + mockup image | Same, with visual specs |
| `/plan` | After `/prd` answers are confirmed | `plans/*-plan.md` |
| `/implement` | After plan is APPROVED | Code + commits on feature branch |
| `/implement <task-id>` | Resume after interruption | Continues from that task |
| `/code-feedback "<text>"` | After `/implement`, on review feedback | PRD/plan updated, changes implemented |
| `/code-feedback ./feedback.md` | Feedback from a file | Same |
| `/test` | After `/implement` completes | `plans/*-test-report.md` |
| `/test --skip-e2e` | When E2E environment unavailable | Same, minus E2E |
| `/raise-pr` | After `/test` passes | Bitbucket PRs + Slack notification |
| `/raise-pr --force` | Test failures need PR review | PR with failure warning |

---

## Key files reference

| File | Owned by | Never manually edit |
|------|----------|---------------------|
| `architecture.yml` | You (via `/update-architecture`) | No — edit directly when services change |
| `CLAUDE.md` (root) | `/update-context` | Yes — always regenerate |
| `<domain>/README.md` | You (manually) | No — fill in boundary rules |
| `<service>/CLAUDE.md` | `/update-service-context` + you | No — update when patterns change |
| `plans/prd-summary-*.md` | `/prd` | No — re-run `/prd` if PRD changes |
| `plans/*-plan.md` | `/plan` | No — re-run `/plan` with feedback |
| `plans/*-test-report.md` | `/test` | Yes — regenerate with `/test` |
| `.claude/pr-config.yml` | You (first run of `/raise-pr`) | No — edit directly |
| `<domain>/pr-config.yml` | You (manually from template) | No — edit directly |

---

## Onboarding a new developer

```bash
# Clone the workspace config repo (contains .claude/ with commands, skills, templates + architecture.yml)
git clone git@bitbucket.org:yourorg/workspace-config.git ~/workspace

cd ~/workspace

# Clone each project component into the appropriate workspace folder
git clone <git-url>/feature-pipeline.git domain-forecasting/feature-pipeline
git clone <git-url>/model-training.git domain-forecasting/model-training
git clone <git-url>/inference-service.git platform/inference-service
# ... repeat per component

# Launch Claude Code from workspace root
claude

# Verify setup
/update-architecture --validate
```

That's it. The new developer has full system context immediately.
