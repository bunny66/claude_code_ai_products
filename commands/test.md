# Execute testing plan

Execute the testing plan from the approved plan file, running each test category in
the correct order, reporting results, and stopping on failures that need human decisions.

This command runs after `/implement` completes. It does not write new test code —
`/implement` wrote tests alongside each task. This command runs those tests, fills gaps
the testing plan identified, validates cross-service contracts, and produces a test
report used by `/raise-pr`.

## Arguments
Optional: `$ARGUMENTS`
- No arguments: locate the approved plan from the current session
- A plan file path: `plans/<feature-slug>-plan.md`
- `--unit-only`: run unit tests only, skip integration, E2E, and contract tests
- `--skip-e2e`: run everything except E2E tests (useful when E2E environment is unavailable)

---

## Pre-flight checks

### Check 1 — Locate the plan file

Same logic as `/implement` Check 1. If a plan file was used in this session, use it.
Otherwise search for `plans/*-plan.md`.

Read the plan file and verify:
- `> Status: APPROVED` header line exists
- All non-deployment tasks are marked `[x]` — implementation is complete

If any non-deployment task is still `[ ]`, stop:
```
✗ Implementation is not complete. [N] tasks are still pending:
  [ ] [task-id] [title]
  [ ] [task-id] [title]

Run /implement to complete these tasks before running /test.

Override: if you want to test only completed tasks, run:
  /test --partial
```

### Check 2 — Read plan testing section

Read the `Testing plan` section from the plan file. Extract:
- All unit test requirements per service
- All integration test requirements per service
- All cross-service E2E scenarios
- All contract test requirements
- All event flow test requirements
- All data-quality, model-evaluation, and reproducibility requirements
- All regression test requirements
- All edge case requirements
- Test environment requirements (seed data, mocks, feature flags, services needed)
- Excluded items

### Check 3 — Read service CLAUDE.md files for test conventions

For each affected service, read its `CLAUDE.md` test patterns section.
This is what determines:
- Test file location convention
- Mock/fixture and dataset/version conventions
- Test framework and runner commands
- Evaluation tools and metric/baseline definitions, if applicable
- End-to-end tool or workflow (browser, pipeline, batch job, serving request, or none)
- How to run the full test suite vs targeted tests

### Check 4 — Verify test environment

Before running any tests, check the environment requirements from the testing plan.

For each requirement, verify:

**Services and assets required by the plan:**
Check reachability only for components that expose a health endpoint. Verify access to
required datasets, model artifacts, registries, object stores, compute, and credentials
through the project's existing checks; never print secret values.
```bash
# Example only when the component exposes an HTTP health endpoint
curl -f http://localhost:[PORT]/health 2>/dev/null && echo "UP" || echo "DOWN"
```

If a required service is down, ask:
```
⚠ Test environment incomplete

Required services:
  ✓ [service-a]   UP  (localhost:[PORT])
  ✗ [service-b]   DOWN — required for integration and E2E tests

Options:
  A. Start the service manually, then press enter to continue
  B. Skip integration and E2E tests, run unit tests only
  C. Abort — resolve environment issues first
```
Wait for response.

**Seed data:**
If the testing plan requires seed data, check if it exists or needs to be created:
```
Testing plan requires seed data: [description from plan]
Has this seed data been loaded? (yes / no — I'll create it)
```
If no: generate and run the seed script following the pattern in the service's existing
seed files. If no seed pattern exists, ask the developer.

**External mocks:**
If the testing plan lists external services to mock (SMTP, third-party APIs), verify
the mock server is running or the test framework handles it (e.g. MSW auto-starts).

**Feature flags:**
If the testing plan specifies a feature flag state, verify it is set correctly in
the test environment config before running any tests.

---

## Test execution

Run tests in this strict order. Do not skip levels unless explicitly requested.

### Level 1 — Unit tests (per component)

For each affected component, use the test runner and command documented by its
`CLAUDE.md`, project configuration, or existing CI workflow. Never assume npm, Jest,
or a particular coverage threshold.

