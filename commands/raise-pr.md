# Raise a Bitbucket PR and notify Slack

Compose the PR description from plan and test report artifacts, push all feature
branches, raise a PR per service repo on Bitbucket, and post a notification to the
configured Slack channel.

## Arguments
Optional: `$ARGUMENTS`
- No arguments: use plan and test report from the current session
- `--force`: raise PR even if test report status is FAILED (reviewer is warned)
- `--skip-slack`: raise the PR but skip the Slack notification
- A plan file path: `plans/<feature-slug>-plan.md`

---

## Configuration

Each workspace has its own `.claude/pr-config.yml`. There are no per-workspace overrides —
a workspace is the unit of configuration.

```yaml
# .claude/pr-config.yml
slack_channel_id: ""        # channel to notify for this workspace's PRs
bitbucket_workspace: ""     # Bitbucket org slug
default_target_branch: "main"
repo_name_overrides: {}     # optional: service-folder-name -> bitbucket-repo-slug
```

If any required field is empty, ask only for the missing ones and write them back.
If the file is missing entirely:
```
No pr-config.yml found in this workspace. I need:
  1. Slack channel ID for PR notifications
     (right-click channel → View channel details → Channel ID)
  2. Bitbucket workspace slug
     (the org part of bitbucket.org/[workspace]/repo)
  3. Default target branch (main / master / develop) [default: main]
```
Write the answers to `.claude/pr-config.yml`, then continue.

Print the resolved config once before proceeding:
```
# Example output — your actual values will differ
Workspace:     <workspace-name>
Slack channel: <channel-id>
Target branch: main
Bitbucket:     <bitbucket-workspace>
Correct? (enter to confirm)
```

## Pre-flight checks

### Check 1 — Locate plan file

Same logic as `/implement` and `/test`. Find the approved plan file from session
context or search `*/plans/*-plan.md`.

Read from the plan file header:
- `> Feature:` — feature slug
- `> Branch:` — branch name
- `> Services:` — comma-separated service list with (backend)/(frontend) labels
- `> Workspace:` — this workspace's name
- `> External contracts touched:` — contracts other workspaces consume, or "none"

### Check 2 — Verify test report exists

Look for `plans/<feature-slug>-test-report.md`.

If test report is missing:
```
✗ No test report found for this feature.

Expected: plans/<feature-slug>-test-report.md

Run /test before raising a PR, or explain why tests were skipped:
  A. Run /test now
  B. Raise PR without test report — reviewer will be notified tests were not run
```
Wait for response. If B: note `tests_run: false` and continue.

Read the test report and extract:
- `> Status:` — PASSED / PASSED WITH NOTES / FAILED
- Summary table (pass/fail counts per level)
- Skipped tests list
- Known gaps

If status is `FAILED` and `--force` was not passed:
```
✗ Test report status is FAILED.

Failed tests must be resolved before raising a PR.
Run /test to fix failures, then re-run /raise-pr.

To raise the PR anyway (not recommended):
  /raise-pr --force
```
Stop unless `--force` was passed.

If `--force` was passed with a FAILED status, note it prominently in the PR description.

### Check 3 — Verify all branches are pushed

For each service in the plan, check if the feature branch has been pushed to Bitbucket:

```bash
cd [service-folder]
git status -sb   # shows tracking branch
git log origin/[branch-name]..[branch-name] --oneline 2>/dev/null | wc -l
```

If there are unpushed commits in any service:
```
⚠ Unpushed commits found:

  <service-a>: [N] commits ahead of origin
  <service-b>: up to date

Pushing now...
```

Push all unpushed branches:
```bash
cd [service-folder]
git push origin [branch-name]
```

If push fails (e.g. no upstream set):
```bash
git push --set-upstream origin [branch-name]
```

If push is rejected (remote has changes):
```
✗ Push rejected for <service-name> — remote branch has diverged.

This usually means someone else pushed to this branch.
Options:
  A. Pull and rebase: git pull --rebase origin [branch-name]
  B. Abort — resolve manually then re-run /raise-pr
```
Wait for response.

### Check 4 — Confirm target branch

Read `default_target_branch` from `.claude/pr-config.yml`. Every service repo in this
workspace targets that branch. Verify it exists on each remote before raising PRs.

