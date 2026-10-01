# Generate implementation plan

Generate a robust, review-ready implementation plan for the feature described in the current
session. This command runs after `/prd` — it expects PRD content and clarification answers
to already be in context.

If `/prd` has not been run yet, stop immediately and print:
```
✗ No PRD context found in this session.
Run /prd <google-doc-url> first, answer the clarifying questions, then run /plan.
```

---

## Inputs expected from session context
- PRD content (read by `/prd`)
- Affected services identified by `/prd`
- Clarification answers provided by developer
- `architecture.yml` — service relationships and guardrails
- Each affected service's `CLAUDE.md` — implementation constraints
- `architecture.yml` `workspace.boundary_rules` and `external:` contracts

---

## Instructions

Work through every phase below sequentially. Do not skip phases. Do not generate code.
Do not make assumptions — if something is unclear, ask before proceeding to the next phase.

---

### Phase 0 — Scope confirmation

Everything in this plan lives inside THIS workspace. A feature never spans workspaces.
This phase confirms which of the workspace's services are in scope and what each needs.

#### Step 0a — Read architecture.yml

Read `architecture.yml` from the workspace root. Extract:
- Declared services in this workspace
- `workspace.boundary_rules` — how outside workspaces may interact with this one
- `external.consumed_by` — what outside consumers depend on (breaking-change radar)

If `architecture.yml` is missing, stop and print:
```
✗ architecture.yml not found in this workspace.
Run /update-architecture first, then re-run /prd and /plan.
```

If a service named by `/prd` is not declared in `architecture.yml`, stop:
```
⚠ Service '[name]' is referenced in the PRD but not declared in architecture.yml.
Run /update-architecture to register it, then re-run /plan.
```
If the PRD names a service that belongs to a DIFFERENT workspace, stop:
```
✗ '[name]' is not part of this workspace. Features cannot span workspaces.
  Split this into a separate feature in that workspace, and record the contract
  between them under `external:` in both workspaces' architecture.yml.
```

#### Step 0b — Resolve skills per in-scope service

For each affected service, resolve skills via `.claude/skills/skill-registry.yml`
(declared `skills:` wins; otherwise detect; load `Requires:` transitively).
A service is **UI-capable** if any resolved skill is in the registry's `ui_capable` list.

```
# Example output — your actual values will differ
Services in scope
  <service-a>   [typescript, nodejs, nestjs]              backend
  <service-b>   [typescript, reactjs, nextjs, tailwind]   UI-capable
```

#### Step 0c — Read boundary rules and external contracts

From `architecture.yml`, record:
- `workspace.boundary_rules` — constraints this feature must not violate
- `external.consumed_by` — every event/endpoint an outside workspace relies on.
  Any task touching one of these is a **potential breaking change for another team**
  and must be flagged in Phase 3, even though this plan cannot change their code.

If `workspace.boundary_rules` is empty:
```
⚠ No workspace boundary rules recorded in architecture.yml.
Proceeding, but changes to externally-consumed contracts will be flagged conservatively.
```

#### Step 0d — Confirm scope with developer

```
# Example output — your actual values will differ
Feature scope (this workspace only):
  Services: <service-a>, <service-b>
  Plan will be saved to: plans/<feature-slug>-plan.md

External contracts this feature may touch:
  ⚠ event policy.renewed — consumed by workspace 'notification' (#team-notification)

Press enter to confirm, or correct the scope.
```

Wait for confirmation before proceeding.

#### Step 0e — Read frontend page map from PRD summary

If any in-scope service is UI-capable, read the `Frontend page map` section from
`plans/prd-summary-<feature-slug>.md`.

Extract and hold for Phase 4:
- Each route and its file path; change type per route
- New components required and their location; existing components to modify
- **Design vocabulary** — exact tokens and closest-analogue components to scaffold from
- **Mockup visual specs** — per-component layout/state/copy constraints
- **Design tensions** — flagged mockup-vs-convention conflicts and their resolution

If a UI-capable service is in scope but no page map exists, stop:
```
✗ A UI-capable service is in scope but no page map was found in the PRD summary.
  A. Re-run /prd — it will scan and generate the page map
  B. Provide the page map manually: route, file, change type per entry
```
Do not generate frontend tasks without a page map — vague tasks like "update the
renewal page" are not acceptable output.

### Phase 1 — Confirm understanding

Restate the feature in your own words in this format:

