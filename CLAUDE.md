# AI/ML Workspace — System Context

> Starter context for a reusable Claude Code workflow. This file is generated from
> `architecture.yml` in a configured project workspace.

## Workspace status

This repository contains reusable workflow commands and templates, not a configured
application workspace. Its `architecture.yml` intentionally starts with an empty
`services` map. When these files are placed at the root of a project workspace, run
`/update-architecture` to discover and register its local services, pipelines, model
projects, and supporting components.

## Working principles

- Derive implementation details from the target repository's code, configuration,
  tests, and documentation; do not assume a particular language, framework, or model stack.
- Treat datasets, prompts, model artifacts, and logs according to the project's data
  handling requirements.
- Preserve reproducibility and make uncertainty, evaluation limits, and incomplete
  results explicit.
- Use each project's existing tools and conventions for development, evaluation,
  testing, packaging, and deployment.

Run `/update-context` after architecture or service context changes to generate the
project-specific system overview.