```bash
cd [component-folder]
[project-native targeted test command]
```

If targeted execution is unavailable, run the narrowest relevant existing suite. Include
tests for changed functions, data transformations, model interfaces, APIs, and pipeline
stages as applicable. Follow each component's skill and `CLAUDE.md` test conventions.

**If any unit test fails:**
```
✗ Unit test failure in [service-name]

Failed: [test name]
File:   [file path]
Error:  [error message]

This test was written during /implement. Options:
  A. I investigate and fix the test or implementation
  B. Skip this test (marks as TODO in test report — will block PR)
  C. Abort testing session
```
Wait for response. Do not proceed to Level 2 until all Level 1 failures are resolved
or explicitly skipped.

Print per-service unit test summary:
```
# Example output — your actual values will differ
Unit tests — <component-name> ([role])
  ✓ [N] passed
  ✗ [N] failed  ← if any
  ○ [N] skipped
  Coverage: [reported value, if configured]
```

---

### Level 2 — Integration tests (per service)

For each component, run integration tests for the changed boundaries using its
project-native commands:

```bash
cd [component-folder]
[project-native integration test command]
```

Cover relevant system boundaries such as data storage, pipeline stages, model loading,
serving APIs, external dependencies, and schema migrations. Verify expected success,
invalid input, failure handling, and idempotency where applicable. Start only the test
dependencies documented by the project.

**If any integration test fails:**
Same decision flow as Level 1 failures — stop, report, and wait for resolution.

Print per-service integration test summary:
```
# Example output — your actual values will differ
Integration tests — <service-name>
  ✓ [N] passed
  ✗ [N] failed
  Duration: [N]s
```

---

### Level 3 — Contract tests (cross-service, run if plan requires them)

Run only if the testing plan includes contract checks. Contracts may include API schemas,
feature or dataset schemas, model input/output signatures, serialized artifacts, or
cross-component file formats.

Verify producer and consumer agree using the project's schema validators, generated
types, compatibility tests, or artifact inspection tools. For model changes, compare
input/output names, shapes, dtypes, required preprocessing, and serialization/runtime
compatibility against the declared contract. Do not infer contract compatibility from
successful compilation alone.

Print contract test summary:
```
# Example output — your actual values will differ
Contract tests
  ✓ Declared contracts match between <component-a> and <component-b>
  [or:]
  ✗ Type mismatch: <FieldName> in <DtoName> — see above
```

---

### Level 4 — Data and model evaluation (when applicable)

Run the data-quality and model-evaluation checks from the approved testing plan and the
component's established evaluation workflow. Use only the agreed evaluation datasets,
metrics, baselines, and acceptance criteria; do not invent thresholds or substitute a
convenient dataset for the representative one.

For every evaluated result, record the dataset/version, evaluation configuration, metric
definition, relevant slice or subgroup checks, and comparison baseline. Check for data
leakage and train/evaluation contamination when the change affects data splits or training.
Report uncertainty, variance, known limitations, and incomplete evaluation explicitly.
For data pipeline changes, validate schema, missingness, ranges, distributions, and
other domain invariants defined by the project.

If no agreed evaluation criteria exist for a model-affecting change, stop and ask the
developer to define them rather than declaring model quality passed.

Print a data/model evaluation summary:
```
Data/model evaluation — <component-name>
  Dataset/version: [identifier]
  Checks/metrics:  [names and results]
  Baseline:        [comparison or none defined]
  Limitations:     [uncertainty, variance, or known gaps]
  Result:          [PASS / FAIL / INCOMPLETE]
```

### Level 5 — Event flow tests (cross-component, run if plan requires them)

Run only if the testing plan includes `Event flow tests` entries.

These tests verify the full publish → subscribe chain across services.

**Setup:**
Both the publisher service and subscriber service must be running.
Redis (or the configured event transport) must be running.

For each event flow in the testing plan:

```bash
# Trigger the publisher side (e.g. via API call or direct function call in test)
# Verify the subscriber side received and processed it (check DB state or mock assertion)
```

