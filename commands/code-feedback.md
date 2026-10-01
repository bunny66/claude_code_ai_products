# Apply feedback to an implemented feature

Take feedback on code produced by `/implement`, classify it, update the PRD summary and
plan as appropriate, get approval, then implement the changes on the same feature branch.

Runs after `/implement`. Does NOT run tests — run `/test` yourself afterward.

## Arguments
`$ARGUMENTS` — one of:
- Inline feedback text: `/code-feedback the renewal banner should be dismissible and persist the dismissal`
- A feedback file: `/code-feedback ./feedback/round1.md` (`.md` / `.txt`)
- A plan path + feedback, when multiple features are in flight:
  `/code-feedback plans/<slug>-plan.md ./feedback.txt`

If empty, stop:
```
✗ No feedback provided.
  /code-feedback "<your feedback>"
  /code-feedback ./feedback/round1.md
```

---

## Pre-flight

### Check 1 — Locate the feature

Find the plan file (session context → explicit path → search `plans/*-plan.md`).
If multiple and none specified, list them and ask which feature the feedback targets.

Verify the plan header has `> Status: APPROVED` and that implementation has run — at
least some tasks marked `[x]`. If nothing is implemented yet:
```
⚠ No tasks are implemented yet for this feature. Feedback on unimplemented code
  doesn't apply. Did you mean to revise the plan instead? Run /plan with feedback.
```

Read into context: the plan file, the PRD summary (`prd-summary-<slug>.md`), each
affected service's `CLAUDE.md`, resolved skills, and the branch name from the plan header.

### Check 2 — Load the feedback

Read inline text or the feedback file. If a file, support `.md`/`.txt`. Preserve the
feedback verbatim — it becomes part of the audit trail.

---

## Phase 1 — Classify each feedback item

Feedback is often several points at once. Split it into discrete items and classify each:

| Class | Meaning | PRD impact |
|-------|---------|-----------|
| `requirement change` | New/changed behaviour not in the original PRD | Updates PRD summary |
| `bug fix` | Implemented code doesn't do what the PRD/plan already specified | No PRD change |
| `refinement` | Cosmetic/quality tweak within existing scope (naming, spacing, copy, minor UX) | No PRD change |

Print the classification and let the developer correct it — classification drives
everything downstream, so it must be right:
```
# Example — your items will differ
Feedback classified into [N] items:

  1. [requirement change] Banner must persist dismissal across sessions
       → not in original PRD; will update PRD summary
  2. [bug fix] Renewal date shows in UTC, PRD said user's local timezone
       → PRD already requires local; implementation is wrong
  3. [refinement] Button label "Submit" → "Confirm renewal"
       → cosmetic, within scope

Is this classification correct? (enter to accept, or correct any item)
```
Wait for confirmation.

---

## Phase 2 — Clarify (only if needed)

For any item that's ambiguous or under-specified, ask — batched, numbered, once.
Prefer answers from the PRD/plan/skills before asking. `requirement change` items are
the most likely to need clarification (they introduce new intent); `bug fix` and
`refinement` items usually don't.

If an item can't be actioned without an answer and none is given, mark it deferred and
exclude it from this round rather than guessing.

---

## Phase 3 — Update the PRD summary (requirement-change items only)

Only `requirement change` items touch the PRD. `bug fix` and `refinement` items never do.

For each requirement-change item, amend `prd-summary-<slug>.md`:
- Add/modify the relevant functional requirement, tagged with source `[Feedback R<n>]`
  (where `<n>` is the feedback round — see Phase 4 for the round number)
- If it introduces a new UI surface, extend the Frontend page map (route/file/change type)
  and, if a mockup or design note came with the feedback, capture its visual spec
- Append a dated note under a `## Feedback history` section recording the raw feedback item

Do not rewrite unrelated parts of the PRD. Preserve everything else verbatim.

If no requirement-change items exist, skip this phase and note "PRD unchanged — no
requirement changes in this feedback."

---

## Phase 4 — Amend the plan (append a feedback round)

Determine the round number: scan the plan for existing `Feedback round N` sections;
this round is N+1 (first feedback round = 1).