For each service repo, verify the target branch exists on remote:
```bash
git ls-remote --heads origin [target-branch]
```

If target branch does not exist, ask:
```
Target branch '[branch]' not found in <service-name>.
What branch should this PR target?
  Common options: main, master, develop, staging
```

---

## Compose PR description

Generate one PR description. This is used for all service PRs for this feature
(reviewers should see the same context regardless of which service repo they open).

Use this exact structure:

```markdown
## [Feature title from PRD]

[Ticket ID — link if pattern matches PROJ-123 format, else plain text]
[PRD Google Doc link from prd-summary file if available]

---

### What this PR does

[2–3 paragraph summary derived from:
 - The feature objective from the PRD summary
 - The implementation plan's feature summary
 - The services changed and why]

---

### Services changed

| Service | Workspace | Type | Changes |
|---------|--------|------|---------|
[One row per service. Changes column: brief list of what changed — endpoint added,
migration, event published, component added, etc.]

---

### Implementation approach

[One paragraph describing the technical approach.
Derive from the implementation plan tasks — what was built and key design decisions.
Include any developer-delegated decisions from Phase 7 of the plan.]

---

### Areas of impact

[Summarise the Areas of Impact section from Phase 3 of the plan.
Call out breaking changes, new env vars, and deploy order explicitly.]

**Breaking changes:** [Yes — describe / No]
**New environment variables:** [list or "None"]
**Deploy order:** [service-a before service-b — reason / No constraint]
**DB migrations:** [Yes — run before deploying / No]

---

### Test results

[If test report exists:]
| Level | Passed | Failed | Skipped |
|-------|--------|--------|---------|
[rows from test report summary table]

**Overall status:** ✅ PASSED / ⚠️ PASSED WITH NOTES / ❌ FAILED

[If PASSED WITH NOTES: list skipped tests and known gaps]
[If FAILED + --force: large warning block explaining what failed]
[If no test report: "⚠️ Tests not run — see reviewer notes below"]

---

### Checklist

- [ ] I have reviewed the code changes in each service
- [ ] Breaking changes have been communicated to affected teams
- [ ] New env vars have been added to staging config
- [ ] DB migrations are ready to run before deployment
- [ ] Deployment order has been noted
[Add or remove items based on what's relevant to this feature]

---

### Reviewer notes

[Any items from the plan's open items, delegated decisions, or test gaps
that the reviewer should pay specific attention to.]

[If --force with FAILED tests:]
> ⚠️ WARNING: This PR was raised with failing tests. Do not merge until resolved.
> Failed tests: [list]
```

---

## Raise PRs on Bitbucket

For each service in the plan, raise a PR using the Bitbucket `bb` CLI.

### Step 1 — Verify `bb` CLI is available

```bash
which bb
bb --version
```

If `bb` is not installed:
```
✗ Bitbucket CLI (bb) not found.

Install it: https://bitbucket.org/atlassian/bb/src/master/
Or raise the PR manually at: https://bitbucket.org/[workspace]/[repo]/pull-requests/new

After raising manually, provide the PR URL so I can post it to Slack:
```
Wait for manual PR URL if CLI is unavailable. Skip to Slack step.

### Step 2 — Raise PR per service

For each service in the plan:

```bash
cd [service-folder]

bb pr create \
  --title "[ticket-id]: [feature-title]" \
  --description "[pr-description]" \
  --source [branch-name] \
  --destination [target-branch] \
  --no-prompt
```

If `bb` does not support `--description` inline (some versions require a file):
```bash
# Write description to temp file first
cat > /tmp/pr-description.md << 'EOF'
[pr-description]
EOF

bb pr create \
  --title "[ticket-id]: [feature-title]" \
  --description-file /tmp/pr-description.md \
  --source [branch-name] \
  --destination [target-branch] \
  --no-prompt
```

Capture the PR URL from the `bb` output.

If PR creation fails:
```
✗ Failed to create PR for <service-name>:
  [bb error output]

Options:
  A. Retry after fixing the issue above
  B. Raise manually and provide the URL
  C. Skip this service's PR (note it as missing in Slack notification)
```

### Step 3 — Collect all PR URLs