Approach depends on what test infrastructure exists in the component:
- If there are existing event flow tests: run the documented project-native command
- If no tests exist yet: write a new integration test that:
  1. Calls the endpoint/function that triggers the event
  2. Waits for the subscriber to process (with a timeout)
  3. Asserts the subscriber's side effect (DB record created, notification sent, etc.)

**Idempotency check:**
For each new event subscriber, trigger the event twice and verify the result is
identical to triggering it once (no duplicate records, no duplicate notifications).

Print event flow test summary:
```
# Example output — your actual values will differ
Event flow tests
  ✓ [event.name]: publisher → subscriber — processed correctly
  ✓ [event.name]: idempotency verified (triggered twice, one result)
  ✗ [event.name]: subscriber timed out — see above
```

---

### Level 6 — Regression tests

Identify which existing tests are actually affected by the changes. When CodeGraph is
available (per `.claude/context-policy.md`), use it to trace affected tests structurally
rather than guessing from the plan's regression list alone:
```bash
git diff --name-only [base]...HEAD | codegraph affected --stdin
```
This traces import dependencies transitively to the test files that exercise the changed
code — catching regressions the plan's `Regression` section may not have anticipated.
Merge these with the plan's listed scenarios. If CodeGraph is unavailable, use the plan's
`Regression` section plus grep for tests referencing changed modules.

For each affected item, run the relevant existing test suite to verify unchanged behaviour:

Use the project's native command to run affected existing regression tests.

If no existing tests cover the regression scenario, write a regression test now:
```
⚠ No existing test covers regression scenario:
  "[existing flow] — verify [specific behaviour]"

I'll write this test now before proceeding.
[write the test, run it, confirm it passes]
```

A new regression test written here is committed to the branch before `/raise-pr`.

Print regression summary:
```
# Example output — your actual values will differ
Regression tests
  ✓ [N] existing scenarios verified unchanged
  + [N] new regression tests written and passing
```

---

### Level 7 — End-to-end tests (skip with --skip-e2e)

Run end-to-end tests for the workflows in the testing plan, such as data ingestion to
artifact production, training to registered model, or request to inference response.
Include UI browser tests only when a user interface is in scope.

**If the project uses Playwright:**
```bash
cd [UI-component-folder]
[project-configured Playwright command] --grep "[feature-name]"
```

If no tagged E2E tests exist for the new pages/flows added by this feature, write them:

For each `new page` task in the plan:
1. Create or update a Playwright test file: `e2e/[feature]/[page-name].spec.ts`
2. Cover: page loads correctly, primary user action succeeds, error state shown on failure

```typescript
// Playwright test pattern — match existing e2e/ test structure
test.describe('[Feature name]', () => {
  test('user can [primary action]', async ({ page }) => {
    await page.goto('/[route]');
    await expect(page.getByRole('heading', { name: '[Page title]' })).toBeVisible();
    // perform the primary action
    // assert the outcome
  });

  test('shows error when [failure condition]', async ({ page }) => {
    // mock the API to return an error
    // assert the error state is shown
  });
});
```

**If the project uses Cypress:**
```bash
cd [UI-component-folder]
[project-configured Cypress command] --spec "cypress/e2e/[feature]/**"
```

**If E2E environment is missing (services not running):**
```
⚠ E2E tests require these services to be running:
  [list from testing plan]

Not all are available. Options:
  A. Start missing services and press enter
  B. Skip E2E — they will be flagged as not run in the test report
     (PR can still be raised but reviewer will be notified)
  C. Abort
```

Print E2E summary:
```
# Example output — your actual values will differ
E2E tests ([Playwright/Cypress])
  ✓ [N] scenarios passed
  ✗ [N] scenarios failed
  ○ [N] not run (environment unavailable)
  Duration: [N]s
```

---

### Level 8 — Coverage and project quality gates

After all levels complete, run the coverage and quality gates defined by the project
and approved plan. Use the configured command; do not assume a runner or global
percentage threshold.

```bash
[project-native coverage or quality-gate command]
```