```
Feature summary
================
What it does:    [one paragraph, plain English, no jargon]
Who it affects:  [user roles / actors involved]
Why it matters:  [business reason from PRD, one sentence]

Workspace: [workspace-name]

Services in scope:
  - [service-name] — [why it's involved]
  - [service-name] — [why it's involved]

Services explicitly NOT in scope:
  - [service-name] — [why it's excluded]

Boundary rules that apply:
  - [rule from architecture.yml workspace.boundary_rules]

External contracts this feature may touch:
  - [event/endpoint] — consumed by workspace '[name]' ([contact])  ⚠ coordinate
  - [or "none"]

Assumptions carried forward from clarification answers:
  1. [assumption]
  2. [assumption]
```

Ask: "Is this understanding correct? Any corrections before I proceed?"
Wait for confirmation. Do not move to Phase 2 until confirmed.

---

### Phase 2 — Ambiguity elimination

Review the PRD and clarification answers for any remaining gaps. Ask ALL outstanding
questions in a single numbered list — do not drip-feed questions one at a time.

Organise by category — only include categories with actual questions:

```
Remaining questions before I can plan
=======================================

Business logic
  1. [question]

Data and state
  2. [question]

Domain boundaries
  3. [question — e.g. "Should <service-b> store a copy of the data, or
     always fetch it from <service-a> via REST?"]

Service interactions
  4. [question — e.g. "Should this be a synchronous REST call or an async event?
     The workspace boundary rules say events-only for external writes, but this looks
     like it needs a synchronous response."]

Error handling and edge cases
  5. [question]

Security and access control
  6. [question]

Performance and scale
  7. [question]
```

Wait for all answers. If developer says "your call" for any question, record it as a
**developer-delegated decision** — document the choice made and the reason.
Do not silently assume.

---

### Phase 3 — Areas of impact analysis

Map every surface this feature touches. Organise by service.

**Determining impact (follow `.claude/context-policy.md`, question-type B):**
For "what does changing X affect" — consumers of a DTO/type, callers of an endpoint,
blast radius of a signature change — use CodeGraph when available:
```bash
codegraph impact <symbol>      # full blast radius, incl. dynamic-dispatch hops
codegraph callers <symbol>     # direct callers
```
This catches re-exports, aliased imports, and dynamic dispatch that `grep -r "Name"`
silently misses — the exact source of "missed a consumer, broke it in prod" failures.
If CodeGraph is unavailable or the service has no index, fall back to grep and note:
`ℹ CodeGraph unavailable for [service] — impact via grep (may miss indirect refs).`
Use the artifacts (architecture.yml, CLAUDE.md) for *intent* (boundary rules, whether a
consumer is allowed), never for the structural fact of who consumes what.

