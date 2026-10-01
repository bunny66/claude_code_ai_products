# Update architecture.yml — service relationship map

Create or update `architecture.yml` at the workspace root. This file is the single source
of truth for how services are connected. It is read by `/update-context` and `/prd` — keeping
it accurate is what prevents Claude from making wrong assumptions about your system.

## Arguments
- No arguments: interactive mode — scan the workspace and ask questions to build the file
- `--help`: print the architecture.yml format reference and exit
- `--validate`: read the existing architecture.yml and report any inconsistencies, do not write

---

## Instructions

### If `$ARGUMENTS` is `--help`

Print this and stop:

```
architecture.yml format reference
==================================
# Scope: THIS workspace only. A workspace holds the repos for one bounded area,
# and a feature never spans workspaces. Services outside this workspace are not
# listed under `services:` — declare only the contracts they depend on under
# `external:` (see bottom) so breaking changes are still caught.

workspace:
  name: <workspace-name>              # e.g. demand-forecasting
  description: one sentence — what this workspace owns as a whole
  boundary_rules:                     # what other workspaces may/may not do
    - "Training datasets are versioned and immutable once an experiment starts"
    - "Only approved workloads may access production datasets or registered models"

services:
  <service-folder-name>:
    description: one sentence — what this service owns
    type: api | gateway | worker | scheduler | data-pipeline | model-training | model-serving | evaluation | frontend | library | other
    skills: [python, pytorch, mlflow]      # OPTIONAL — explicit skill list from
                                           # .claude/skills/skill-registry.yml. Declared skills
                                           # override detection in all commands.
                                           # Omit to let the registry auto-detect.
    stack:
      language: Python | TypeScript | R | Julia | ...
      framework: PyTorch | TensorFlow | scikit-learn | FastAPI | ...
      database: PostgreSQL             # primary persistent store, if applicable
      cache: Redis                     # optional
      experiment_tracking: MLflow      # optional
      model_registry: <registry>       # optional
    exposes:
      - REST: METHOD /path             # e.g. REST: GET /api/policies
      - GRPC: ServiceName.Method       # if applicable
    publishes:
      - event: <event.name>
        transport: redis | sqs | kafka
        payload_ref: <DTO name or schema file path>   # optional
    subscribes:
      - event: <event.name>
        from: <publishing-service-name>
    calls:
      - service: <service-name>
        via: REST | GRPC
        purpose: one sentence why        # e.g. "load a registered model for batch scoring"
    env_vars:
      - name: DATABASE_URL
        required: true
        shared: false                   # true = also used by another service
    guardrails:
      - "Never read another workspace's database directly"
      - "Always validate input with class-validator before processing"

shared_infrastructure:
  databases:
    - name: <db-name>
      type: PostgreSQL | MongoDB | MySQL
      owner: <service-name>             # who owns schema migrations
      readers: [<service>, <service>]   # services with read access only
  caches:
    - name: <cache-name>
      type: Redis | Memcached
      used_by: [<service>, <service>]
  queues:
    - name: <queue-name>
      type: Redis | SQS | Kafka
      used_by: [<service>, <service>]
  data_platforms:
    - name: <platform-name>
      type: data lake | warehouse | lakehouse | other
      used_by: [<service>, <service>]
  feature_stores:
    - name: <feature-store-name>
      type: <technology>
      used_by: [<service>, <service>]
  model_registries:
    - name: <registry-name>
      type: <technology>
      used_by: [<service>, <service>]
  artifact_stores:
    - name: <store-name>
      type: object storage | package registry | other
      used_by: [<service>, <service>]

api_gateway:
  service: <service-folder-name>
  routes:
    - path: /v1/predictions
      target: model-serving
    - path: /v1/health
      target: model-serving

# CONTRACTS WITH THE OUTSIDE WORLD
# Features never span workspaces, but the code still has cross-workspace contracts.
# Declaring them here is what lets /prd and /plan flag a breaking change even though
# they cannot plan the other workspace's side of it.
external:
  consumed_by:                          # who outside depends on what we expose
    - workspace: model-monitoring
      depends_on:
        - event: model.evaluation.completed
        - REST: POST /v1/predictions
      contact: "<team-or-owner>"        # who to notify on a breaking change
  we_depend_on:                         # what we consume from outside
    - workspace: data-platform
      we_use:
        - model: <registered-model-name>
      notes: "Document owner, versioning, and compatibility expectations"
```

