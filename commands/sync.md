# Sync local repos and detect context staleness

Bring local service repos up to date with their remote base branch before planning or
implementing, and detect whether `architecture.yml` and service `CLAUDE.md` files have
drifted from the current code.

Run this at the start of a feature, before `/prd`, or any time you suspect your local
clones or context files are stale.

## Arguments
Optional: `$ARGUMENTS`
- No arguments: sync all service repos in the workspace
- A service path: sync only that service
- A service path: `workspace-policy/policy-service` — sync only that service
- `--fetch-only`: fetch and report status for all repos, but do not pull or switch branches
- `--context-check`: skip the git sync, only run the context staleness check (Phase 3)

---

## Phase 0 — Resolve the base branch per workspace

The base branch (what to sync against and branch from) is not always `main`. Read it
from the same config `/raise-pr` uses, so "what I branch from" matches "what I merge into".

Resolve `default_target_branch` for this workspace:
1. `.claude/pr-config.yml` — this workspace's config
2. If missing or empty, default to `main` and note it

Print the resolved base branch:
```
# Example output — your actual values will differ
Workspace: <workspace-name>
Base branch: main   (from .claude/pr-config.yml)
```

---

## Phase 1 — Discover and inspect repos

Determine which service folders to process based on `$ARGUMENTS` (all, one workspace,
or one service).

For each service repo, capture its state without changing anything yet:

```bash
cd [service-folder]

# Current branch (or detached HEAD)
git rev-parse --abbrev-ref HEAD

# Working tree state
git status --porcelain

# Fetch remote refs (read-only, safe)
git fetch origin --prune
```

After fetching, for the current branch compute ahead/behind relative to its upstream
and relative to the base branch:

```bash
# Ahead/behind vs the base branch for this workspace
git rev-list --left-right --count origin/[base-branch]...HEAD

# Last commit date on current branch
git log -1 --format=%cr
```

Classify each repo into exactly one state:

| State | Condition |
|-------|-----------|
| `CLEAN_ON_BASE` | Working tree clean, current branch == base branch |
| `DIRTY` | Working tree has uncommitted changes (any branch) |
| `CLEAN_ON_OTHER` | Working tree clean, on a branch other than base |
| `DETACHED` | Detached HEAD (not on any branch) |

Process each repo independently — in a multi-repo workspace, one service may be
`CLEAN_ON_BASE` while another is `CLEAN_ON_OTHER`. Never assume the whole workspace
is in one consistent state.

---

## Phase 2 — Resolve each repo by state

### State: CLEAN_ON_BASE

Fast-forward to the latest base branch:
```bash
git pull --ff-only origin [base-branch]
```

If the fast-forward fails (local has diverged from base even on the base branch —
unusual but possible if someone committed locally to main):
```
⚠ [service] is on [base-branch] but has diverged from origin/[base-branch]:
  [N] local commits not on remote, [N] remote commits not local

Options:
  A. Rebase local commits onto origin: git pull --rebase origin [base-branch]
  B. Skip this repo — resolve manually
```
Wait for response. Never auto-merge or auto-rebase a diverged base branch.

Report:
```
✓ [service]  [base-branch]  updated ([N] new commits pulled)
```

### State: DIRTY

Stop for this repo. Never pull or switch branches over uncommitted changes.
```
⚠ [service] has uncommitted changes:
  [git status --porcelain output]

Options:
  A. Stash: git stash  — I'll sync, then you restore later with git stash pop
  B. Commit them manually, then re-run /sync
  C. Skip this repo for now
```
Wait for response. If stash chosen:
```bash
git stash push -m "sync-autostash-[date]"
# proceed to sync as CLEAN_ON_BASE or CLEAN_ON_OTHER depending on branch
# after sync, remind the developer:
```
```
Note: stashed changes in [service]. Restore with: cd [service] && git stash pop
```

### State: CLEAN_ON_OTHER

This is the ambiguous case — the branch may be forgotten debris from a finished
feature, or active work in progress. Give the developer the data to decide.

```
# Example output — your actual values will differ
[service] is on '[current-branch]', not the base branch '[base-branch]'.
  vs origin/[base-branch]: [N] ahead, [N] behind
  vs origin/[current-branch]: [pushed / N unpushed commits / branch not on remote]
  last commit: [relative time, e.g. "4 days ago"]

Options:
  A. Switch to [base-branch] and update it (branch the new feature from there)
     — choose this if [current-branch] is finished/merged work
  B. Stay on [current-branch] and sync [base-branch] underneath
     — choose this if [current-branch] is active work to build on
  C. Skip this repo
```