```
# Example output — your actual values will differ
Areas of impact
================

━━━ SERVICE: <service-a> ━━━━━━━━━━━━━━━━━━━━━━━━━━

  Database changes (<db-name>)
    - New tables:      [list or "none"]
    - Modified tables: [table — columns added/changed/removed]
    - New indexes:     [list or "none"]
    - Migrations:      [forward only / reversible]
    - Seed data:       [yes/no]

  API surface changes
    - New endpoints:      [METHOD /path]
    - Modified endpoints: [METHOD /path — what changes]
    - Deprecated:         [METHOD /path — migration path]
    - Breaking changes:   [yes/no ⚠]

  Data and model lifecycle changes (when applicable)
    - Data sources/schemas: [source, schema, version, compatibility]
    - Pipeline stages/features: [what changes and downstream consumers]
    - Training/configuration: [code, parameters, environment, compute needs]
    - Models/artifacts: [format, versioning, registry, storage, rollback]
    - Evaluation: [dataset/version, metrics, baselines, acceptance criteria]
    - Serving: [input/output signature, latency, cost, scaling, fallback]
    - Monitoring: [data/model drift, quality, alerting, ownership]

  Events published (new or changed)
    - [event.name] — payload: [fields] — breaking change: [yes/no]

━━━ SERVICE: <service-b> ━━━━━━━━━━━━━━━━━━━━━━━━━━

  Database changes (<db-name>)
    - [same structure as above]

  Events subscribed (new or changed)
    - [event.name] from [service] — what triggers as a result

━━━ CROSS-DOMAIN ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Shared infrastructure
    - Redis keys added/changed: [key pattern — TTL — purpose]
    - New queues or channels:   [name — type]

  External contract impact (other workspaces)
    - DTOs added or changed:    [DTO name — which services share it]
    - Breaking contract changes: [⚠ list — requires coordinated deploy]

  Boundary + external-contract check
    [Verify against architecture.yml workspace.boundary_rules and external.consumed_by]
    - ✓ [change] does not touch any externally-consumed contract
    - ⚠ BREAKING FOR ANOTHER WORKSPACE: [event/endpoint] is consumed by
      workspace '[name]' ([contact]) — this plan cannot change their code.
      Required: coordinate before merge, or make the change additive/backward-compatible.

━━━ SYSTEM-WIDE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Auth and permissions
    - New roles or permissions:           [list or "none"]
    - Endpoints whose access changes:     [list or "none"]

  Configuration and environment
    - New env vars: [VAR_NAME — service — required/optional]
    - Feature flags: [flag name — default — which services check it]

  External dependencies
    - New third-party APIs/services: [name — component — failure impact]
    - New packages/models/data assets: [name/version — component — reason]

  Existing functionality at risk
    [Name every existing flow that could regress. Be specific.]
    - [flow name] — risk: [why it could break]
    - [flow name] — risk: [why it could break]

━━━ FRONTEND SURFACE CHANGES ━━━━━━━━━━━━━━━━━━━━

  [Only present if frontend services are in scope]
  [One section per frontend service]

  [service-name] ([App Router | Pages Router])
  ─────────────────────────────────────────────
  Pages modified:
    - [route] ([file]) — [what changes]
    - [route] ([file]) — [what changes]

  New pages:
    - [route] → [file to be created]

  New components:
    - [ComponentName] → [src/components/path/ComponentName.tsx]
      Used by: [page or parent component]

  Modified components:
    - [ComponentName] ([file]) — [what changes, props added/removed]
      Used by: [list all pages that import this — check for unintended impact]

  New API hooks:
    - [useFeatureName] → [src/lib/api/feature.ts]
      Calls: [METHOD /api/endpoint] — declared in [backend task ID]

  Router-specific considerations:
    App Router:
      - New pages default to Server Components unless interactivity required
      - Check if any new route needs a loading.tsx or error.tsx
      - If adding to a route group layout, list all routes sharing that layout
    Pages Router:
      - Check _app.tsx and _document.tsx for global providers needed
      - getServerSideProps vs getStaticProps decision needed for new pages
```

If any change violates `workspace.boundary_rules`, or breaks a contract listed under
`external.consumed_by`, raise it immediately:
```
# Example output — your actual values will differ
⚠ Boundary / external-contract violation detected
  architecture.yml workspace.boundary_rules states: "<boundary rule>"
  But the current plan has <service-a> accessing <service-b>'s data directly.
  How should this be resolved?
  Option A: <service-a> fetches via REST from <service-b>
  Option B: <service-b> includes all needed data in the event payload

  Which approach should I plan for?
```

Wait for resolution before continuing.

Ask: "Does this impact analysis look complete? Any areas missed?"
Wait for confirmation before proceeding.

---

### Phase 4 — Implementation plan

Generate the full ordered task list. Each task must be atomic — completable in one sitting,
independently reviewable, and independently testable.

```
Implementation plan
====================
Feature:         [feature name]
Workspace:       [workspace-name]
Services:        [comma-separated list with resolved skills in brackets]
Total tasks:     [N]

━━━ PREREQUISITE TASKS ━━━━━━━━━━━━━━━━━━━━━━━━━
(complete before any feature work begins)

[ ] P1. [Task title]
    Service:    [service-name]
    What:       [specific enough that another developer could do it without asking questions]
    Why first:  [what this unblocks]
    Files:      [folder/filename.ts — one per line]
    Depends on: none

━━━ SERVICE: [service-name] ━━━━━━━━━━━━━━━━━━━━━

[ ] S1-1. [Task title]
    Service:    [service-name]
    Skills:     [resolved skills, e.g. typescript, nodejs, nestjs]
    Type:       [MUST be a task type contributed by one of this service's resolved
                skills — read each skill file's "Task types contributed" section.
                Never invent a type; if no skill covers the needed work, flag it
                as a skill gap for the developer instead of improvising.]

    What:       [specific description]
    Files:      [list files]
    Depends on: [task ID or "none"]
    Risk:       low | medium | high
    Guardrail:  [relevant rule from service CLAUDE.md or workspace.boundary_rules, if any]

[ ] S1-2. [Task title]
    ...

━━━ SERVICE: [service-name] ━━━━━━━━━━━━━━━━━━━━━

[ ] S2-1. [Task title]
    ...

━━━ CROSS-DOMAIN TASKS ━━━━━━━━━━━━━━━━━━━━━━━━━
(tasks that touch more than one service in this workspace)

[ ] C1. [Task title]   ← cross-SERVICE task (within this workspace)
    Services:   [service-a → service-b]
    Services:   [list]
    What:       [description]
    Depends on: [task IDs]
    Risk:       high  ← cross-service tasks default to high until proven otherwise

━━━ DEPLOYMENT TASKS ━━━━━━━━━━━━━━━━━━━━━━━━━━━

[ ] D1. Deployment order
    Order:  [service-a → service-b]
    Why:    [reason — subscriber before publisher, migration before code, etc.]

[ ] D2. Migration execution
    Run migrations for [service] before deploying new code.
    Risk: [data risk if any]

[ ] D3. Environment variables
    Add [VAR_NAME] to [service] in [staging/prod] before deploy.
```