---

### If `$ARGUMENTS` is `--validate`

Read `architecture.yml`. Check for:

1. **Missing services** — any project folder with a recognized project manifest or `CLAUDE.md`
   that is NOT listed in `architecture.yml`
2. **Stale services** — services declared in `architecture.yml` whose folder does not exist
3. **Broken event references** — a service subscribes to an event but no service publishes it
4. **Broken call references** — a service calls another service not listed in `architecture.yml`
5. **Undeclared shared DBs** — two services declare the same DATABASE_URL env var but the DB
   is not listed under `shared_infrastructure`
6. **Missing external contracts** — events/endpoints in `exposes`/`publishes` with no
   entry under `external.consumed_by`, where an outside consumer is known to exist
7. **Missing guardrails** — services with no guardrails listed

Print a validation report:
```
architecture.yml validation
============================

✓ All service folders exist
✓ Event references consistent
⚠ feature-validation found in this workspace but not in architecture.yml
⚠ model-serving consumes a model from an undeclared model-training component
⚠ 2 services share DATABASE_URL but no shared DB declared in shared_infrastructure
✗ api-gateway has no routes declared

Issues found: 4 warnings, 1 error
Run /update-architecture (no args) to fix interactively.
```

Stop after printing — do not modify any files.

---

### Interactive mode (no arguments)

#### Step 1 — Check for existing architecture.yml

If `architecture.yml` exists at the workspace root:
- Read it fully
- If `services` is empty or `{}`, print: "Found the reusable architecture starter with no services. I will discover this workspace's projects and add them."
- Otherwise print: "Found existing architecture.yml with [N] services. I will update it, not replace it."
- Any service already declared will be preserved unless you find its folder is missing
- Preserve workspace metadata and guardrails when adding discovered services; do not carry over service entries from another workspace

If it does not exist:
- Print: "No architecture.yml found. I will create it from scratch."

---

#### Step 2 — Discover all services

Scan this workspace's subfolders for project markers such as `pyproject.toml`,
`requirements.txt`, `environment.yml`, `setup.py`, `package.json`, `composer.json`,
`Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `dvc.yaml`, `MLproject`, or
`CLAUDE.md`. A directory containing notebooks may also be a project when it has
supporting code, configuration, or data/model pipeline files. Ignore generated
environments, caches, data/artifact directories, and nested folders that belong to
an already-discovered project. Build a flat list of
`{ folder, hasClaudeMd, hasManifest }`. Only this workspace's repos are in scope —
never scan sibling workspaces.

Print the discovered list and ask:
```
# Example output — your actual values will differ
Workspace: <workspace-name>
Discovered services:
  - feature-pipeline       (has CLAUDE.md ✓)
  - model-training         (pyproject.toml only)
  - model-serving           (has CLAUDE.md ✓)

Are any services missing from this list? List them now, or press enter to continue.
```

Wait for response. Add any manually provided services to the list.

---

#### Step 3 — Gather service details

For each service, read its `CLAUDE.md` and relevant project manifests/configuration
(for example `pyproject.toml`, `requirements.txt`, `environment.yml`, `Dockerfile`,
`dvc.yaml`, or `MLproject`). Extract what can be established from files:
- Description, stack, env_vars, guardrails
- Workload type (data processing, feature engineering, training, evaluation, inference,
  serving, orchestration, or another observed responsibility)
- Relevant data/model inputs and outputs, artifact or model-registry integrations, and
  experiment/evaluation tooling when explicitly present

For details you cannot reliably infer (interfaces, events, data/model inputs and outputs,
or other components it calls), ask one focused question per component rather than a long form.

Ask in this format — batch all questions for one service together:
```
feature-pipeline — I need a few details I can't infer from code:

  1. What interfaces does it expose (APIs, CLI/jobs, pipeline entry points, or scheduled tasks)?
     (or type 'check code' and I will scan the relevant files)

  2. Does it publish any events? If yes: event name, transport (Redis/SQS/Kafka)

  3. What data, models, or artifacts does it consume or produce, and where are they stored?

  4. Does it call any other services directly? If yes: which service and why?