Wait for response.

If A (switch to base):
```bash
git checkout [base-branch]
git pull --ff-only origin [base-branch]
```

If B (stay, update base in background):
```bash
# Update the local base branch ref without checking it out
git fetch origin [base-branch]:[base-branch] 2>/dev/null || \
  echo "Could not fast-forward local [base-branch] (checked-out elsewhere or diverged) — base ref updated via fetch only"
```
Report that the feature will be based on `[current-branch]`, not `[base-branch]`.

If C: skip, note it in the summary.

### State: DETACHED

Stop for this repo:
```
⚠ [service] is in detached HEAD state (not on any branch).
This usually means a specific commit or tag is checked out.

Check out a branch before syncing:
  cd [service] && git checkout [base-branch]

Then re-run /sync. Skipping this repo for now.
```

---

## Phase 3 — Context staleness check

After git sync, check whether `architecture.yml` and service `CLAUDE.md` files have
drifted from the now-current code.

This relies on a `last_synced_commit` SHA recorded in the context files (see Phase 4 —
`/sync` writes this stamp so future runs can compare against it).

### Step 3a — Read recorded SHAs

Read `architecture.yml`. For each service, look for a `last_synced_commit` field:
```yaml
services:
  policy-service:
    last_synced_commit: a1b2c3d    # written by /sync
    # ... rest of service config
```

If a service has no `last_synced_commit` (first run, or added before this feature existed),
flag it as `UNKNOWN` — context freshness cannot be verified, recommend a full
`/update-architecture` for that service.

### Step 3b — Compare recorded SHA against current HEAD

For each service with a recorded SHA:
```bash
cd [service-folder]
git rev-parse HEAD                                    # current SHA after sync
git diff [recorded-sha]..HEAD --name-only             # files changed since context built
```

If current HEAD == recorded SHA: context is fresh for this service. Nothing to do.

If they differ: scan the changed files for signals that context may now be wrong.

### Step 3c — Scan changed files for context-relevant changes

From the `git diff --name-only` output, look for changes that affect what
`architecture.yml` or `CLAUDE.md` declare:

**Backend (NestJS) — flag if changed files include:**
- Controller files (`*.controller.ts`) → endpoints may have changed (architecture.yml `exposes`)
- Event emitter calls (grep changed files for `.emit(`) → events may have changed (`publishes`)
- Event handlers (grep for `@OnEvent`) → subscriptions may have changed (`subscribes`)
- Module files (`*.module.ts`) → structure may have changed (service CLAUDE.md)
- DTO files used across services → shared contracts may have changed
- `.env.example` → env vars may have changed (architecture.yml `env_vars`)

**Frontend (Next.js) — flag if changed files include:**
- New files under `app/` or `pages/` → routes may have changed
- `next.config.*` → redirects, image workspaces may have changed
- `middleware.ts` → auth/routing behaviour may have changed
- New component folders → component conventions in CLAUDE.md may be incomplete

For each service with relevant changes, report:
```
# Example output — your actual values will differ
⚠ Context may be stale for [service]
  Context built at: [recorded-sha] ([relative time])
  Current HEAD:      [current-sha]
  [N] commits since context was generated

  Changes that may affect context:
    - 2 controller files changed → endpoints in architecture.yml may be outdated
    - .env.example changed → env_vars in architecture.yml may be outdated
    - 1 new module added → policy-service/CLAUDE.md may be incomplete

  Recommended:
    /update-architecture           (refresh service relationships and endpoints)
    /update-service-context [service]  (refresh this service's CLAUDE.md)
```

If no context-relevant files changed (e.g. only test files or internal refactors):
```
✓ [service]: [N] commits since context built, but no endpoint/event/structure changes
  Context is likely still accurate. No update needed.
```

### Step 3e — CodeGraph index + graph-vs-declared validation

CodeGraph (see `.claude/context-policy.md`) provides structural ground truth. This step
keeps its index present and uses it to mechanically validate `architecture.yml` — the
strongest available drift check (reality vs declared), per policy question-type C.