Ordering rules — applied strictly:
- Schema migrations before the service code that uses them
- Event subscriber tasks before event publisher tasks (avoid dropped events at deploy)
- Shared DTO / shared-lib changes before all services that consume them
- Consuming-service tasks after producing-service tasks where there is a dependency
- Never group independent changes into one task
- All cross-service tasks default to `Risk: high`
- Any task touching production data defaults to `Risk: high`

Frontend-specific ordering rules:
- Backend API tasks before the frontend tasks that consume them
  (frontend hooks and pages depend on the API existing)
- `new API hook` before `new page` or `new component` that uses the hook
- `state management` tasks before components that read from that store
- `new component` before `new page` that composes it
- `shared-lib change` (e.g. shared types, utilities) before any service that imports them

**Frontend task generation rules — applies when Layer is `frontend`:**

Generate one task per route change and one task per new or modified component.
Never bundle multiple page changes into one task.

For each entry in the frontend page map from the PRD summary:

`new page` task format:
```
[ ] FE-[N]. Create [route] page
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new page
    Route:      [/path/to/route]
    File:       [app/(group)/path/page.tsx  OR  pages/path.tsx]
    Router:     App Router | Pages Router
    Rendering:  Server Component | Client Component
                [If Client: state the reason — e.g. "uses useState for filter state"]
                [If Server: confirm no hooks, event handlers, or browser APIs]
    What:       Create the [name] page at [route]. Renders [components].
    Components: [list new components this page will compose — link to their tasks]
    API data:   [which API hook or server fetch provides data — link to hook task]
    Files:      [page file, loading.tsx if needed, error.tsx if needed]
    Layouts:    [which layout.tsx this page inherits — list inherited providers/nav]
    Depends on: [API hook task ID, component task IDs]
    Risk:       low | medium | high
```

`modify existing page` task format:
```
[ ] FE-[N]. Add [feature] to [route] page
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       modify component
    Route:      [/path/to/route]
    File:       [exact file path from page map]
    Router:     App Router | Pages Router
    What:       Add [description] to existing [PageName] page.
                Existing imports preserved. Adding: [ComponentName].
    Impact:     [list other routes/layouts that share this file via inheritance]
    Files:      [existing page file]
    Depends on: [new component task ID if applicable]
    Risk:       low | medium | high
```

`new component` task format:
```
[ ] FE-[N]. Create [ComponentName] component
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new component
    File:       [src/components/[feature]/ComponentName.tsx]
    What:       [what the component renders and what interaction it handles]
    Props:      [prop name: type — description, one per line]
    API data:   [data it needs — from hook, prop, or server]
    Used by:    [which page task or component task uses this — link task ID]
    Scaffold from: [closest existing component to copy — from page map, or "none found"]
    Design tokens: [the vocabulary classes this must match — from page map]
    Mockup spec:   [layout/states/copy from mockup, or "no mockup — match app patterns"]
    Files:      [component file, index.ts barrel update if required]
    Depends on: [API hook task ID if it fetches data directly]
    Risk:       low
```