After raising PRs for all services, collect:
```
PRs raised:
  ✓ <service-a>: https://bitbucket.org/[workspace]/[service-a]/pull-requests/[N]
  ✓ <service-b>: https://bitbucket.org/[workspace]/[service-b]/pull-requests/[N]
```

---

## Post Slack notification

One workspace, one channel — post a single message covering every PR raised.

```
:git: *PR raised — [feature-title]* [ticket-id if present]

[One sentence: what the feature does]

*Services changed:*
• <[PR URL]|[service-a]> ([N] commits)
• <[PR URL]|[service-b]> ([N] commits)

*Tests:* [✅ PASSED / ⚠️ PASSED WITH NOTES / ❌ FAILED / ⚠️ Not run]
[If not clean: one line on what needs attention]

*Branch:* `[branch-name]`  →  `[target-branch]`
[If breaking changes:] :warning: *Breaking change — coordinate deploy*
[If new env vars:] :key: *New env vars required before deploy*
[If deploy order matters:] :arrow_right: *Deploy order: [service-a] → [service-b]*
[If external contract touched:] :satellite: *Affects workspace '[name]' — [contact] notified separately*

_Raised by Claude Code — review and add reviewers in Bitbucket_
```

Send via the Slack `send_message` tool using `slack_channel_id` from
`.claude/pr-config.yml`.

If the plan header lists `External contracts touched`, add a line naming the affected
workspace and its contact — the other team owns their side of the change and this PR
cannot make it for them.

If the Slack send fails, print the message text for manual posting and continue —
never block completion on Slack; the PR is already raised.

## Update plan file status

After all PRs are raised, append a `> PR Status:` block to the plan file header:

```markdown
> PR Status: RAISED
> PR raised: [today's date]
> PRs:
>   [service-a]: https://bitbucket.org/.../pull-requests/N
>   [service-b]: https://bitbucket.org/.../pull-requests/N
> Slack notified:
>   [channel-id-A] ([primary-workspace]): yes
>   [channel-id-B] ([secondary-workspace]): yes / no — reason if no
```

Update `plans/<feature-slug>-plan.md`.

---

## Completion

```
# Example output — your actual values will differ
════════════════════════════════════════════════════
PR RAISED
════════════════════════════════════════════════════

Feature:  <feature-slug>
Branch:   feature/<ticket-id>-<feature-slug>
Target:   <target-branch>

PRs:
  ✓ <service-a>  https://bitbucket.org/[workspace]/[repo]/pull-requests/N
  ✓ <service-b>  https://bitbucket.org/[workspace]/[repo]/pull-requests/N

Slack:    ✓ Posted to #<channel-name>
          [or: ✗ Failed — post manually]

Test status in PR: [PASSED / PASSED WITH NOTES / FAILED / Not run]

════════════════════════════════════════════════════

Next steps (manual):
  1. Open the PR links above and add reviewers in Bitbucket
  2. Ensure new env vars are added to staging config if needed
  3. Coordinate deploy order with ops if services have dependencies
  [If breaking changes:] 4. Notify downstream teams of breaking API changes
```

---

## Edge cases

**Single service feature (most common case):**
One PR is raised, one Slack message posted. No coordination needed.

**Multi-service feature with no deploy dependency:**
Multiple PRs raised independently. Slack message lists all PRs. Reviewer decides
merge order.

**Multi-service feature with deploy dependency:**
Deploy order from the plan is shown prominently in the PR description and Slack
message. A `:warning:` is added to the Slack message.

**PR already exists for this branch:**
`bb pr create` will error with "PR already exists". Handle gracefully:
```bash
# Get existing PR URL instead
bb pr list --source [branch-name] --format json | jq -r '.[0].links.html.href'
```
Use the existing PR URL in the Slack message and completion summary.
Add a note: "PR already existed — description was not updated. Update manually if needed."

**Force-push needed (after rebase):**
If the developer rebased the branch after the last push, `git push` will be rejected.
```
⚠️ Branch <service-name> needs force push (diverged from remote after rebase).

This will overwrite the remote branch history.
Confirm force push? (yes / no)
```
Never force-push without explicit confirmation.
```bash
git push --force-with-lease origin [branch-name]
```
Use `--force-with-lease` not `--force` — it aborts if someone else pushed to the branch.
