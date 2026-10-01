# Read and analyse a PRD

Read a product requirements document, load all system context, identify affected services
in this workspace, ask targeted clarifying questions, then produce a structured PRD summary that
/plan can consume directly.

## Arguments
`$ARGUMENTS` — one of the following input types, or a combination:

**Single source:**
- A Google Docs URL: `https://docs.google.com/document/d/...`
- A Jira issue URL: `https://yourorg.atlassian.net/browse/PROJ-123`
- A Jira issue ID: `PROJ-123` or `INS-456` (detected by pattern `[A-Z]+-[0-9]+`)
- A local file path: `./docs/feature-name.md` — supports `.md`, `.txt`, `.pdf`, `.docx`
- A mockup image: `./mockups/renewal-page.png` — supports `.png`, `.jpg`, `.jpeg`, `.webp`
- Raw text pasted inline (no URL, no path — treat entire argument as PRD content)

**Multiple sources (combine with space or comma):**
- `https://jira.../PROJ-123 ./docs/wireframes.md` — Jira ticket + local spec file
- `https://docs.google.com/... PROJ-123` — Google Doc + Jira ticket
- `./requirements.md ./designs/flow.md` — multiple local files merged

When multiple sources are provided, merge all content before extraction.
Label each source in the PRD summary so the origin of each requirement is traceable.

If `$ARGUMENTS` is empty, stop and print:
```
✗ No PRD provided.

Usage:
  /prd https://docs.google.com/document/d/<id>/edit
  /prd https://yourorg.atlassian.net/browse/PROJ-123
  /prd PROJ-123
  /prd ./docs/my-feature.md
  /prd PROJ-123 ./docs/additional-notes.md
  /prd (then paste PRD text directly)

Accepted sources: Google Doc URL, Jira URL or issue ID,
local file (.md .txt .pdf .docx), or plain text.
Multiple sources can be combined.
```

---

## Instructions

Work through every phase below in order. Do not skip phases.
Do not identify affected services or ask questions until you have loaded all context in Phase 1.

---

### Phase 1 — Load system context

Load all context files before reading the PRD. This prevents wrong service identification.

#### Step 1a — Verify workspace structure

Check that the following exist at the workspace root:
- `architecture.yml`
- `CLAUDE.md`
- At least one service folder in this workspace

If `architecture.yml` is missing, stop and print:
```
✗ architecture.yml not found at workspace root.

The workflow requires system context before reading any PRD.
Run /update-architecture first, then re-run /prd.
```

If `CLAUDE.md` is missing but `architecture.yml` exists, print a warning and continue:
```
⚠ Root CLAUDE.md not found — consider running /update-context to generate it.
Continuing with architecture.yml only.
```

#### Step 1b — Read architecture.yml

Read the full file. Extract and hold in memory:
- Every service name, its `description`, `type`, and declared `skills` (if any)
- `workspace.boundary_rules` and the `external:` contracts block
- Every `exposes` (REST endpoints)
- Every `publishes` and `subscribes` (events)
- Every `calls` (synchronous service-to-service)
- `shared_infrastructure` (databases, caches, queues)
- `api_gateway` routes
- Per-service `guardrails`

#### Step 1c — Read workspace boundary rules and external contracts

From `architecture.yml`, read `workspace.boundary_rules` and `external:`.
Record:
- Purpose
- Boundary rules
- External contracts: what other workspaces consume from us, and whom to notify

Flag missing READMEs but continue:
```
# Example output — your actual values will differ
⚠ workspace.boundary_rules is empty in architecture.yml — boundary rules unknown
```

#### Step 1d — Read root CLAUDE.md

Read `CLAUDE.md` at the workspace root for cross-service conventions, forbidden patterns,
and developer workflow context.

#### Step 1e — Confirm context loaded

Print a one-line confirmation before reading the PRD:
```
Context loaded: [N] services in workspace '<workspace-name>'. Reading PRD...
```

---

### Phase 2 — Read the PRD

#### Step 2a — Detect input type and fetch content

First, identify every source in `$ARGUMENTS`. Parse each token:

**Jira URL** (`https://*.atlassian.net/browse/PROJ-123`):
Use the Atlassian MCP tool to fetch the issue. Extract:
- Summary (issue title)
- Description (full body)
- Acceptance criteria (if in a dedicated field or section)
- Labels, components, priority, fix version
- Any linked issues or sub-tasks (list them but do not fetch — flag as related context)

If Jira fetch fails:
```
✗ Could not fetch Jira issue: [url or ID]

Possible causes:
  - Atlassian MCP is not connected (check MCP config)
  - Issue does not exist or you lack permission
  - URL is malformed

Alternatively, export the issue as text and run:
  /prd ./path/to/exported-issue.txt
```

**Jira issue ID** (pattern `[A-Z]{2,10}-[0-9]+`, e.g. `PROJ-123`, `INS-456`):
Treat as a Jira issue ID. Use the Atlassian MCP tool with the issue key directly.
Same fetch logic and failure handling as Jira URL above.
Also: store this as `SESSION.ticket_id` immediately — skip Phase 4 ticket ID capture
for this value (it was provided explicitly as the input).

**Google Docs URL** (`https://docs.google.com/document/d/...`):
Use the Google Drive MCP tool to fetch the document content.
If the fetch fails:
```
✗ Could not read Google Doc at: [url]

Possible causes:
  - Document is not shared (set to "Anyone with the link can view")
  - Google Drive MCP is not connected (check MCP config)
  - URL is malformed

Alternatively, export the doc as markdown and run:
  /prd ./path/to/exported-file.md
```

**Local file path** (starts with `./`, `/`, or `../`, or ends with `.md` `.txt` `.pdf` `.docx`):
Read the file from disk.
- `.md` / `.txt`: read directly as text
- `.pdf`: extract text content using available PDF reading tools
- `.docx`: extract text content using available document reading tools

If file not found:
```
✗ File not found: [path]
Check the path is relative to the workspace root or provide an absolute path.
Supported formats: .md .txt .pdf .docx
```