`new API hook` task format — always created before the pages/components that use it:
```
[ ] FE-[N]. Create use[FeatureName] API hook
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new API hook
    File:       [src/lib/api/[feature].ts  OR  src/hooks/use[Feature].ts]
    What:       Fetches [data] from [backend endpoint]. Returns [shape].
    Endpoint:   [METHOD /api/path — from backend task]
    Library:    [React Query / SWR / custom fetch — match discovered pattern]
    Files:      [hook file, types file if new response type needed]
    Depends on: [backend endpoint task ID — must exist before hook is built]
    Risk:       low
```

`new custom hook` task format (non-API utility hooks):
```
[ ] FE-[N]. Create use[HookName] hook
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new custom hook
    File:       [src/hooks/use[HookName].ts]
    What:       [what the hook does — e.g. debounces a value, detects outside clicks]
    SSR safe:   [yes — no browser APIs used / no — guarded with typeof window check]
    Used by:    [which component tasks use this — link task IDs]
    Files:      [hook file]
    Depends on: none
    Risk:       low
```

`new context / provider` task format:
```
[ ] FE-[N]. Create [FeatureName] context and provider
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new context/provider
    Files:      [src/contexts/[Feature]Context.tsx]
    What:       [what state or value the context provides to its subtree]
    Rendering:  Client Component — contexts require 'use client'
    Placed in:  [which layout.tsx or page.tsx wraps the provider]
                [list all routes that will receive this context]
    Consumers:  [which components use this context — link task IDs]
    Depends on: [layout or page task ID where provider is placed]
    Risk:       medium — incorrect placement silently breaks all consumers
```

`new layout` task format (App Router only):
```
[ ] FE-[N]. Create [segment] layout
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new layout
    File:       [app/(group)/path/layout.tsx]
    What:       [what this layout wraps — nav, auth check, provider, shell]
    Rendering:  Server Component | Client Component
    Affects:    [list every route segment this layout will wrap — be exhaustive]
    Providers:  [any context providers added here — affects all child routes]
    Depends on: [context task IDs if adding providers]
    Risk:       high — affects all routes in the segment
```

`new middleware` task format:
```
[ ] FE-[N]. Create / update middleware
    Service:    [frontend-service-name] (frontend)
    Layer:      frontend
    Type:       new middleware
    File:       middleware.ts (root of service)
    What:       [what the middleware does — auth redirect, locale, headers]
    Matcher:    [exact matcher config — which routes it runs on]
    Runtime:    Edge — no Node.js APIs, no npm packages with native bindings
    Risk:       high — runs on every matched request, errors affect all users

---

### Phase 5 — Testing plan

List what must be tested, at what level, and why. Do not write test code.

```
Testing plan
=============

━━━ SERVICE: [service-name] ━━━━━━━━━━━━━━━━━━━━━

  [service-name] — unit tests
    - [what to test] — [edge case or rule it covers]
    - [what to test]

  [service-name] — integration tests
    - [scenario] — verifies [component] behaves correctly when [condition]

━━━ SERVICE: [service-name] ━━━━━━━━━━━━━━━━━━━━━

  [service-name] — unit tests
    - [what to test]

  [service-name] — integration tests
    - [scenario]
  [component-name] — data/model evaluation (when applicable)
    - Dataset and version: [identifier]
    - Checks/metrics and baseline: [agreed evaluation criteria]
    - Data leakage, slice, and reproducibility checks: [applicable scenarios]

━━━ CROSS-DOMAIN ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  End-to-end scenarios
    - [full flow name] — entry point: [endpoint/trigger] → expected outcome: [result]
    [Cover at least one happy path and one failure path per cross-service interaction]

  Contract tests
    [Only if a shared DTO or API contract changed]
    - [service-a] ↔ [service-b] : [what contract to verify]
    - ⚠ New contract test required — did not exist before this feature

  Event flow tests
    - [event.name] published by [service-a] is correctly consumed by [service-b]
    - [event.name] consumer handles missing/malformed payload gracefully

━━━ REGRESSION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[For each item in "Existing functionality at risk" from Phase 3]

  - [existing flow] — verify [specific behaviour] unchanged after [modified file/module]

━━━ EDGE CASES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Derived from Phase 2 answers]

  - [edge case] — expected: [what should happen]
  - [edge case] — expected: [what should happen]

━━━ EXCLUDED FROM TESTING ━━━━━━━━━━━━━━━━━━━━━━
[Be explicit — reduces test noise and review debate]

  - [thing] — reason: [e.g. pure config change, no logic]

