# Update root CLAUDE.md — system-wide context

Rebuild the root `CLAUDE.md` at the workspace root by reading from three authoritative sources
in strict priority order:
1. `architecture.yml` — service map and connections (highest authority, never infer over this)
2. `architecture.yml` `workspace.boundary_rules` — boundary rules
3. Each `<service>/CLAUDE.md` — implementation specifics

## Arguments
Optional: `$ARGUMENTS` — comma-separated service folder names to scope the scan.
If not provided, scan every service in this workspace.

---

## Instructions

### Step 1 — Verify architecture.yml exists

Check for `architecture.yml` at the workspace root.

- If it exists: read it fully. This is the source of truth for all service relationships,
  events, REST connections, and shared infrastructure. Do not infer or guess anything that
  is declared here — use it exactly as written.

- If it does NOT exist: stop immediately and print:
  ```
  ✗ architecture.yml not found at workspace root.

  This file is required before running /update-context.
  Run /update-architecture first to generate it, or create it manually.

  See architecture.yml format: run /update-architecture --help
  ```
  Do not proceed further.

---

### Step 2 — Discover services in this workspace

If `$ARGUMENTS` is provided, treat each comma-separated value as a service folder name.
If empty, list every service declared in `architecture.yml` (and any service folder
present on disk but undeclared — flag those).

Only this workspace is in scope. Never scan sibling workspaces.

```
# Example output — your actual values will differ
Workspace: <workspace-name>
Services:
  - policy-service        (CLAUDE.md ✓)
  - premium-calculator    (CLAUDE.md missing — run /update-service-context)
  - policy-web            (CLAUDE.md ✓)
```

Ask: "Proceed with these services? (enter to continue, or list changes)"

### Step 3 — Read workspace README files

For each workspace folder that has a `README.md`, read it and extract:
- Workspace purpose (one sentence)
- Boundary rules — what this workspace owns exclusively
- What other workspaces must NOT do with this workspace's data
- Any cross-workspace contracts explicitly declared

If `README.md` is missing for a workspace, flag it:
```
⚠ platform/ has no README.md — workspace boundary rules unknown
```

---

### Step 4 — Read each service's CLAUDE.md

For every service discovered in Step 2, read its `CLAUDE.md` if present.

Extract:
- Service responsibility (one sentence)
- Tech stack (framework, language, databases)
- Key environment variables it needs
- Hard rules and forbidden patterns
- Local dev setup instructions

If a service has no `CLAUDE.md`, read `package.json` only for stack inference. Flag clearly:
```
⚠ premium-calculator has no CLAUDE.md — stack inferred from package.json only
```

Do NOT infer service relationships from code at this step. All relationship data
comes exclusively from `architecture.yml`.

---

### Step 5 — Read existing root CLAUDE.md

If `CLAUDE.md` already exists at the workspace root, read it and identify any sections
that are NOT in the standard template below (e.g. deployment runbooks, on-call notes,
ADR links). You will preserve these verbatim at the end of the regenerated file.

---

### Step 6 — Write the updated root CLAUDE.md

Write to `CLAUDE.md` at the workspace root. Use this exact structure:

```markdown
# [Project name] — System context
> Last updated: [today's date]
> Sources: architecture.yml + [N] workspace folders + [N] service CLAUDE.md files
> Run `/update-context` to regenerate this file

---

## 1. System overview

[2–3 sentences describing what this platform does overall.
Derive from workspace names and service descriptions if clear.
If not clear, write: "TODO: Add system overview — run /update-context after editing architecture.yml"]

---

## 2. Workspace map

[One section per workspace folder. Each section has:]

### workspace-[name]
**Purpose:** [one sentence from workspace README, or TODO if no README]
**Boundary rules:**
[bullet list from workspace README boundary section, or TODO if missing]
**Services:**
[list each service in this workspace with its one-line responsibility from its CLAUDE.md]

---

## 3. Service interaction map

> Source: architecture.yml — do not edit this section manually. Run /update-context to refresh.

### Synchronous calls (REST)
[Generated directly from architecture.yml services[].exposes and the routes in api-gateway.
Format each entry as:]
`caller → callee : METHOD /path — purpose`

[If architecture.yml has no REST entries, write:]
`TODO: Add REST interactions to architecture.yml`

### Asynchronous events
[Generated from architecture.yml services[].publishes and services[].subscribes.
Format each entry as:]
`publisher → event-name (transport) → subscriber`

[If architecture.yml has no event entries, write:]
`TODO: Add event flows to architecture.yml`

### Shared infrastructure
[Generated from architecture.yml shared_infrastructure section.
List databases, caches, queues, data platforms, feature stores, model registries,
and artifact stores with their owners/users. Flag shared dependencies as coupling,
data-governance, or model-versioning risks where relevant.]
`⚠ [resource-name] is shared by [component-a] and [component-b] — coordinate changes`

---

## 4. Tech stack

| Layer | Technology |
|-------|-----------|
| Language | [from service CLAUDE.md files] |
| Framework | [from service CLAUDE.md files] |
| Data/model tooling | [data platforms, pipeline orchestration, ML frameworks, if present] |
| Experiment tracking / model registry | [from service CLAUDE.md files, or TODO if unknown] |
| Primary DB | [from architecture.yml shared_infrastructure] |
| Cache | [from architecture.yml shared_infrastructure] |
| Events / Queue | [from architecture.yml shared_infrastructure] |
| Auth | [from service CLAUDE.md files] |
| Cloud | [from architecture.yml or service CLAUDE.md] |
| CI/CD | [from architecture.yml or infer from .github/.bitbucket] |
| Package/environment manager | [from project manifests or service CLAUDE.md] |

---

## 5. Cross-service coding conventions

[Merge conventions sections from all service CLAUDE.md files.
Include only rules that appear in 2+ services, or rules explicitly marked as system-wide.
Service-specific rules stay in their own CLAUDE.md — do not copy them here.]

### Naming
- [merged naming rules]

### Module structure
- [merged module structure rules]

### Error handling
- [merged error handling rules]

### Testing requirements
- [merged testing rules]

### Forbidden patterns (system-wide)
- [merged forbidden patterns — these apply to every service]

---

## 6. Shared environment variables

[List only variables used by more than one service, sourced from service CLAUDE.md files.
Do NOT list service-specific vars here.]

| Variable | Used by | Purpose |
|----------|---------|---------|

---

## 7. Developer workflow

### Starting a new feature
1. `cd ~/workspace && claude`
2. `/prd <google-doc-url>` — Claude reads PRD, identifies affected services, asks questions
3. Answer clarifying questions in terminal
4. `/plan` — Claude generates implementation plan across affected services
5. Review and approve plan (Checkpoint B)
6. Claude implements, commits to feature branch, raises Bitbucket PR
7. Review PR (Checkpoint C)
8. Claude runs E2E tests, posts results to Slack
9. Approve merge (Checkpoint D)

### Updating system context
- New service added: run `/update-architecture` → then `/update-context`
- Service relationships changed: edit `architecture.yml` directly → run `/update-context`
- Service implementation details changed: update `<service>/CLAUDE.md` → run `/update-context`

### PR conventions
- Branches: `feature/<ticket-id>-short-description`
- All PRs require mandatory review on Bitbucket
- Claude raises PRs via `bb` CLI after implementation

### Local development
[TODO: fill in per-workspace local dev steps]

---

[PRESERVED FROM PREVIOUS CLAUDE.md — DO NOT AUTO-REGENERATE]
[Append any non-standard sections from previous root CLAUDE.md verbatim here]
```

---

### Step 7 — Print completion report

```
✓ Root CLAUDE.md written

Sources used:
  ✓ architecture.yml          ([N] services, [N] events, [N] REST routes)
  ✓ workspace-policy/README.md
  ✓ workspace-notification/README.md
  ⚠ platform/ — no README.md, workspace boundaries not documented

Service CLAUDE.md files read:
  ✓ policy-service
  ✓ notification-service
  ⚠ premium-calculator — inferred from package.json only

Sections needing manual review:
  ⚠ Section 1 (System overview)     — TODO placeholder written
  ⚠ Section 7 (Local dev setup)     — TODO placeholder written

Next steps:
  1. Fill in TODO sections in ./CLAUDE.md
  2. Add missing workspace README.md files (run /update-architecture for guidance)
  3. Commit CLAUDE.md and architecture.yml to version control
```