**Ensure the index exists per synced service:**
```bash
cd [service-folder]
codegraph status 2>/dev/null        # is CodeGraph available at all?
```
- CLI not found: skip this step, print once:
  `ℹ CodeGraph not installed — skipping structural validation (see .claude/context-policy.md).`
  Staleness detection falls back to the Step 3c changed-file heuristic only.
- Available but no `.codegraph/` index for the service:
  ```
  [service] has no CodeGraph index. Build it now? (recommended, one-time)
    A. Yes — codegraph init (indexes this repo; auto-syncs thereafter)
    B. Skip — use the changed-file heuristic for this service
  ```

**Validate declared vs real** (services with an index): compare architecture.yml against
the graph — declared `exposes` vs detected routes, `publishes`/`subscribes` vs real
emit/handler edges, `calls` vs real cross-service edges. Per policy, CodeGraph wins:
```
# Example output — your actual values will differ
Graph vs architecture.yml — [service]
  ⚠ POST /api/renewals in code but not declared
  ⚠ event 'policy.cancelled' emitted but not in `publishes`
  ⚠ declared call → billing-service not found in code (declaration may be obsolete)
  ✓ everything else matches
  → architecture.yml is stale. Run /update-architecture to reconcile.
```
Advisory only — never edits architecture.yml. Converts "files changed, might be stale"
into a concrete list of exactly what drifted.

### Step 3d — Context check summary

```
# Example output — your actual values will differ
Context staleness summary
  ✓ [service-a]   fresh · graph matches architecture.yml
  ⚠ [service-c]   graph shows undeclared endpoints/events — architecture.yml stale
  ? [service-d]   no recorded SHA and no CodeGraph index

Recommended before /prd:
  /update-architecture
  /update-service-context [service-c]
```

This phase only detects and recommends — it does not auto-run `/update-architecture`.
The developer decides whether to refresh context based on the changes reported.

---

## Phase 4 — Stamp synced SHAs

After a successful sync, record the current HEAD SHA of each synced service so future
`/sync` runs have a baseline to compare against.

For each service that was synced (not skipped), update its entry in `architecture.yml`:
```yaml
services:
  policy-service:
    last_synced_commit: [current-HEAD-sha]
    last_synced_date: [today's date]
    # ... rest of service config unchanged
```

Only update the `last_synced_commit` and `last_synced_date` fields — do not touch any
other part of `architecture.yml`.

**Important distinction:** `last_synced_commit` tracks when the repo was last pulled.
It is updated on every `/sync`. The context freshness check in Phase 3 compares against
a *separate* baseline — the SHA at which `/update-architecture` last ran. To support this,
`/update-architecture` and `/update-service-context` should record their own
`context_built_commit` SHA (a one-line addition to those commands).

If `context_built_commit` exists, Phase 3 compares against it. If only `last_synced_commit`
exists, Phase 3 compares against that as a fallback (less precise but still useful).

---

## Completion

```
# Example output — your actual values will differ
════════════════════════════════════════════════════
SYNC COMPLETE
════════════════════════════════════════════════════

Git sync:
  ✓ [service-a]  main     updated (3 commits pulled)
  ✓ [service-b]  develop  updated (0 commits — already current)
  ⊙ [service-c]  on feature/PROJ-100 — stayed (active work), base synced
  ⊘ [service-d]  skipped (dirty — changes stashed)

Context staleness:
  ✓ 2 services fresh
  ⚠ 1 service may have stale context — see recommendations above
  ? 1 service has no baseline

Next steps:
  [If context warnings:] Run /update-architecture before /prd to refresh context
  [If stashes created:]  Restore stashed work: cd [service] && git stash pop
  [If all clean:]        Ready to run /prd

════════════════════════════════════════════════════
```

Do not run `/prd` or `/update-architecture` automatically. Stop here and let the
developer act on the recommendations.

---

## Notes on safety

- `git fetch` is always safe and runs on every repo. Pulling, switching, and stashing
  only happen with explicit handling per state.
- Never switch branches without developer confirmation — the `CLEAN_ON_OTHER` state
  always asks.
- Never pull over a dirty tree — `DIRTY` always stops first.
- `--ff-only` is used for all pulls — this refuses to create a merge commit and fails
  loudly if the branch has diverged, rather than silently merging.
- The context check is advisory only — it never modifies context files. It tells the
  developer what might be stale and lets them decide.