━━━ TEST ENVIRONMENT REQUIREMENTS ━━━━━━━━━━━━━━

  Seed data:               [yes — describe / no]
  External mocks required: [SMTP, third-party API names, or "none"]
  Feature flag state:      [on / off / not applicable]
  Data/model assets:       [required dataset, model, artifact, registry, or "none"]
  Services that must run:  [list all services needed for E2E tests]
```

---

### Phase 6 — Risk register

```
Risk register
==============

⚠ HIGH
  [risk title]
  What could go wrong: [description]
  Likelihood:          [why it might happen]
  Mitigation:          [specific action]
  Rollback plan:       [what to do if it goes wrong in production]

⚡ MEDIUM
  [risk title]
  What could go wrong: [description]
  Mitigation:          [action]

ℹ LOW
  [risk title]
  What could go wrong: [description]
  Mitigation:          [action]
```

Mandatory risk categories to evaluate every time:
- Breaking change to a shared API contract
- DB migration on a table with existing production data
- New external dependency with no fallback
- Event payload change affecting a downstream subscriber
- Partial deployment — one service updated, other not yet
- New env var missing from production at deploy time
- Redis key collision with an existing pattern
- Workspace boundary rule violated under time pressure
- Shared DTO changed between services deployed independently

---

### Phase 7 — Developer-delegated decisions log

```
Delegated decisions
====================
[If none: "None — all questions answered explicitly."]

  DD-1. [topic]
        Choice made:  [what was decided]
        Reason:       [why this is the right default]
        Override by:  [what to change in the plan to pick a different option]
```

---

### Phase 8 — Plan summary and checkpoint

```
# Example output — your actual values will differ
════════════════════════════════════════════════════
IMPLEMENTATION PLAN READY FOR REVIEW
════════════════════════════════════════════════════

Feature:          <feature name from PRD>
Workspace:        <workspace-name>
Services:         <list with resolved skills>

Tasks:            [N] total
                  [N] prerequisite
                  [N] per-service
                  [N] cross-service ← all default high risk, review carefully
                  [N] deployment

High-risk tasks:  [N] — listed below for focused review:
                  [task ID] [task title]

Breaking changes: [yes ⚠ — describe / no]
New env vars:     [list or "none"]
Deploy order:     [service-a] → [service-b]

Compliance:
  Workspace boundary rules: [✓ compliant / ⚠ issue — describe]
  External contracts:       [✓ none touched / ⚠ breaking for workspace '[name]']

Plan will be saved to:
  plans/<feature-slug>-plan.md

Open items (resolve before development):
  [list or "none"]

════════════════════════════════════════════════════

Type APPROVE to save the plan and begin development.
Provide feedback to revise — only affected phases will be updated.
Do not begin implementation until APPROVE is received.
```

---

### On APPROVE

1. Derive `feature-slug` as kebab-case from the feature name (e.g. `policy-renewal-reminders`)

2. Create `plans/` at the workspace root if it doesn't exist.

3. Before writing, check for a slug collision:
   ```
   ⚠ plans/<feature-slug>-plan.md already exists (created [date], status [status]).
     A. Overwrite — the existing plan is obsolete
     B. Use a different slug: <feature-slug>-v2
     C. Abort
   ```
   Never silently overwrite an existing plan.

4. Write ONE plan file — `plans/<feature-slug>-plan.md`. There are no duplicate copies:
   a single workspace has a single plans folder, which is also its clear git home.

   The file MUST begin with this exact header block — `/implement`, `/test`,
   `/code-feedback`, and `/raise-pr` all read these fields directly:

   ```markdown
   # Implementation plan — [Feature name]
   > Status: APPROVED
   > Approved: [today's date]
   > Feature: [feature-slug]
   > Branch: feature/[ticket-id-][feature-slug]
   > Workspace: [workspace-name]
   > Services: [comma-separated list with resolved skills in brackets]
   > External contracts touched: [list or "none"]
   ```

   Write the header first, then the full plan content beneath it.

5. Print:
```
# Example output — your actual values will differ
✓ Plan approved and saved

  plans/<feature-slug>-plan.md

Header written:
  Status:    APPROVED
  Branch:    feature/<ticket-id>-<feature-slug>
  Workspace: <workspace-name>
  Services:  <service-a> [nestjs], <service-b> [nextjs]
  External:  ⚠ event policy.renewed consumed by workspace 'notification'

Commit this file before starting development.
Run /implement to begin.
```

Do not begin any implementation. Stop here.