Keep code coverage distinct from data-quality and model-quality metrics. Apply only
thresholds that the repository or approved testing plan actually defines. If none are
defined, report measured values without labeling them as passed/failed thresholds.

If below threshold on any changed file:
```
⚠ Coverage below threshold

File: [path]
  Lines:    [N]% (configured target, if any)
  Branches: [N]% (configured target, if any)
  Missing:  [list uncovered lines]

Options:
  A. I write the missing tests now
  B. Accept below-threshold — noted in test report (reviewer will see this)
  C. Propose a threshold change — requires reviewer agreement
```

---

## Writing the test report

After all levels complete, write the full test report to the plans folder.

Report file: `plans/<feature-slug>-test-report.md`

```markdown
# Test report — [Feature name]
> Feature: [feature-slug]
> Branch:  [branch-name]
> Date:    [today's date]
> Status:  PASSED | PASSED WITH NOTES | FAILED

---

## Summary

| Level | Component/Scope | Checks | Passed | Failed | Skipped | Results/metrics |
|-------|-----------------|--------|--------|--------|---------|-----------------|
| Unit | [component] | [N] | [N] | [N] | [N] | [configured coverage, if any] |
| Integration | [component] | [N] | [N] | [N] | [N] | — |
| Data/model | [component] | [N] | [N] | [N] | [N] | [metric/baseline or N/A] |
| Contract | cross-component | [N] | [N] | [N] | [N] | — |
| Event flow | cross-component | [N] | [N] | [N] | [N] | — |
| Regression | all affected | [N] | [N] | [N] | [N] | — |
| End-to-end | affected workflow | [N] | [N] | [N] | [N] | — |
| Reproducibility | applicable artifacts | [N] | [N] | [N] | [N] | versions/lineage |

---

## Failures (if any)
[List each failure with: test name, error, resolution taken]

---

## Skipped tests (if any)
[List each skipped test and why — these are flagged for reviewer attention]

---

## New tests written during /test
[List any regression or E2E tests written during this command, not during /implement]

---

## Known gaps
[Anything in the testing plan that was not testable due to environment or time constraints]

---

## Environment
- Services running: [list]
- Seed data: [used / not needed]
- Dataset/model/artifact versions: [list or "not applicable"]
- External mocks: [list or "none"]
- Feature flag state: [on/off/n/a]
```

Set report status:
- `PASSED` — all required checks passed and configured thresholds were met
- `PASSED WITH NOTES` — checks were skipped or a documented threshold exception was accepted;
  model/data evaluation remains incomplete if agreed criteria were not verified
- `FAILED` — one or more tests failed and were not resolved

---

## Completion

```
# Example output — your actual values will differ
════════════════════════════════════════════════════
TEST RUN COMPLETE
════════════════════════════════════════════════════

Feature:  <feature-slug>
Branch:   feature/<ticket-id>-<feature-slug>
Status:   PASSED | PASSED WITH NOTES | FAILED

Results:
  Unit:        [N] passed, [N] failed, [N] skipped
  Integration: [N] passed, [N] failed
  Contract:    [N] passed, [N] failed
  Event flow:  [N] passed, [N] failed
  Regression:  [N] passed, [N] new written
  E2E:         [N] passed, [N] failed, [N] not run

Coverage (changed files):
  <service-a>: [N]% lines, [N]% branches
  <service-b>: [N]% lines, [N]% branches

Report saved to:
  plans/<feature-slug>-test-report.md

════════════════════════════════════════════════════
```

If status is `FAILED`:
```
✗ Test run has failures. Resolve before raising a PR.
  See report: plans/<feature-slug>-test-report.md

Run /test again after fixing, or run /raise-pr with --force to raise PR despite failures
(not recommended — reviewer will see the failed status in the report).
```

If status is `PASSED` or `PASSED WITH NOTES`:
```
✓ Ready to raise a PR.
Run /raise-pr to create the Bitbucket PR with the test report attached.
```

Do not run `/raise-pr` automatically. Stop here.