Append a new section to the plan file — never rewrite existing tasks (they're already
implemented and committed):

```
━━━ FEEDBACK ROUND [N] ━━━━━━━━━━━━━━━━━━━━━━━━━
Source: [inline / filename] · Date: [today]
Raw feedback recorded in PRD summary Feedback history.

[ ] FB[N]-1. [Task title]
    Origin:     feedback item [#] ([requirement change | bug fix | refinement])
    Service:    [service-name]
    Skills:     [resolved skills]
    Type:       [task type from a resolved skill's contributed types]
    What:       [specific change]
    Files:      [files to touch]
    Depends on: [prior task IDs if any]
    Risk:       [low | medium | high]

[ ] FB[N]-2. ...
```

Task IDs use the `FB[round]-[n]` prefix so feedback tasks are distinguishable from
original `S`/`FE`/`C` tasks in the same file. Apply the same ordering rules as `/plan`
(dependencies, migrations before code, API before consuming UI, etc.). Write to both
the plan file.

---

## Phase 5 — Approval gate (weight depends on classification)

**If the round contains any `requirement change` item → full gate.**
Present the PRD changes, the new tasks, areas of impact, and risk, then:
```
This feedback round changes requirements. Review the PRD updates and new tasks above.
Type APPROVE to implement, or give feedback to revise.
```
Wait for `APPROVE`.

**If the round is only `bug fix` and/or `refinement` → lighter gate.**
Show the task list and proceed unless the developer objects:
```
Feedback round [N]: [N] fixes/refinements, no requirement changes.
[task list]
Proceeding to implement unless you object. (press enter to continue, or comment to change)
```
Brief pause for objection, then continue. Still stop before any task marked `Risk: high`
even in the lighter path.

On APPROVE (full gate), stamp the plan header:
```
> Feedback round [N]: APPROVED [date]
```

---

## Phase 6 — Implement the feedback tasks

Follow the same execution model as `/implement`, scoped to this round's `FB[N]-*` tasks:
- Work per service, respecting the resolved skills' implementation and test rules
- Pattern discovery is already known from the original run; re-scan only a service not
  previously touched in this feature
- Write/adjust tests alongside each change per the skills' test rules
- Run each skill's verification commands before committing
- **Commit with the `feedback:` prefix** to distinguish from original work:
  ```
  git commit -m "feedback: [FB[N]-x] [task title]

  Round [N] · origin: [requirement change | bug fix | refinement]
  Feature: [feature-slug]"
  ```
- Mark each task `[x]` in the plan file as it completes
- Stop before any `Risk: high` task and ask, exactly as `/implement` does
- After each service's feedback tasks, checkpoint before the next service

If a bug-fix reveals the root cause is broader than the reported symptom, stop and
report rather than silently expanding scope — offer to add a task or keep it narrow.

---

## Completion

```
# Example — your values will differ
════════════════════════════════════════════════════
FEEDBACK ROUND [N] IMPLEMENTED
════════════════════════════════════════════════════

Feature:  <feature-slug>
Branch:   feature/<ticket-id>-<feature-slug>
Items:    [N] requirement changes, [N] bug fixes, [N] refinements

Changes:
  PRD summary:  [updated for N requirement changes / unchanged]
  Plan:         Feedback round [N] appended, [N] tasks, all [x]
  Commits:      [N] feedback: commits on the feature branch

Deferred (need answers, not done this round):
  [item] — [what's blocking]

════════════════════════════════════════════════════

Run /test to verify the changes.
Then /raise-pr — or if a PR already exists, push updates the same branch.
```

Do not run `/test` or `/raise-pr` automatically. Stop here.

---

## Notes

- Multiple feedback rounds accumulate: round 1, round 2, … each appended, never
  overwriting prior rounds. The plan file becomes the full history of the feature.
- If a PR was already raised, the new `feedback:` commits push to the same branch and
  appear on the existing PR automatically — no new PR needed.
- This command never runs tests. Test status in any existing PR/test-report goes stale
  the moment feedback is implemented — re-running `/test` is the developer's explicit step.
