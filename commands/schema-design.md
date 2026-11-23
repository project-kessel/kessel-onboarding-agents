---
description: Generate draft Kessel schemas from an approved ServiceProfile — a unified Starlark schema for permissions-only services, or the legacy inventory-api + rbac-config KSL pair for services that define a resource type.
argument-hint: "--profile PATH [--codebase_ref PATH] [--unified_schema_path PATH] [--output_dir PATH]"
---

## Name

kessel-onboarding:schema-design

## Synopsis

```
/kessel-onboarding:schema-design --profile artifacts/profiles/{slug}-profile.json
/kessel-onboarding:schema-design --profile artifacts/profiles/activation-keys-profile.json --codebase_ref ~/dev/activation-keys
/kessel-onboarding:schema-design --profile artifacts/profiles/host-based-inventory-profile.json --output_dir ./schemas-draft
```

_For OpenAI, substitute `/kessel-onboarding:schema-design` with `@kessel-onboarding onboarding:schema-design` for any commands._

## Description

Reads a completed ServiceProfile and generates the schema files a service needs for Kessel integration. Conducts a short follow-up Q&A for reporter naming, v2 permission names, and reporter-specific fields, then writes all output as local files.

**Workflow:** Load profile → Classify asset types → **Choose schema source** → Reporter Q&A → Permission naming Q&A → Generate schema → Generate permissions.json + roles.json → Present for review

**Schema source.** Services that onboard **permissions only** — no resource type of their own, nothing reported to inventory — get a single Starlark file for [starlark-unified-schema](https://github.com/project-kessel/starlark-unified-schema), which compiles to the downstream artifacts. No hand-authored KSL and no inventory-api JSON Schema are generated for them. Services that define a resource type still get the legacy pair. The skill decides from the asset-type classification and states the verdict and its reason before generating.

Nothing is written to Jira or any external system. All artifacts land in `./artifacts/schemas/{slug}/` (or `--output_dir`).

## Implementation

Load and execute [skills/onboarding-schema-design/SKILL.md](../skills/onboarding-schema-design/SKILL.md).

## Examples

```
/kessel-onboarding:schema-design --profile artifacts/profiles/activation-keys-profile.json
/kessel-onboarding:schema-design --profile artifacts/profiles/host-based-inventory-profile.json --codebase_ref ~/dev/insights-host-inventory
```

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--profile PATH` | Yes | Path to `{slug}-profile.json` from Interview Agent |
| `--codebase_ref PATH` | Yes* | GitHub/GitLab URL or local path to service repo. *Can be omitted if the profile already has `interview.codebase_ref` set; otherwise the skill will ask for it before proceeding. |
| `--unified_schema_path PATH` | No | GitHub URL or local path to `starlark-unified-schema`. Used to check for an existing namespace, cross-check extension signatures, and compile the generated `.star`. Falls back to `~/dev/starlark-unified-schema` and `../starlark-unified-schema`. |
| `--output_dir PATH` | No | Output directory (default `./artifacts/schemas/{slug}/`) |
| `--test-mode` | No | Activates the Kessel blindfold — ignores existing KSL and Starlark schema sources, inventory-api resource schema dirs, and Kessel client code during codebase analysis. Artifacts written to `artifacts/test/{slug}/schemas/` instead of `artifacts/schemas/{slug}/`. |

## Return value

Paths to all generated files under the output directory.

Permissions-only services (unified schema source):
- `starlark-unified-schema/schema/{namespace}.star`
- `starlark-unified-schema/Makefile.patch` — the `SHIPPED_SCHEMA_FILES` / `KSL_RELEASE_FILES` entries the PR needs
- `rbac-config/permissions/{app}.json`
- `rbac-config/roles/{app}.json`
- `README.md` with next steps and validation commands

Services that define a resource type (legacy schema source):
- `inventory-api/{asset_type}/` directories (one per resource type)
- `rbac-config/schemas/src/{namespace}.ksl`
- `rbac-config/permissions/{app}.json`
- `rbac-config/roles/{app}.json`
- `README.md` with next steps and validation commands

## Prerequisites

- ServiceProfile must have `asset_types[]`, `v1_permissions`, and `patterns[]` populated (run `/kessel-onboarding:interview` first)
- **Codebase access is required** — the skill will not generate schemas without analyzing the service's source code
- `~/.config/kessel-onboarding/config.json` present (for `artifacts_dir`)
- If the interview was deferred, check for `{slug}-schema-context.md` alongside the profile — it contains interview findings and open questions relevant to schema design

## Changelog

- 2026-11: Permissions-only services now default to a unified Starlark schema instead of hand-authored KSL + inventory-api JSON Schema. Added `--unified_schema_path`, the schema-source step in the workflow, and a per-path return value list.
- 2026-09: Added the OpenAI invocation substitution for command examples.
- 2026-07: Initial version.

Assisted-by: Claude (Anthropic)