Answer each by number, or press enter to skip and mark as TODO.
```

Wait for answers. Then repeat for the next service.

For `api-gateway` specifically, also ask:
```
api-gateway — what routes does it proxy?
Format: /path → service-name  (one per line)
Or type 'check code' and I will scan the routing config.
```

If developer types `check code` for any question, scan the relevant source files:
- API routers/controllers (for example FastAPI or Flask route declarations)
- Pipeline, training, evaluation, batch-inference, and scheduled-job entry points
- Event handlers for subscriptions (`@OnEvent`, Redis subscriber patterns)
- HTTP/SDK client calls, data-store access, model-registry and artifact-store integrations

Report what you find and ask for confirmation before using it.

---

#### Step 4 — Gather shared infrastructure

Ask once for the whole system:
```
Shared infrastructure — what do these services share?

  1. Databases: list any DB used by more than one service, or owned by one but read by others
     Format: db-name (type) — owner: service-name — readers: [service, service]

  2. Caches: any shared Redis or Memcached instances?
     Format: cache-name — used by: [service, service]

  3. Queues/brokers: any shared SQS, Kafka, or Redis pub/sub channels?
     Format: queue-name (type) — used by: [service, service]

  4. Data platforms: any shared warehouse, lake, or lakehouse?
     Format: platform-name (type) — used by: [service, service]

  5. Feature stores, model registries, or artifact stores?
     Format: name (type) — used by: [service, service]

Press enter to skip any you don't have.
```

---

#### Step 5 — Write architecture.yml

Write the complete file to `architecture.yml` at the workspace root.

Rules:
- Use the format from `--help` exactly
- Mark any field where the answer was skipped as `# TODO`
- Preserve all fields from the existing file that were not updated in this session
- For each service, record the commit SHA the context was built from — this is the
  baseline `/sync` uses to detect context staleness later:
  ```bash
  cd [service-folder]
  git rev-parse HEAD
  ```
  Write it into the service's entry:
  ```yaml
  services:
    [service-name]:
      context_built_commit: [HEAD-sha]    # baseline for /sync staleness check
      context_built_date: [today's date]
      # ... rest of service config
  ```
  If a service folder is not a git repo or HEAD can't be read, omit the field and note it.
- Add a header comment:

```yaml
# architecture.yml — service relationship map
# Last updated: [today's date]
# Updated by: /update-architecture command
#
# This is the source of truth for service connections.
# Edit this file when services change, then run /update-context to refresh CLAUDE.md.
# Run /update-architecture --validate to check for inconsistencies.
# context_built_commit per service is the baseline /sync uses to detect drift.
```

---

#### Step 6 — Confirm workspace boundary rules

Before writing the file, check `workspace.name` and `workspace.description`.
If either is missing or still `TODO`, suggest a name from the workspace directory and
a one-sentence description from the discovered project responsibilities. Ask the
developer to confirm or edit both; do not leave starter placeholders in a completed map.

Then check workspace boundary rules. If `workspace.boundary_rules` is empty or missing,
ask once:

```
Boundary rules live in `architecture.yml` under `workspace.boundary_rules`. This workspace
has no boundary rules recorded. These define how OTHER workspaces
may interact with this one — used to flag breaking changes to external consumers.

Examples:
  - "Other workspaces consume our data via events only — never direct DB reads"
  - "/api/internal/* is not a public contract and may change without notice"

Provide them now, or press enter to leave as TODO.
```

Write the answers into `workspace.boundary_rules`.



---

#### Step 7 — Print completion report

```
✓ architecture.yml written ([N] services, [N] events, [N] REST routes)

Domain README stubs created:
  ✓ platform/README.md         (stub — fill in boundary rules)

Fields marked TODO (need your input):
  - premium-calculator: exposes (could not infer, check code manually)
  - notification-service: publishes (you skipped this)
  - shared_infrastructure: queues (you skipped this)

Next steps:
  1. Fill in TODO fields in architecture.yml
  2. Fill in workspace.boundary_rules and the external: contracts block
  3. Run /update-architecture --validate to check consistency
  4. Run /update-context to regenerate root CLAUDE.md
  5. Commit both files to version control
```