**Raw text** (anything that doesn't match the above patterns):
Use the text directly as PRD content.

---

**Multiple sources — merge content:**

If more than one source was provided, fetch all sources in parallel then merge:

```
Reading PRD sources:
  ✓ Jira PROJ-123       — fetched (1,200 words)
  ✓ ./docs/designs.md   — read (800 words)
  ✓ Google Doc          — fetched (2,400 words)
```

Merge strategy:
- Deduplicate: if the same requirement appears in multiple sources, keep the most
  detailed version and note the duplication
- Conflict: if two sources contradict each other on the same requirement, flag it
  explicitly rather than picking one silently:
  ```
  ⚠ Conflicting requirement detected
    Jira PROJ-123:   "Renewal reminders send 30 days before expiry"
    Google Doc:      "Renewal reminders send 7 days before expiry"
  
  Which is correct? (or should both be supported?)
  ```
  Wait for clarification before proceeding
- Additive: content unique to each source is included in the merged PRD
- Label: tag each requirement with its source in the PRD summary
  e.g. `[Jira]`, `[Google Doc]`, `[designs.md]` for traceability

Print a source summary before proceeding to Step 2b:
```
# Example output — your actual values will differ
Sources merged:
  [Jira]       PROJ-123 — <issue title>
  [File]       ./docs/designs.md — <first heading or filename>
  [Google Doc] <document title>

Total content: ~[N] words
Conflicts resolved: [N] / Conflicts pending: [N]
Proceeding to requirement extraction...
```

#### Step 2a-img — Extract and analyse embedded mockup images

UI mockups carry more design intent than any text description. PRDs in Google Docs
and Jira routinely embed screenshots — these must reach the model as images, not be
dropped as text extraction silently does.

**Detect images in each source:**

- **Google Docs:** the document may contain embedded images. Use the Drive MCP to
  retrieve them. If the MCP returns only text (images stripped), note this explicitly:
  ```
  ⚠ This Google Doc appears to contain [N] embedded images, but the Drive MCP
    returned text only. Image content was not captured.

    To include the mockups, either:
    A. Export the images and re-run: /prd PROJ-123 ./mockups/screen1.png ./mockups/screen2.png
    B. Continue without visual specs (UI will be built from text description only —
       higher risk of mismatch with the intended design)
  ```
  Wait for the developer's choice.

- **Jira:** check the issue for attachments and embedded images via the Atlassian MCP.
  Same fallback if images can't be retrieved.

- **Local image files** passed as arguments (`.png`, `.jpg`, `.jpeg`, `.webp`): read
  them directly as images.

**Analyse each image as an image (not OCR):**

For every mockup image obtained, view it and produce a structured visual spec:

```
Visual spec — [image source / filename]
=========================================
Screen/page: [what page or screen this depicts]
Layout:      [overall structure — header, sidebar, main content, columns, grid]
Components visible:
  - [component] — [position, e.g. "top-right of card"]
  - [component] — [description]
Key UI elements:
  - [buttons, their labels, their apparent prominence (primary/secondary)]
  - [form fields, their types, their order, their grouping]
  - [tables/lists, their columns, their density]
  - [status indicators, badges, tags and their states]
States shown:
  - [empty / populated / loading / error — whichever the mockup depicts]
Visual hierarchy:
  - [what draws the eye first, what is emphasised, what is secondary]
Interaction cues:
  - [anything implying behaviour — dropdowns, toggles, expandable rows, tabs]
Text/copy visible:
  - [exact labels, headings, helper text shown in the mockup]
```

Map each visual spec to the requirement(s) and the page it relates to. If a mockup
shows a screen that doesn't correspond to any extracted requirement, flag it — the
mockup may imply scope the text missed:
```
⚠ Mockup "[filename]" shows a [screen description] but no requirement references it.
  Is this in scope? It may be implied UI the text PRD didn't spell out.
```

These visual specs flow into the Frontend page map (Step 3d) and become per-component
design constraints in the PRD summary. They are the reference `/implement` matches
against when building UI.

If no images were found in any source:
```
No mockup images found in the PRD sources.
UI will be built to match existing app patterns only (see Step 3d).
If you have mockups, re-run with: /prd [sources] ./path/to/mockup.png
```

#### Step 2b — Extract PRD structure

Parse the document and extract the following fields. If a field is absent from the PRD,
mark it explicitly as `NOT PROVIDED` — do not infer or fabricate it.

- **Title**: feature or epic name
- **Ticket ID**: Jira/linear/issue number (look for patterns like `INS-123`, `PROJ-456`,
  `#123`, or any identifier in the doc header or title)
- **Author**: who wrote the PRD
- **Status**: draft / ready for development / approved (look in doc header/metadata)
- **Objective**: the business goal this feature achieves
- **Background**: context or problem statement
- **User stories / use cases**: who does what and why
- **Functional requirements**: numbered list of what the system must do
- **Non-functional requirements**: performance, security, scale expectations
- **Out of scope**: explicitly excluded items
- **Open questions**: questions already noted in the PRD itself
- **Acceptance criteria**: how to verify the feature is done
- **Dependencies**: other features, services, or external systems this relies on
- **Deadline or milestone**: if mentioned

For AI/ML or data-related requirements, also extract when present; otherwise mark
`NOT APPLICABLE` or `NOT PROVIDED` rather than filling gaps by assumption:
- **Data**: source, owner, access constraints, schema/version, quality expectations,
  retention, and lineage requirements
- **Model/training**: model or task, training configuration, compute constraints,
  artifact/versioning requirements, and compatibility expectations
- **Evaluation**: approved dataset/version, metrics and their definitions, baseline,
  acceptance criteria, slice analysis, and known limitations
- **Inference/operations**: input/output contract, latency/cost/reliability targets,
  rollout/rollback, monitoring, and human review or fallback behavior

Additionally, scan every user story, requirement, and acceptance criterion for
implementation signals. Look for data sources and transformations, training and
evaluation workflows, model or artifact changes, inference/serving behavior, APIs,
events, and UI surfaces when applicable. Do not treat model-quality objectives as
verified unless the PRD specifies an evaluation method and acceptance criteria.

For UI-capable work, also scan for language that implies a specific page or interaction:

UI signal words to look for:
- Page/screen references: "on the dashboard", "renewal page", "policy detail screen",
  "settings page", "a new page", "navigate to"
- Action verbs on UI: "user clicks", "user sees", "user can view", "display", "show",
  "form", "button", "modal", "table", "list", "filter", "search"
- Navigation signals: "redirect to", "go back to", "breadcrumb", "tab", "step"
- New UI: "new screen", "new section", "new widget", "new form"

For each UI signal found, record:
```
UI surfaces detected from PRD
===============================
  REQ-3: "User can view renewal schedule on the policy detail page"
    → Signal: existing page (policy detail) getting new section
    → Page: policy detail
    → Change type: new section on existing page

  REQ-5: "User receives a confirmation modal after submitting renewal"
    → Signal: new modal component
    → Page: unknown — triggered from renewal form
    → Change type: new component

  AC-2: "Renewal reminder banner appears on the dashboard"
    → Signal: existing page (dashboard) getting new component
    → Page: dashboard
    → Change type: new component on existing page
```

If no UI signals are found, note it explicitly:
```
UI surfaces detected: NONE — this appears to be a backend-only feature
```

---

### Phase 3 — Identify affected services

Using only the content from Phase 2 and the system context from Phase 1:

#### Step 3a — Match requirements to services

For each functional requirement, identify which service(s) own the relevant data or
behaviour based on `architecture.yml` service descriptions and workspace boundary rules.

Do not guess. If a requirement cannot be mapped to a declared service, flag it:
```
⚠ Requirement [N] — "[requirement text]"
  No service in architecture.yml owns this. Options:
  A. This belongs to [service] — confirm?
  B. This requires a new service not yet declared
  C. This is a frontend concern outside this workflow's scope
```

#### Step 3b — Identify primary and secondary services and classify type

- **Primary service**: owns the core data entity being created or modified
- **Secondary services**: consume events, provide data, or need coordinated changes
- **Touched but unchanged**: services whose contracts are read but not modified (e.g. auth)

For every identified service, resolve its skills via `.claude/skills/skill-registry.yml`
(declared `skills:` in architecture.yml wins; otherwise detect via registry signals;
load `Requires:` transitively). Record the resolved skill list per service.

A service is **UI-capable** if any resolved skill is in the registry's `ui_capable`
list (e.g. reactjs, nextjs, html). UI-capable services trigger the frontend page-map
flow (Step 3d) and receive UI task types in `/plan` — flag them explicitly.

If a PRD requirement references a user-facing UI change but no UI-capable service
is in the affected list, flag it:
```
⚠ Requirement [N] appears to require a UI change but no UI-capable service
  is listed as affected. Is there a frontend app in scope?
  Options:
  A. [name the UI-capable service from architecture.yml that should be added]
  B. This is a backend-only change — UI is out of scope for this feature
```

#### Step 3d — Scan frontend service codebase to ground UI surfaces

Run this step only if at least one frontend service was identified in Step 3b.

For each frontend service in scope:

**Step 3d-i — Detect router type**

Check the service folder:
- If `app/` directory exists with `page.tsx` or `layout.tsx` files → App Router
- If `pages/` directory exists with `.tsx` files (excluding `pages/api/`) → Pages Router
- If both exist → Mixed (note which routes are in each)

Record:
```
# Example output — your actual values will differ
<service-name> router: App Router (app/ directory)
```

**Step 3d-ii — Build the route map**

App Router: scan `app/` recursively. For each `page.tsx` file, derive the route:
- `app/page.tsx` → `/`
- `app/dashboard/page.tsx` → `/dashboard`
- `app/(dashboard)/policies/page.tsx` → `/policies` (route group, strip parens)
- `app/(dashboard)/policies/[id]/page.tsx` → `/policies/[id]`
- `app/(dashboard)/policies/[id]/renewal/page.tsx` → `/policies/[id]/renewal`

Pages Router: scan `pages/` recursively, skipping `pages/api/`. For each `.tsx` file:
- `pages/index.tsx` → `/`
- `pages/dashboard.tsx` → `/dashboard`
- `pages/policies/[id].tsx` → `/policies/[id]`

Print the route map for each frontend service:
```
# Example output — your actual values will differ
<service-name> route map (App Router)
===================================
  /                          app/page.tsx
  /dashboard                 app/(dashboard)/page.tsx
  /[resource]                app/(dashboard)/[resource]/page.tsx
  /[resource]/[id]           app/(dashboard)/[resource]/[id]/page.tsx
  /settings                  app/(dashboard)/settings/page.tsx
```

**Step 3d-iii — Map UI surfaces to existing routes**

Take each UI surface detected in Phase 2b and match it to the route map:

For each UI surface:
1. Look for an exact route match by page name or keyword
2. If found → record the existing file path
3. If not found → this is likely a new page

When CodeGraph is available (per `.claude/context-policy.md`), use
`codegraph explore "<surface description>"` to locate where related logic already lives
even when route names don't obviously match — its FTS + graph finds the handling code
that a keyword match on the route map would miss. Fall back to keyword matching on the
route map when CodeGraph is unavailable.

```
UI surface → file mapping
==========================
"policy detail page" (REQ-3)
  → matched: app/(dashboard)/policies/[id]/page.tsx
  → change type: modify existing page — add renewal schedule section

"renewal confirmation modal" (REQ-5)
  → no direct route match — modal component, not a page
  → likely triggered from: app/(dashboard)/policies/[id]/page.tsx or renewal page
  → change type: new component — location TBD, ask developer

"dashboard renewal banner" (AC-2)
  → matched: app/(dashboard)/page.tsx
  → change type: modify existing page — add banner component
```

If a UI surface cannot be matched, flag it as a clarifying question for Phase 5:
```
⚠ Could not locate page for: "renewal reminder banner on dashboard"
  Route map shows: app/(dashboard)/page.tsx exists
  Is this the correct page? Adding to clarifying questions.
```

**Step 3d-iv — Check for existing related components**

For each matched page, scan for existing components imported into that page.
Look for components whose names suggest relevance to the feature
(e.g. for a renewal feature: `Renewal*`, `Policy*`, `reminder`, `schedule`).

```
Existing components in scope
==============================
app/(dashboard)/policies/[id]/page.tsx imports:
  - PolicyHeader          src/components/policy/PolicyHeader.tsx
  - PolicyDetailsCard     src/components/policy/PolicyDetailsCard.tsx
  - PolicyDocuments       src/components/policy/PolicyDocuments.tsx
  ← no renewal-related components found — all will be new

app/(dashboard)/page.tsx imports:
  - DashboardSummary      src/components/dashboard/DashboardSummary.tsx
  - RecentPolicies        src/components/dashboard/RecentPolicies.tsx
  ← no banner component found — RenewalReminderBanner will be new
```

**Step 3d-v — Extract design vocabulary from the closest existing components**

This step matters most when the component library is loosely enforced and patterns
vary by page. Since there is no single design-system source of truth, the reference
for any new component must be the *closest existing component of the same kind*.

For each new component the feature will require, identify the closest existing analogue
by kind:
- New card → find an existing card component
- New form → find an existing form
- New list/table → find an existing list/table
- New modal → find an existing modal
- New banner/alert → find an existing banner/alert

Read the source of each closest analogue and extract its concrete visual vocabulary —
the actual class strings, not a generalisation:

```
Design vocabulary — extracted from closest existing components
================================================================
Closest card: PolicyDetailsCard.tsx
  Container:   rounded-lg border bg-card p-6 shadow-sm
  Heading:     text-lg font-semibold mb-4
  Body text:   text-sm text-muted-foreground
  Spacing:     space-y-4 between rows

Closest form: PolicyEditForm.tsx
  Field layout:   label above input, error below
  Label:          text-sm font-medium mb-1.5
  Input:          uses <Input> from src/components/ui/input
  Error text:     text-sm text-destructive mt-1
  Submit button:  <Button> from ui/button, full width on mobile
  Field spacing:  space-y-4

Closest list: RecentPolicies.tsx
  Row:         flex items-center justify-between py-3 border-b last:border-0
  Density:     comfortable (py-3), not compact

Closest banner: (none found — no existing banner component)
  ⚠ No analogue exists. New banner has no internal reference to match.
  Recommend: developer provides a reference or approves a proposed design.
```

**The constraint for `/implement`:** any new component must reuse these exact tokens.
When a close analogue exists, the strongest approach is to copy that component as a
scaffold and adapt it — not generate from scratch. Where a mockup visual spec exists
(from Step 2a-img), the component must satisfy both: the existing design vocabulary
AND the layout/hierarchy shown in the mockup. If these conflict (mockup shows a style
the app doesn't use), flag it:
```
⚠ Design tension for [component]:
  Mockup shows: [e.g. pill-shaped buttons, rounded-full]
  App convention: [e.g. rounded-md buttons everywhere else]
  Which wins? (match the app / match the mockup / ask designer)
```

This vocabulary and any per-component mockup spec are written into the Frontend page
map so `/plan` and `/implement` carry them as constraints.

From the services identified:
- If a requirement needs a service that is NOT in this workspace, stop and flag it:
  ```
  ✗ Requirement [N] needs '[service]', which is not part of this workspace.
    Features cannot span workspaces. Options:
    A. Narrow this feature to what this workspace owns; raise a separate feature
       in the other workspace for their side
    B. This service actually belongs here — run /update-architecture to register it
  ```
- Check `external.consumed_by`: if the feature changes an event or endpoint another
  workspace depends on, flag it as a coordination item (it cannot be planned here)

Print the service map:
```
# Example output — your actual values will differ
Service map
============
Workspace: <workspace-name>
Primary service:   <service-a>  [typescript, nodejs, nestjs]
Secondary:         <service-b>  [typescript, nodejs, nestjs] — event consumer
Secondary:         <service-c>  [typescript, reactjs, nextjs] — UI-capable
Touched only:      <service-d>  — routing unchanged

External contracts touched: YES | NO
  ⚠ event policy.renewed — consumed by workspace 'notification' (#team-notification)
     This feature cannot change their code. Coordinate before merge.

Frontend services in scope: YES — <service-c>
  /plan will generate frontend task types for this service
```

---

### Phase 4 — Ticket ID capture

#### Step 4a — Check if ticket ID was found

**If a Jira URL or Jira issue ID was provided as input in Phase 2:**
The ticket ID is already set as `SESSION.ticket_id`. Confirm it:
```
Ticket ID: <ticket-id> (from Jira input)
Branch will be: feature/<ticket-id>-<feature-slug>
Correct? (enter to confirm or provide a different ID)
```

**If ticket ID was found in document content (Phase 2b):**
Confirm it:
```
# Example output — your actual values will differ
Ticket ID found: <ticket-id>
Branch will be: feature/<ticket-id>-<feature-slug>
Is this correct? (press enter to confirm or provide the correct ID)
```

**If no ticket ID found from any source:**
```
No ticket ID found.
Provide a Jira/issue ID to include in the branch name (e.g. PROJ-123),
or press enter to skip — branch will be: feature/<feature-slug>
```

Wait for input. If the developer provides an ID, store it as `SESSION.ticket_id`.
If they press enter or type nothing, set `SESSION.ticket_id` to `null` and continue.
Ticket ID is optional — never block progress waiting for one.

#### Step 4b — Derive feature slug

From the PRD title, derive a kebab-case slug of 3–5 words.
Example: "Policy Renewal Reminder Notifications" → `policy-renewal-reminders`

Derive branch name based on whether a ticket ID was provided:
- With ticket ID:    `feature/<ticket-id>-<feature-slug>`
- Without ticket ID: `feature/<feature-slug>`

Confirm:
```
# Example output — your actual values will differ
Feature slug:  <feature-slug>
Branch name:   feature/<ticket-id>-<feature-slug>   (or without ID if skipped)
Correct? (enter to confirm or type a different slug)
```

Wait for confirmation. Store confirmed slug as `SESSION.feature_slug`.

---

### Phase 5 — Clarifying questions

Review the PRD content, the extracted requirements, and the service map.
Identify every gap that would block `/plan` from producing a complete implementation plan.

Ask ALL questions in a single message — never drip-feed one at a time.
Organise by category. Only include categories with actual questions.
Do not ask questions answerable from `architecture.yml` (including boundary rules).

```
Clarifying questions
=====================
[N] questions before /plan can proceed. Answer by number.

Business logic
  1. [specific question tied to a requirement number]
  2. [specific question]

Data ownership
  3. [question about which service owns a new entity or field]

External contract impact
  4. [question about a change to an externally-consumed event/endpoint]
     Context: workspace.boundary_rules says [rule] — does this feature comply,
     or does the other workspace need to be coordinated with first?

Error handling
  5. [question about failure modes — e.g. what happens if the event consumer is down?]

Security and access control
  6. [question about who can trigger this feature, what auth is required]

Non-functional
  7. [question about scale, latency, or reliability expectations if not in PRD]

If the answer to any question is "your call", I will document it as a
developer-delegated decision in the plan. You don't need to answer every question —
but unanswered questions become explicit TODOs that block plan approval.
```

Wait for all answers before proceeding to Phase 6.

---

### Phase 6 — Produce the PRD session summary

After receiving clarification answers, write the full session summary.
This summary is what `/plan` reads as its input — it must be self-contained and unambiguous.

Write this summary to a file at:
`plans/prd-summary-<feature-slug>.md`

Create the `plans/` folder if it doesn't exist.

Use this exact structure:

```markdown
# PRD Summary — [Feature title]
> Generated by: /prd command
> Date: [today's date]
> Sources:
>   [Jira]       PROJ-123 — https://yourorg.atlassian.net/browse/PROJ-123
>   [Google Doc] https://docs.google.com/document/d/...
>   [File]       ./docs/designs.md
> Status: ready for /plan

---

## Identifiers
- Ticket ID:     [e.g. INS-123 or "none"]
- Feature slug:  [e.g. policy-renewal-reminders]
- Branch name:   feature/[ticket-id-][feature-slug]
- PRD author:    [name or NOT PROVIDED]
- PRD status:    [draft / approved / NOT PROVIDED]

---

## Objective
[One paragraph. Exact objective from PRD or synthesised from user stories if not explicit.]

## Background
[Context from PRD. What problem does this solve? Why now?]

## Functional requirements
[Numbered list from all sources. Rewritten to be unambiguous where needed.
Each requirement mapped to a service in brackets and labelled with its source.]

  1. [Requirement text]  → [service-name]  [Jira]
  2. [Requirement text]  → [service-name]  [Google Doc]
  3. [Requirement text]  → NEW SERVICE NEEDED ⚠  [File: designs.md]

## Non-functional requirements
[Performance, security, scale. "NOT PROVIDED" if absent from PRD.]

## Out of scope
[Explicitly excluded items from PRD. "NOT PROVIDED" if absent.]

## Acceptance criteria
[How to verify done. Exact from PRD or "NOT PROVIDED".]

---

## Service map
Workspace: [workspace-name]
Primary service:   [service] [resolved skills]
Secondary:         [service] [resolved skills] — [role]
Touched only:      [service] — [what is read but not changed]

Frontend services in scope: [YES — list / NO]
External contracts touched: [YES — list + owning workspace + contact / NO]
  [If YES: interaction pattern and boundary compliance status]

---

## Frontend page map
[Only present if frontend services are in scope. One section per frontend service.]

### [service-name] ([App Router | Pages Router])

| Route | File | Change type | Source requirement |
|-------|------|-------------|-------------------|
| /policies/[id] | app/(dashboard)/policies/[id]/page.tsx | modify — add renewal section | REQ-3 |
| /dashboard | app/(dashboard)/page.tsx | modify — add banner component | AC-2 |
| /policies/[id]/renewal | — | new page | REQ-7 |

New components required:
  - RenewalScheduleSection  → in or alongside policies/[id]/page.tsx
  - RenewalConfirmModal     → src/components/policy/ (new)
  - RenewalReminderBanner   → src/components/dashboard/ (new)

Existing components to modify:
  - PolicyDetailsCard       → src/components/policy/PolicyDetailsCard.tsx (add renewal status field)

### Design vocabulary (match these exact tokens)
[From Step 3d-v — the concrete classes extracted from the closest existing components.
/implement must reuse these so new UI matches the existing app.]

  Cards:    [exact container classes from closest existing card]
  Forms:    [field layout, label/input/error pattern from closest existing form]
  Lists:    [row structure, density from closest existing list]
  Buttons:  [size, radius, variant classes in use]
  Closest-analogue components to copy as scaffolds:
    - [new component] → copy from [existing component path]

### Mockup visual specs
[From Step 2a-img — one block per mockup image that maps to a page/component.
/implement matches the built UI against these. If no mockups: "No mockups provided."]

  [component or page name] — from mockup [filename/source]:
    Layout:    [structure from the visual spec]
    Elements:  [key elements, their order, prominence]
    States:    [states the mockup shows]
    Copy:      [exact labels/headings visible in the mockup]

  Design tensions flagged: [any mockup-vs-app-convention conflicts, or "none"]

[If no frontend services: "Not applicable — no frontend services in scope"]

---

## Domain context
### [service-name]
Boundary rules relevant to this feature:
  - [rule]
  - [rule]

### [service-name] (if secondary)
Boundary rules relevant to this feature:
  - [rule]

---

## Clarification answers
[Every question from Phase 5 with the developer's answer, or "DELEGATED" if skipped.]

  Q1. [question]
  A1. [answer or DELEGATED — [what was decided]]

  Q2. [question]
  A2. [answer]

---

## Open questions (unresolved)
[Any questions the developer explicitly deferred. These become TODOs in /plan.]
  - [question] — impact: [what part of the plan is blocked until resolved]

---

## Dependencies
[Other features, external systems, or services this relies on. "None" if absent.]

---

## Deadline / milestone
[From PRD, or "NOT PROVIDED".]
```

---

### Phase 7 — Handoff

After writing the summary file, print:

```
# Example output — your actual values will differ
════════════════════════════════════════════
PRD ANALYSIS COMPLETE
════════════════════════════════════════════

Feature:      <feature title from PRD>
Ticket:       <ticket-id or "none">
Branch:       feature/<ticket-id>-<feature-slug>

Services:     [N] affected
  Primary:    <service-a>
  Secondary:  <service-b>

Requirements: [N] functional, [N] non-functional
Open items:   [N] unresolved questions → will become TODOs in plan

PRD summary saved to:
  plans/prd-summary-<feature-slug>.md

════════════════════════════════════════════

Run /plan to generate the implementation plan.
/plan will read the summary above — no need to re-answer questions.
```

Do not run `/plan` automatically. Stop here and wait for the developer to proceed.

---

## Session state contract for /plan

The following values must be established in this session for /plan to work correctly.
If /plan is run in a new session, it will read the saved PRD summary file instead.

| Key | Value | Set in |
|-----|-------|--------|
| SESSION.ticket_id | e.g. INS-123 or `null` if skipped | Phase 4 |
| SESSION.feature_slug | e.g. policy-renewal-reminders | Phase 4 |
| SESSION.branch_name | feature/INS-123-[slug] or feature/[slug] if no ticket | Phase 4 |
| SESSION.primary_service | e.g. policy-service | Phase 3 |
| SESSION.workspace | e.g. policy | Phase 1 |
| SESSION.secondary_services | e.g. [notification-service] | Phase 3 |
| SESSION.frontend_services | e.g. [web-portal] or [] if none | Phase 3 |
| SESSION.prd_summary_path | e.g. plans/prd-summary-*.md | Phase 6 |
