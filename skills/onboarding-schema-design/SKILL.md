---
name: onboarding-schema-design
description: >
  Phase 2 schema design for Kessel service onboarding. Reads a ServiceProfile
  from the interview skill, conducts a short follow-up Q&A for reporter and
  permission naming details, then generates draft schemas. Permissions-only
  services get a single Starlark file for starlark-unified-schema; services that
  define their own resource type get the legacy pair (inventory-api JSON Schema +
  rbac-config KSL). Both get permissions.json + roles.json.
  All output is local files — no Jira or external writes.
---

# Onboarding schema design

## When to use

- After a service's onboarding interview is complete (Phase 0/1) and before Phase 2 schema modeling begins — gives the team a head start on resource and permissions schemas.
- Only when a codebase reference (repo URL, local path, or archive) is available — this skill stops and asks for one if it's missing, since schema generation without code access produces output that needs too much manual correction to be useful.

## What this skill does

Takes a completed ServiceProfile from the interview skill and generates the schema files a service needs for Kessel integration. **Which files depends on whether the service onboards permissions only, or also defines its own resource type** — see Step 1.5 for the eligibility rule.

**Permissions-only services (the common case — default to this when eligible):**

1. **Unified schema** — a single `schema/{app}.star` Starlark file for [starlark-unified-schema](https://github.com/project-kessel/starlark-unified-schema), which compiles to the downstream KSIL. No hand-authored `.ksl`, no inventory-api JSON Schema.
2. **RBAC registration** for rbac-config — `permissions.json` + `roles.json`

**Services that define a resource type / report to inventory:**

1. **Resource schemas** for inventory-api — `config.yaml` + JSON Schema files per resource type
2. **Permissions schemas** for rbac-config — `.ksl` file + `permissions.json` + `roles.json`

Conducts a short follow-up Q&A to fill in details the interview defers to Phase 2 (reporter naming, v2 permission names, reporter-specific fields, cross-service dependencies).

Does **not** create PRs, Jira issues, or any external resources. All output is local files under `artifacts/schemas/{slug}/`.

## Inputs

| Input | Required | Source |
|-------|----------|--------|
| ServiceProfile JSON path | yes | Interview output or `--profile` flag |
| `codebase_ref` | **yes** | From profile `interview.codebase_ref` or `--codebase_ref` flag |
| `rbac_config_path` | no | Local path to rbac-config repo root. Used by Step 8.5 for permissions/roles structural validation, and for the KSL syntax check on the legacy path. If omitted, checks `~/dev/rbac-config` and `../rbac-config`. |
| `unified_schema_path` | no | Local path or git URL to the `starlark-unified-schema` repo root. Used on the unified path for the existing-namespace check (Step 0), extension signature cross-check (Step 5A), and the interpreter build (Step 8.5). If omitted, checks `~/dev/starlark-unified-schema` and `../starlark-unified-schema`; if still not found, generate the `.star` anyway and report the build check as unvalidated. |
| `output_dir` | no | Default `./artifacts/schemas/{slug}/` |
| `test_mode` | no | When `true`, activates the Kessel blindfold and routes output to `{artifacts_dir}/test/{slug}/schemas/` |

## Prerequisites

- ServiceProfile must have `asset_types[]`, `v1_permissions`, and `patterns[]` populated (run the interview skill first)
- `dedup.status` should not be `duplicate_found`
- **Codebase access is required.** The service's source code provides essential context for resource type modeling, reporter field definitions, and permission mapping. If `interview.codebase_ref` is null and `--codebase_ref` is not provided, **stop and ask** for a repo URL, local path, or archive before proceeding. Do not attempt to generate schemas without codebase analysis — the output would require too much manual correction to be useful.

## Test mode

When `test_mode = true` (set via `--test-mode` flag or passed from the interview agent):

**Banner:** Display `⚗️ TEST MODE — Existing Kessel schemas and implementation will be ignored. Outputs written to artifacts/test/{slug}/schemas/`.

**Output routing:** Write all artifacts to `{artifacts_dir}/test/{slug}/schemas/` instead of `{artifacts_dir}/schemas/{slug}/`.

**Codebase blindfold — ignore during Step 0 analysis:**

| What to ignore | Examples |
|---|---|
| Existing KSL files for this service | Any `.ksl` in `rbac-config/configs/*/schemas/src/` matching the service namespace |
| Existing Starlark schema sources for this service | Any `schema/{app}.star` or `schema/*/reporters/{reporter}/*.star` in `starlark-unified-schema` matching the service namespace, and any matching `{app}.json` KSIL in `rbac-config/configs/*/schemas/src/` |
| Existing inventory-api resource schema dirs | Any `data/schema/resources/{type}/` dirs for this service's asset types |
| Existing Kessel permission class definitions | `KesselPermission`, `KesselResourceType`, v2 permission name constants |
| Existing Kessel client code | `lib/kessel.py`, `ClientBuilder` setup, `Check`/`CheckBulk` call sites, `kessel.rbac.v2.list_workspaces()` (Kessel SDK — distinct from `/api/rbac/v2/workspaces/` RBAC REST calls which are kept) |
| `migrated_apps.lst` entries for this service | Treat the service as not yet migrated |

**Still analyse — derive schemas from pre-Kessel signals:**

Every service onboarding will already be using the RBAC v1 service — read all of that integration code, not just enums and constants.

| What to keep | Why |
|---|---|
| Domain model classes, DB columns | Source of reporter-specific JSON Schema fields |
| **All RBAC service integration code (v1 and v2)** | The migration surface — read both v1 and v2 fully |
| v1 RBAC enums/constants | `RbacPermission`, `RbacResourceType` — the v1 permission surface |
| v1 RBAC middleware and permission-checking logic | Full middleware file — reveals the complete permission enforcement model |
| v1 RBAC client calls | `RBAC_ROUTE`, `get_rbac_filter()`, any function calling `/api/rbac/v1/access/` |
| v2 RBAC REST API calls | `get_rbac_workspace_by_id()`, `get_rbac_workspaces()`, `RBAC_V2_ROUTE`, any direct call to `/api/rbac/v2/workspaces/` — the RBAC service's own workspace API, not Kessel. Reveals whether the service already models workspace/group concepts. |
| Existing rbac-config permissions/roles files for this service | `permissions/{app}.json`, `roles/{app}.json` — the definitions being migrated into KSL |
| API resource names and endpoint handlers | Confirms asset type scope and permission surface |
| `patterns[]` rationale annotations from the interview | If the interview annotated an asset type as "uses existing rbac.workspace type — do NOT generate a public type", respect that instruction in Step 1 classification — do not generate a `public type` or inventory-api resource schema for that asset type |

**Reporter name in test mode:** Since existing Kessel client code is ignored, the reporter name cannot be inferred from namespace config. Ask the reporter Q&A question cold (Step 2) for all asset types that need a resource schema.

## Configuration

Read `~/.config/kessel-onboarding/config.json` for `artifacts_dir`. If missing, direct the user to [docs/configuration.md](../../docs/configuration.md).

## Execution

### Step 0 — Load profile, validate codebase access, and load reference

Read the ServiceProfile JSON. Validate:

- `asset_types[]` is non-empty
- `v1_permissions.items` is non-empty (or explicit "unknown" flagged)
- `patterns[]` is non-empty and every `asset_types[]` entry appears in exactly one pattern's `asset_types[]`

**Codebase access gate:** Resolve `codebase_ref` from the profile's `interview.codebase_ref` field or the `--codebase_ref` flag. If neither is set, **stop and ask**: "Schema design requires access to the service's source code. Please provide a GitHub/GitLab URL, local path, or archive."

Once `codebase_ref` is available, attempt to access it. If access fails (private repo, no matching tool, unreadable path), **stop and report the error** — do not fall back to asking questions cold. The user must resolve access before continuing.

**URL inputs — clone before reading:** For each path input (`codebase_ref`, `rbac_config_path`, `inventory_api_path`, `unified_schema_path`): if the value is a GitHub or GitLab URL (contains `github.com` or `gitlab.com`), clone the repository to a local temp directory (`work/codebase/`, `work/rbac-config/`, `work/inventory-api/`, `work/unified-schema/` respectively) before any analysis. Do **not** fetch individual files via URL — rate limits, private repo auth, blob URL normalization, and lack of directory traversal make URL fetching unreliable. Use the local clone path for all subsequent reads within this skill run. If cloning fails, **stop and report the error** — do not fall back to URL fetching.

**Existing namespace check — non-test mode only:** Before generating any schema source, check **both** repos for an existing definition of the target namespace:

1. `{rbac_config_path}/configs/stage/schemas/src/` — look for `{namespace}.ksl` or `{namespace}.json`. A `.json` here may be generated KSIL released from starlark-unified-schema rather than hand-authored; check the unified-schema repo before assuming it is editable in place.
2. `{unified_schema_path}/schema/` — look for `{namespace}.star` at the top level, and for `{namespace}` as a reporter under `schema/*/reporters/{namespace}/`.

If found in either:
1. Read the file in full.
2. Generate only **additive** content — do not re-declare types, relations, or extensions already present in the existing file. Identical extension calls are deduplicated by the Starlark interpreter, but re-declaring them is still noise in review.
3. Note in the generated README which declarations were omitted because they already exist, and which file they live in.
4. **If the namespace already exists as Starlark, stay on the unified path** regardless of the Step 1.5 eligibility verdict, and generate an additive `{app}.star` edit. Do not introduce a competing hand-authored `.ksl` for a namespace already authored in Starlark — KSL is not forward-compatible with Starlark extensions.

In test mode, skip this check (the Kessel blindfold applies to existing KSL and Starlark schema files alike).

When access succeeds, perform initial codebase analysis before starting the Q&A:

| What to look for | Where to look |
|---|---|
| Domain models / resource types | Model classes, DB migrations, API resource definitions, protobuf messages |
| Reporter-specific fields | **Primary:** actual `ReportResource` call sites — outbox writers, gRPC client code, event publisher payloads. **Secondary (only when no call sites found):** DB columns, API response schemas, serializers, marshmallow/pydantic schemas, protobuf messages |
| Existing Kessel SDK usage | `kessel-sdk` imports, `ClientBuilder` calls, permission constants, `KesselPermission` classes |
| Existing permission definitions | rbac-config references, permission YAML/JSON, `@access` decorators |
| Service namespace / reporter name | App config, Kessel client setup, ClowdApp config |
| Cross-service dependencies | Stacked `@access` decorators, imports from other services' permission modules |

Carry all analysis results into Steps 1–3 as drafts for the confirm-or-correct pattern. **Never invent file contents** — only draft a value when the agent actually read a matching file.

Check for a schema-context snapshot at `{artifacts_dir}/profiles/{slug}-schema-context.md`. If present, this means the user deferred schema design from a prior interview session — load the snapshot for additional context (interview findings, codebase analysis notes, open questions). The snapshot supplements the profile but the profile JSON is always authoritative for field values.

Read [reference.md](reference.md) for schema format templates and naming conventions.

**Tool preflight:** Before generating any files, check which validators are available in the session environment and report their status. This determines what validation can be run in Step 8.5.

| Tool | Check command | Used for |
|---|---|---|
| `go` + `unified_schema_path` | `which go` and the resolved `unified_schema_path` | Building the Starlark interpreter to compile the generated `.star` (unified path) |
| `ksl` binary | `which ksl \|\| ls $GOBIN/ksl \|\| ls ~/go/bin/ksl` | KSL syntax + import check (legacy path) |
| `jq` | `which jq` | JSON syntax validation of all `.json` files |
| `python3` + `jsonschema` | `python3 -c "import jsonschema"` | Structural validation of permissions.json/roles.json against rbac-config schemas; draft-07 meta-validation of inventory-api JSON Schema files |

If `go` or `unified_schema_path` is unavailable, note: "Starlark compile check unavailable — clone https://github.com/project-kessel/starlark-unified-schema, run `make build-interpreter` (requires Go), then re-run schema-design or validate manually with `make build-shipped-schema`."

If `ksl` is not found, note: "KSL syntax check unavailable — install with `make init` in the rbac-config repo (requires Go), then re-run schema-design or manually validate with `make ksl-test-schema-stage`."

Do not block generation on missing tools — report availability and proceed. Validation happens in Step 8.5 after files are written.

### Step 1 — Classify each asset type

`patterns[]` is the sole authoritative source for classification. For each entry in `asset_types[]`, determine which schema artifacts it needs based on its matched pattern:

| Pattern | Needs inventory-api resource schema? | Needs KSL `public type` definition? |
|---|---|---|
| `native` or `native-ws-list` | Yes (when `inventory_migration_required = true`) | Yes — `public type` with workspace relation |
| `default-workspace` | Yes (when `inventory_migration_required = true`) | **Yes** — `public type` with workspace relation |
| `root-workspace` | No | No — workspace-level permissions only |
| `org-level` | No | No — workspace-level permissions only |

**Why `default-workspace` requires a `public type`:** When resources are reported to inventory-api, the consumer always writes a workspace relationship tuple of the form `{namespace}/{type}:{id}#workspace@rbac/workspace:{workspace_id}` into SpiceDB. SpiceDB will reject this write with `object definition not found` if the type is not declared in the schema. This is required regardless of whether the application calls `Check` on the resource directly or on the default workspace — SpiceDB must know the type exists to store the tuple.

If the interview narrative summary notes that an asset type "maps to rbac.workspace" (from the Group 5 ownership follow-up) but its pattern is `native` or `native-ws-list`, that is a contradiction — pause and ask the EM to confirm which is correct before proceeding. Do not silently override the pattern; pattern classification and platform-type ownership must agree.

**"New type vs workspace extension" — ask when language is ambiguous:** If the interview narrative uses language like "extends the workspace hierarchy," "workspace-aware," or "workspace-scoped" for an asset type without explicitly confirming it is a new standalone resource type, ask before generating: "Is `{asset_type}` a new resource type that will exist alongside workspaces (needs a `public type` block), or will it attach additional relations to the existing `rbac.workspace` type (needs a `public extension` on workspace)?" These require completely different KSL outputs and cannot be inferred from the description alone. Record the EM's answer as a classification annotation before proceeding.

Present a classification table to the EM/tech lead for confirmation before proceeding.

Asset types matched to `root-workspace` or `org-level` do not get inventory-api resource schemas — they use the existing `rbac.workspace` type for permission checks and do not report resources to inventory.

Asset types matched to `native`, `native-ws-list`, or `default-workspace` but with `inventory_migration_required = false` also skip resource schema and KSL type generation — flag this as a follow-up for when the team is ready to report to inventory.

### Step 1.5 — Choose the schema source: unified Starlark or legacy KSL

Decide once, for the whole service, which schema generation to target. Read the
[Unified schema (Starlark)](reference.md#unified-schema-starlark--default-for-permissions-only-services)
section of reference.md before applying this rule.

**The service is unified-schema eligible when the Step 1 table assigned "No" to the
`Needs KSL public type definition?` column for every asset type** — i.e. the service onboards
permissions only and declares no resource type of its own. In practice this means every asset
type is `root-workspace` / `org-level`, or is `native` / `native-ws-list` / `default-workspace`
with `inventory_migration_required = false`.

| Verdict | Generate | Skip |
|---|---|---|
| **Eligible — default to this** | `{app}.star` (Step 5A), `permissions.json` (Step 6), `roles.json` (Step 7) | Step 4 (inventory-api resource schemas), Step 5B (KSL) |
| **Not eligible** | inventory-api resource schemas (Step 4), `{namespace}.ksl` (Step 5B), `permissions.json`, `roles.json` | Step 5A |

Overrides, in precedence order:

1. **Namespace already authored in Starlark** (found by the Step 0 existing-namespace check) →
   unified path, additive, regardless of eligibility. KSL is not forward-compatible with Starlark
   extensions, so a namespace must not be split across both.
2. **Namespace already hand-authored as `.ksl` in rbac-config** and the service is eligible →
   ask the EM: "`{namespace}.ksl` already exists in rbac-config. Add these permissions to the
   existing KSL file, or migrate the namespace to a Starlark `{app}.star` in
   starlark-unified-schema? Migrating moves the whole namespace at once — all existing
   declarations must come along, and the `.ksl` is removed in the same change." Default to adding
   to the existing `.ksl` unless the EM opts into the migration; a partial move produces two
   sources for one namespace.
3. **Not eligible** → legacy path. Record in the README that `resource()` in the unified schema is
   the forward direction for resource-defining services, and the team should confirm the current
   delivery path with the Kessel team before hand-authoring the next change.

State the verdict and the reason to the EM before continuing — one line, not a question:

> Schema source: **unified Starlark** (`schema/task_manager.star`) — no asset type needs a
> resource type definition, so this service onboards permissions only.

Record the verdict as `schema_source` (`unified` or `ksl`); Steps 4–9, the README, and
`migrate-context.md` all branch on it.

### Step 2 — Reporter Q&A (one question per turn per asset type needing a resource schema)

**Ask one question at a time and wait for the answer before asking the next.** Never combine multiple questions into a single message — this applies on both Claude and Codex. For each asset type classified as needing an inventory-api resource schema in Step 1, work through the following questions one at a time:

1. **Reporter name** — "What short name identifies your service as a reporter for `{asset_type}` resources? This becomes the reporter namespace in inventory-api (e.g. `hbi` for Host Based Inventory, `acm` for Advanced Cluster Management, `notifications` for Notifications). Must be lowercase, no hyphens."

2. **Reporter-specific fields — payload, not domain model:** "What fields does your service write into the `Reporter` struct of a `ReportResource` gRPC call for each `{asset_type}`? These are **not** all fields on the domain model or API response — reporter schemas are deliberately minimal. Typically a small set of correlation identifiers (e.g. satellite_id, subscription_manager_id, insights_id for HBI hosts) that other services use to cross-reference resources." Draft candidate fields from actual `ReportResource` call sites in the codebase (outbox writers, gRPC client code) rather than domain model classes or API serializers. If you cannot find actual reporting code, ask explicitly: "What fields do you include in the reporter section of a ReportResource call — just the identifiers or display fields sent to Kessel, not the full object?" For each confirmed field, collect:
   - Field name (snake_case)
   - Type: `string`, `number`, `integer`, `boolean`, `array`, or `object`
   - Required or optional
   - Any constraints: enum values, format (`uuid`, `date-time`, `uri`), pattern (regex), maxLength
   - Whether it can be null (adds `oneOf` with null type)

3. **Common representation fields** — "Does your `{asset_type}` need any common fields beyond `workspace_id`? (Common fields are shared across all reporters for this resource type. Most services only need `workspace_id`.)"

If `codebase_ref` produced model/schema analysis during the interview, draft answers from it (same confirm-or-correct pattern — never auto-accept). Look for:

| Signal | Where to look |
|---|---|
| Reporter name | Service slug, app config, or existing Kessel SDK client setup |
| Reporter fields | Actual `ReportResource` call sites (outbox writers, gRPC client code) — **not** domain model classes or API serializers |
| Common fields | Fields shared across different data sources for the same resource |

**Inferred types not in `asset_types[]` — require EM gate:** If codebase analysis reveals a resource type not listed in the profile's `asset_types[]` (e.g. an `assignment` type implied by permissions found in the codebase), do **not** add it to the generated schemas without EM confirmation. Ask: "I found evidence of a `{type}` resource in the codebase — should it be added as an inventoried Kessel asset type? If yes, it will be added to `asset_types[]` and given a resource schema and KSL type." Only proceed after explicit EM confirmation; update the profile's `asset_types[]` to match.

### Step 3 — Permission naming Q&A

**Step 3a — Namespace and file consolidation**

Ask: "What namespace should your schema use? Convention is a short lowercase identifier (hyphens become underscores). Examples: `hbi`, `notifications`, `content_sources`, `compliance`."

If `codebase_ref` analysis found Kessel SDK usage with a namespace, draft the answer.

The answer is load-bearing on both paths, in slightly different ways:

- **Unified path** — the namespace is the `reporter` argument passed to every extension wrapper **and** the `.star` file name stem (`schema/{namespace}.star`), which determines the output KSIL file (`{namespace}.json`). These three must agree or the app's references scatter across namespaces.
- **Legacy path** — the namespace is the `namespace` header of `{namespace}.ksl`.

Either way it is distinct from `app`, which must match the rbac-config permissions JSON filename.

**Multi-namespace consolidation:** If `v1_permissions.items` spans more than one v1 app namespace (e.g. both `inventory:*` and `staleness:*`), ask: "Are all of these v1 app namespaces owned and operated by the same deployable service? If yes, should they be consolidated into a single schema file under one namespace, or kept in separate files?" The answer determines whether to generate one combined file or one per namespace.

**Default:** If the EM confirms the same service owns all namespaces and does not specify, generate a single schema file using the primary namespace and consolidate secondary namespace declarations at the top of that file with a comment. Do not generate separate schema files for secondary namespaces unless the EM explicitly requests it or the namespaces are owned by different deployable services.

On the unified path the same consolidation applies, and it is purely about the `app` argument — one `{primary_namespace}.star` file, every call passing `"{primary_namespace}"` as the reporter, with the secondary app's calls differing only in their `app` value (e.g. `add_v1_based_permission("hbi", "staleness", "staleness", "read", "staleness_staleness_view")`). Do **not** vary the reporter per app; that splits one service's output across two KSIL files.

**Step 3b — V2 permission name mapping**

**Check for existing v2 name evidence before proposing:** Before applying the naming convention, search the codebase and any available context for existing v2 permission names: Kessel SDK permission constants or enums, feature flag names referencing the permission (e.g. `hbi.rbac-v2`), prior KSL files, or migrate-context notes. If an existing v2 name is found, use it — do not rename to match convention. Apply the convention-derived name only when no existing name is found, and ask for explicit EM confirmation before finalizing any name that differs from what the convention would produce.

Present a proposed mapping table derived from `v1_permissions.items` using the naming convention `{app}_{resource_singular}_{action}`:

| v1 action | v2 action mapping |
|---|---|
| `read` | `view` |
| `write` | Ask: "Does `write` cover update, delete, move, or a combination? List which." |
| `create` | `new` |
| `delete` | `remove` or `delete` |
| `*` | `admin` (or expand to individual permissions) |

For each v1 permission, propose the v2 name and ask for confirmation **one permission at a time** — do not present the entire mapping table as a single prompt. Wait for confirmation on each before moving to the next. Example of a single-question turn:

```
v1: inventory:hosts:read   → v2: inventory_host_view          ✓ confirm?
v1: inventory:hosts:write  → v2: inventory_host_update         ✓ confirm?
                              → v2: inventory_host_delete       (if write includes delete)
                              → v2: inventory_host_move         (if write includes move)
```

**Step 3c — Cross-service dependencies**

Ask: "Do any of your permissions require the user to also have a permission from another service? For example, viewing patch results requires `inventory:hosts:read`. If yes, list which of your permissions depend on which external permission."

This determines whether the KSL file needs:
- `@rbac.add_contingent_permission(first, second, contingent)` calls
- `import hbi` and `@hbi.expose_host_permission(v2_perm, host_perm)` calls (for host-scoped permissions)

**Step 3d — Extension point (only for services defining `public type` resources)**

If any asset type requires a KSL `public type` definition (native, native-ws-list, or default-workspace pattern with inventory migration), ask:

> "Will other services in the platform need to check permissions on individual `{asset_type}` records your service manages — not just 'does the user have permission in this workspace', but 'does the user have permission on this specific `{asset_type}`?' If yes, your KSL needs a public extension so those services can scope their own permissions to your resource instances."

**Policy:**
- EM says **yes** → include `expose_{asset_type}_permission`.
- EM says **no** → omit.
- EM is **unsure** → omit. New services have no existing dependents; a missing extension does not cause KSL compilation failures. Note in the generated README that the extension can be added later if other services need to scope permissions to individual `{asset_type}` objects.

### Step 4 — Generate inventory-api resource schemas

**Skip this step entirely when `schema_source = unified`.** A permissions-only service reports nothing to inventory, so there is no resource schema to generate. Note the skip and its reason in the README (Step 8).

For each asset type needing a resource schema, generate the directory tree under `{output_dir}/inventory-api/`:

```
{asset_type}/
  config.yaml
  common_representation.json
  reporters/{reporter_name}/
    config.yaml
    {asset_type}.json
```

**config.yaml** (resource level):
```yaml
resource_type: {asset_type}
resource_reporters:
  - {reporter_name}
```

**common_representation.json**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "workspace_id": { "type": "string" }
  },
  "required": [
    "workspace_id"
  ]
}
```
Add any additional common fields from Step 2.

**reporters/{reporter_name}/config.yaml**:
```yaml
resource_type: {asset_type}
reporter_name: {reporter_name}
namespace: {reporter_name}
```

**reporters/{reporter_name}/{asset_type}.json**:
JSON Schema draft-07 from Step 2 reporter-specific fields. For nullable fields, use `oneOf` with null:
```json
{
  "field_name": {
    "oneOf": [
      { "type": "string", "format": "uuid" },
      { "type": "null" }
    ]
  }
}
```

### Step 5A — Generate the unified Starlark schema (when `schema_source = unified`)

Generate `{output_dir}/starlark-unified-schema/schema/{namespace}.star`. Skip Step 5B.

Read [reference.md](reference.md#unified-schema-starlark--default-for-permissions-only-services) for the full format, the verbatim wrapper signatures, and the KSL→Starlark translation table. Summary of what to emit:

**Structure:**

```python
load("extensions/rbac.star", {only the wrappers actually used, each as a quoted name})
{load("extensions/hbi.star", "expose_host_permission")  — only when exposing host permissions}

{extension calls, grouped by comment block: # App: {app} - Resource: {resource}}
```

No `version` line, no `namespace` line — the namespace is the `reporter` argument on each call.

**Extension selection** — same decision as the legacy path, different call syntax. Every wrapper takes `reporter` first:

| Situation | Call |
|---|---|
| Standard v1→v2 migration (most common) | `add_v1_based_permission("{namespace}", "{app}", "{resource}", "{verb}", "{v2_perm}")` |
| Permission that is v1-only and being deprecated | `add_v1only_permission("{namespace}", "{v2_perm}")` |
| Permission where v1 and v2 names are identical | `add_unified_permission("{namespace}", "{app}", "{resource}", "{verb}")` |
| Permission contingent on another service's permission | `add_contingent_permission("{namespace}", "{external_perm}", "{own_assigned_perm}", "{final_perm}")` |
| Host-scoped permission | `expose_host_permission("{namespace}", "{final_perm}", "{host_perm}")` |

**Generation rules — read before writing any calls:**

1. **Reporter must equal the file name stem** on every call, including calls for a secondary app namespace (Step 3a). A reporter that does not match the file name emits references into a KSIL file nobody ships.
2. **All arguments are positional string literals.** Not KSL's `app:'x'` keyword form, and not Starlark keyword arguments — the wrappers are documented and called positionally. Every parameter value must be a string.
3. **Do not declare permissions owned by `rbac.ksl`** (`rbac_workspace_edit`, `rbac_workspace_create`, `rbac_workspace_delete`, `rbac_workspace_move`). Same rule as the legacy path: they belong in `permissions.json` and `roles.json` only.
4. **Do not emit a `resource()` declaration.** If the Step 1 table called for a `public type`, the service is not eligible for this path — go back to Step 1.5.
5. **Discard return values.** The accessor a wrapper returns is only needed to wire a permission into a `resource()`. Assigning it to an unused variable is noise; call the wrapper as a statement.
6. **Do not repeat an identical call.** The interpreter deduplicates identical `(namespace, name, params)` tuples silently, so a duplicate is invisible at build time but still wrong in review.
7. **Starlark, not Python** — comments use `#`, no `import`, no f-strings, no classes.

**Cross-check the signatures before writing.** If `unified_schema_path` resolved, read `{unified_schema_path}/schema/extensions/rbac.star` and `extensions/hbi.star` and confirm each wrapper's parameter list matches what you are about to emit. The interpreter cannot verify that a KSL extension exists — a wrong name or arity compiles here and fails downstream in rbac-config's SpiceDB validation. If the path did not resolve, emit the calls per reference.md and flag in Step 9 that signatures were not cross-checked against the repo.

**Also generate `{output_dir}/starlark-unified-schema/Makefile.patch`** — a short plaintext note (not a real patch to apply blindly) listing the two Makefile edits the PR needs:

```
Add to SHIPPED_SCHEMA_FILES:   {namespace}.star
Add to KSL_RELEASE_FILES:      {namespace}.json
```

Without both, the file is never compiled for release and the KSIL is never shipped to rbac-config.

### Step 5B — Generate KSL permissions schema (when `schema_source = ksl`)

Generate `{output_dir}/rbac-config/schemas/src/{namespace}.ksl`. Skip Step 5A.

**Structure:**

```
version 0.1
namespace {namespace}

import rbac
{import hbi  — only if using expose_host_permission or contingent with inventory_host_view}
```

**KSL generation rules — read before writing any declarations:**

1. **No redundant top-level declarations for `public type` permissions.** If a permission is declared inside a `public type` block, do NOT also emit a standalone `@rbac.add_v1_based_permission(...)` for it at the top level. The type block's extension auto-generates all required workspace-propagation chains. Emitting both produces duplicate/incorrect KSL.

2. **Do not emit permissions for types owned by `rbac.ksl`.** Asset types that map to `rbac.workspace` (identified in Step 1 via the interview's asset type ownership notes, or by a `KesselResourceType(namespace="rbac")` in the codebase) have their write-operation permissions (`rbac_workspace_edit`, `rbac_workspace_create`, `rbac_workspace_delete`, `rbac_workspace_move`) declared in `rbac.ksl`. The service's KSL should **not** re-declare these. Include them in `permissions.json` and `roles.json` as needed, but not in the KSL.

3. **Top-level declarations are only for workspace-level permissions that have no corresponding `public type` block.** Typical uses: secondary-namespace permissions (e.g. `staleness:staleness:*`), and any permission not associated with a service-owned resource type.

**Permission declarations** — select the extension based on the pattern and Step 3 answers:

| Situation | Extension to use |
|---|---|
| Standard v1→v2 migration (most common) | `@rbac.add_v1_based_permission(app:'{app}', resource:'{resource}', verb:'{verb}', v2_perm:'{v2_perm}')` |
| Permission that is v1-only and being deprecated | `@rbac.add_v1only_permission(perm:'{v2_perm}')` |
| Permission where v1 and v2 names are identical | `@rbac.add_unified_permission(app:'{app}', resource:'{resource}', verb:'{verb}')` |
| Permission contingent on another service's permission | `@rbac.add_contingent_permission(first: '{external_perm}', second: '{own_assigned_perm}', contingent: '{final_perm}')` |
| Host-scoped permission (requires hbi import) | `@hbi.expose_host_permission(v2_perm: '{final_perm}', host_perm: '{host_perm}')` |

**Resource type definition** — only for native/native-ws-list patterns:

```
public type {asset_type} {
    private relation workspace: [ExactlyOne rbac.workspace]
    relation tenant: workspace.tenant

    @rbac.add_v1_based_permission(app:'{app}', resource:'{resource}', verb:'{verb}', v2_perm:'{v2_perm}')
    relation {action}: workspace.{v2_perm}
}
```

**Extension point** — only if Step 3d confirmed other services need to attach permissions:

```
public extension expose_{asset_type}_permission(v2_perm, {asset_type}_perm) {
    type {asset_type} {
        public relation `${{{asset_type}_perm}}`: view and workspace.`${{v2_perm}}`
    }
}
```

Group permissions by comment block (e.g. `// App: {app} - Resource: {resource}`).

### Step 6 — Generate permissions.json

Generate `{output_dir}/rbac-config/permissions/{app}.json` for each distinct v1 app namespace in `v1_permissions.items`.

Group permissions by resource, list verbs:

```json
{
  "{resource}": [
    { "verb": "read" },
    { "verb": "write" }
  ]
}
```

If v1 permissions span multiple app namespaces (e.g. `inventory` and `staleness`), generate one file per app.

### Step 7 — Generate roles.json

**Check for existing deployed file first — non-test mode only:** Before generating, check `{rbac_config_path}/configs/stage/roles/{app}.json`. If it exists, read it fully and use it as the baseline. Present the existing roles to the EM alongside what the scaffold would add or change, and ask which additions are wanted. Do **not** replace the existing file with a scaffold — only generate additive or corrective content. In test mode, skip this check and generate the scaffold from scratch.

**`platform_default` confirmation — always ask:** Before finalizing the viewer role, ask: "Should read access be granted to all platform users by default (`platform_default: true`), or should it require explicit assignment (`platform_default: false`)? Platform-default grants read to every user on the platform without role assignment — appropriate for some services but not for workspace-scoped resources where access should be controlled." Record the EM's answer and apply it to the viewer role. Do not default to `true` without confirmation.

Generate `{output_dir}/rbac-config/roles/{app}.json` with scaffold roles:

1. **{App} administrator** — `system: true`, `admin_default: true`, access: `["{app}:*:*"]`
2. **{App} viewer** — `system: true`, `platform_default: {EM-confirmed value}`, access: all `read` permissions listed individually

Use this template for each role:
```json
{
  "name": "{App} administrator",
  "description": "Perform any available operation on {App} resources.",
  "system": true,
  "platform_default": false,
  "admin_default": true,
  "version": 1,
  "access": [
    { "permission": "{app}:*:*" }
  ]
}
```

Add a note in the README (not in the JSON file itself) that roles are scaffolded defaults — the team should customize role names, descriptions, and permission bundles. Do **not** add a `_comment` or any other extra key to the roles JSON file; `roles.schema` uses `additionalProperties: false` and will reject any key that is not `roles`.

If multiple app namespaces, generate one roles file per app.

### Step 8 — Generate README

Write `{output_dir}/README.md` with:

1. File tree of all generated artifacts
2. **Schema source and why** — one line naming the Step 1.5 verdict (`unified` or `ksl`) and the reason, so a reviewer who did not see the session knows why there is (or is not) a `.ksl` file. On the unified path, state explicitly that no inventory-api resource schema and no `.ksl` were generated because the service onboards permissions only.
3. **Omitted asset types — explain any gap:** If any entry in the profile's `asset_types[]` was not given a generated schema (inventory-api resource schema, KSL type, or Starlark `resource()`), list it explicitly with a one-line rationale. Examples: "`policy` — subsumed into `role_binding` in the v2 model; no separate inventory schema needed", "`permission` — root-workspace pattern; uses existing `rbac.workspace` type, no inventory schema". Never silently omit. If the rationale is uncertain, flag it as an open question.
4. Next steps — **include only the branch matching `schema_source`**:

   **Unified path:**
   - Review and adjust all generated files
   - Copy `{namespace}.star` to `starlark-unified-schema/schema/`
   - Apply the two edits in `Makefile.patch`: add `{namespace}.star` to `SHIPPED_SCHEMA_FILES` and `{namespace}.json` to `KSL_RELEASE_FILES`
   - Build and confirm the output: `make build-interpreter && make build-shipped-schema`, then check `output/ksl/{namespace}.json` exists and contains the expected extension references
   - For permissions/roles JSON: copy to `rbac-config/configs/stage/permissions/` and `rbac-config/configs/stage/roles/`, validate against JSON schemas in `rbac-config/schemas/`
   - Add app name(s) to `rbac-config/configs/stage/schemas/migrated_apps.lst`
   - Open the starlark-unified-schema PR (`.star` source + Makefile change) and the rbac-config PR (permissions, roles, `migrated_apps.lst`). Merging the first publishes a dated release whose `ksl.tar.gz` rbac-config consumes into `configs/{env}/schemas/src/` — do not hand-copy the generated `{namespace}.json` into rbac-config.
   - Land the schema change **before** the service code that checks the new permissions

   **Legacy path:**
   - Review and adjust all generated files
   - For resource schemas: copy `{asset_type}/` directories to `inventory-api/data/schema/resources/`, run `go run main.go preload-schema`, run `make build-schemas`
   - For KSL: copy `{namespace}.ksl` to `rbac-config/configs/stage/schemas/src/`, run `make ksl-test-schema-stage` to validate
   - For permissions/roles JSON: copy to `rbac-config/configs/stage/permissions/` and `rbac-config/configs/stage/roles/`, validate against JSON schemas in `rbac-config/schemas/`
   - Add app name(s) to `rbac-config/configs/stage/schemas/migrated_apps.lst`
   - Open PRs against both repos for review

5. Validation commands (see Step 8.5 results for what was already run; include remaining manual steps)
6. Testing resources — include the following links verbatim in the generated README, grouped by purpose. **Include the "Unified schema (Starlark)" block only on the unified path, and the "inventory-api resource schema" block only on the legacy path.** The "Permissions modelling" and "Ephemeral environment testing" blocks go in both — the `.ksl` reference implementations and the SpiceDB Playground are useful for calibrating a permission model regardless of which syntax it is authored in.

```markdown
## Testing and validation resources

### Unified schema (Starlark)

- **Add a permission to Kessel** (step-by-step how-to)
  https://project-kessel.github.io/docs/building-with-kessel/how-to/design-permissions/
  The end-to-end guide this schema was generated against: permission naming, registering
  user-facing names in rbac-config, declaring the permission in a `.star` file, and submitting.

- **Kessel schema reference**
  https://project-kessel.github.io/docs/building-with-kessel/reference/schema/
  The schema model — reporters, representations, extensions and accessors, resource types,
  fields and cardinality, permission expressions, and which artifacts are generated.

- **Schema DSL API reference**
  https://github.com/project-kessel/starlark-unified-schema/blob/main/schema/API.md
  Every DSL symbol with its signature, generated from the source docstrings. Check wrapper
  arities here before editing the generated `.star`.

- **starlark-unified-schema repository**
  https://github.com/project-kessel/starlark-unified-schema
  Source for the extension wrappers (`schema/extensions/`), worked examples under `schema/`,
  the Makefile allowlists, and `ARCHITECTURE.md` for how `.star` compiles to KSIL and JSON Schema.

### Permissions modelling (applies to both schema generations)

- **Migration guide — Model the permissions**
  https://project-kessel.github.io/docs/building-with-kessel/how-to/migrate-from-rbac-v1-to-v2/
  Explains KSL extension patterns (`add_v1_based_permission`, `add_contingent_permission`, `expose_host_permission`),
  role definitions, and how to check in permission definitions to rbac-config.

- **rbac-config stage schemas** (reference implementations)
  https://github.com/project-kessel/rbac-config/tree/master/configs/stage/schemas/src
  Browse existing `.ksl` files to see how other services (advisor, patch, compliance, content-sources) structured
  their permission schemas. Useful for calibrating your own KSL before opening a PR.

- **SpiceDB Playground** (interactive KSL/zed testing — no running Kessel required)
  https://play.authzed.com/schema
  Run `make ksl-test-schema-stage` in rbac-config to compile your KSL to a `.zed` schema, then paste the output
  here to interactively test relationship checks (e.g. "does user X have `view` on host Y?") before deploying.
  This is the fastest way to verify your permission model behaves as intended.

- **KSL-016: Migrating host and organization level permissions** (internal, Red Hat only)
  https://docs.google.com/document/d/1XnINsHuYeHEi22q_1cS0gUalX-eXl3V19gGf0Wr8NsE/
  The source design document for the five KSL-016 patterns. Authoritative rationale for pattern selection,
  cardinality rules, and the workspace hierarchy model.

- **RBAC Platform documentation** (internal, Red Hat only)
  https://consoledot.pages.redhat.com/docs/dev/services/rbac.html
  RBAC service internals, workspace lookup APIs (`/v2/workspaces?type=default|root`), and service account setup.

### inventory-api resource schema

- **Kessel Inventory API — gRPC reference**
  https://buf.build/project-kessel/inventory-api/docs/main:kessel.inventory.v1beta2
  Full protobuf reference for `Check`, `CheckForUpdate`, `StreamedListObjects`, and `ReportResource` calls.
  Use alongside your resource schema to confirm field names and types match what the API expects.

- **inventory-api repository**
  https://github.com/project-kessel/inventory-api
  Source for the schema loader (`data/schema/resources/`), validation logic, and `go run main.go preload-schema`.

### Ephemeral environment testing

- **Validate in ephemeral environment** (migration guide section)
  https://project-kessel.github.io/docs/building-with-kessel/how-to/migrate-from-rbac-v1-to-v2/#validate-in-ephemeral-environment
  Step-by-step guide to deploying your service + Kessel in an ephemeral environment via insights-service-deployer,
  setting up authentication, and running end-to-end permission checks.

- **insights-service-deployer**
  https://github.com/project-kessel/insights-service-deployer/tree/main
  Tool for spinning up Kessel + your service in an ephemeral environment for integration testing.
```

### Step 8.5 — Validate generated artifacts

Run whichever validators are available (from the Step 0 preflight). All checks are structural — they verify the files are syntactically and schema-conformant, not that permissions behave correctly at runtime.

**JSON syntax — all `.json` files** (requires `jq`):
```bash
jq . {file}.json
```
Run against every generated JSON file: `host.json`, `common_representation.json`, `permissions/{app}.json`, `roles/{app}.json`. A non-zero exit code means the file is malformed JSON. Fix before reporting results.

**permissions.json structural validation** (requires `python3` + `jsonschema` + `rbac_config_path`):
```bash
python3 -c "
import json, jsonschema
schema = json.load(open('{rbac_config_path}/schemas/permissions.schema'))
data   = json.load(open('{output_dir}/rbac-config/permissions/{app}.json'))
jsonschema.validate(data, schema)
print('OK')
"
```
Run for each generated permissions file. Validates structure against rbac-config's own JSON Schema.

**roles.json structural validation** (requires `python3` + `jsonschema` + `rbac_config_path`):
```bash
python3 -c "
import json, jsonschema
schema = json.load(open('{rbac_config_path}/schemas/roles.schema'))
data   = json.load(open('{output_dir}/rbac-config/roles/{app}.json'))
jsonschema.validate(data, schema)
print('OK')
"
```

**inventory-api JSON Schema meta-validation** — legacy path only (requires `python3` + `jsonschema`):
```bash
python3 -c "
import json, jsonschema
for path in ['{output_dir}/inventory-api/{type}/common_representation.json',
             '{output_dir}/inventory-api/{type}/reporters/{reporter}/{type}.json']:
    jsonschema.Draft7Validator.check_schema(json.load(open(path)))
    print(f'OK: {path}')
"
```
Confirms each inventory-api schema file is a valid JSON Schema draft-07 document (not that the schema content is semantically correct — that requires runtime validation via `go run main.go preload-schema`).

**Starlark compile check** — unified path only (requires `go` + `unified_schema_path`):

```bash
# Work on a scratch copy so the real repo is never modified
cp -r {unified_schema_path} /tmp/unified-schema-check
cp {output_dir}/starlark-unified-schema/schema/{namespace}.star /tmp/unified-schema-check/schema/
cd /tmp/unified-schema-check
make build-interpreter
JSONSCHEMA_OUTPUT_DIR=output/jsonschema KSL_OUTPUT_DIR=output/ksl \
  ./bin/interpreter {namespace}.star
test -s output/ksl/{namespace}.json && echo OK
```

Invoking `./bin/interpreter` with the file list directly is equivalent to `make build-shipped-schema` without having to edit `SHIPPED_SCHEMA_FILES` in the scratch copy; the argument is relative to `schema/`. A non-zero exit means a Starlark syntax error, a bad `load()` path, or a wrapper called with the wrong arity — include the interpreter output in the Step 9 report.

**The `test -s` is not optional.** A `.star` file that compiles but emits nothing (e.g. every call deduplicated away, or a reporter that does not match the file name) still exits 0. Confirm `output/ksl/{namespace}.json` exists, is non-empty, and lists one `extension_references` entry per permission you declared. If the reporter was wrong, the references land in a differently-named file — `ls output/ksl/` to catch that.

This check cannot verify that the referenced KSL extensions exist; that happens in rbac-config's SpiceDB validation. Note that limitation alongside the result.

**KSL syntax check** — legacy path only (requires `ksl` binary + `rbac_config_path`):
```bash
# Compile generated KSL alongside all existing KSL files and the v1 permissions JSON
$(GOBIN)/ksl -o /tmp/draft-{namespace}-schema.zed \
  {rbac_config_path}/configs/stage/schemas/src/*.ksl \
  {output_dir}/rbac-config/schemas/src/{namespace}.ksl \
  {rbac_config_path}/configs/stage/schemas/src/rbac_v1_permissions.json
```
This compiles the draft alongside the full real schema set so import resolution works. A non-zero exit means a KSL syntax or type error — include the compiler output in the Step 9 report.

Note: this command reads from but does not modify the real rbac-config repo. The generated `.ksl` is passed as an additional input file.

**After running all available checks**, set a validation status for each artifact type:
- ✅ **Validated** — check ran and passed
- ⚠️ **Partially validated** — some checks ran (e.g. JSON valid but structural schema check skipped due to missing tool)
- ❌ **Failed** — check ran and found errors (include error output; do not proceed to Step 9 without flagging)
- ⏭️ **Unvalidated** — required tool not available

### Step 9 — Present output and review

Show the EM/tech lead:

1. The schema source verdict from Step 1.5 (`unified` or `ksl`) with its one-line reason, then the full content of the generated `{namespace}.star` or `{namespace}.ksl`
2. Summary table of resource schemas generated — on the unified path, state "none — permissions-only service" instead
3. Permissions and roles overview; on the unified path, also show the two `Makefile.patch` lines, since the PR is incomplete without them
4. **Validation results** — one line per artifact type with its status from Step 8.5; if any check failed, show the error output before anything else
5. Any flags or warnings (e.g. "write verb was not split — confirm whether delete/move are separate operations")
6. **Unresolved decisions tracker (UX: "unresolved decisions invisible" feedback):** Always show a dedicated section after the output summary:

> **⚠️ Unresolved schema decisions — review before opening PRs**
>
> | Item | Current value | Impact |
> |---|---|---|
> | {any TBD/unknown/deferred item} | {value} | {one-line consequence} |
>
> _If none: "✅ No unresolved schema decisions."_

Populate from: any v2 permission name that was proposed but not explicitly confirmed; any reporter field marked "TBD"; any open Q&A question deferred to Phase 2; v1 permissions listed as "unknown — Phase 1 follow-up"; roles with `platform_default` not yet confirmed. This gives the team a clear list of what to revisit before filing PRs, without having to re-read all generated files.

Then ask: **"Would you like to proceed with migrating the service's v1 RBAC call sites to Kessel v2 code now?"**

- **Migrate now** — invoke `skills/onboarding-migrate-rbac-v1/SKILL.md` with `--profile {profile_path} --rbac-config {rbac_config_path}` and the service codebase; add `--unified-schema {unified_schema_path}` when `schema_source = unified`. Pass `{output_dir}` so the migrate skill reads v2 permission names directly from the generated `{namespace}.star` or `{namespace}.ksl`.
- **Migrate later** — write `{output_dir}/migrate-context.md` (see below) and stop.
- **Skip** — no migration needed (e.g. no existing v1 call sites, or migration will be done manually).

#### Migration context file (when "migrate later" is chosen)

Write `{output_dir}/migrate-context.md`:

```markdown
# Migration context: {service.name}

**Generated:** {ISO date}
**ServiceProfile:** {profile_path}
**Schema artifacts:** {output_dir}
**Codebase:** {codebase_ref}
**Schema source:** {unified | ksl} — {one-line Step 1.5 reason}
**rbac-config:** {rbac_config_path or "not provided"}
**inventory-api:** {inventory_api_path or "not provided"}
**starlark-unified-schema:** {unified_schema_path or "not provided"}

## Resume command

/kessel-onboarding:migrate-rbac-v1 \
  --context {output_dir}/migrate-context.md \
  {codebase_ref}

## v2 permission name mapping

{for each v1 permission → v2_perm, read from the generated `{namespace}.star` (5th argument of
add_v1_based_permission, or `{app}_{resource}_{verb}` for add_unified_permission) or from the
generated `{namespace}.ksl`, whichever was produced}

## Patterns applied

{for each asset_type: pattern_id, confidence, rationale}

## Open questions

{any medium/low-confidence items or unresolved write-verb splits}
```

## Outputs

| Output | Path | Path |
|--------|------|------|
| | **unified** | **ksl** |
| Unified Starlark schema | `{output_dir}/starlark-unified-schema/schema/{namespace}.star` | — |
| Makefile edits note | `{output_dir}/starlark-unified-schema/Makefile.patch` | — |
| Resource schemas | — | `{output_dir}/inventory-api/{asset_type}/` (one per asset type) |
| KSL schema | — | `{output_dir}/rbac-config/schemas/src/{namespace}.ksl` |
| Permissions JSON | `{output_dir}/rbac-config/permissions/{app}.json` (one per v1 app) | same |
| Roles JSON | `{output_dir}/rbac-config/roles/{app}.json` (one per v1 app) | same |
| README | `{output_dir}/README.md` | same |
| Migration context (when "migrate later") | `{output_dir}/migrate-context.md` | same |

Return all paths to the orchestrating agent or user.

After presenting outputs and Gate 2 options, if the user is not immediately proceeding to migrate or another skill, read `context/implementation-topics.json` and offer 3–5 relevant follow-up implementation topics (see `AGENTS.md` for the selection and presentation pattern).

## MCP policy

| Tool | Allowed |
|------|---------|
| `create_issue` / `update_issue` | **No** |
| `search_issues` / `get_issue` | No (not needed) |
| File reads (codebase_ref) | Yes |
| File writes (output_dir) | Yes — local artifacts only |

## Codebase analysis signals

When `codebase_ref` is available, look for these to draft Step 2–3 answers:

| Field to draft | Signals to look for |
|---|---|
| Reporter name | Kessel SDK `ClientBuilder` calls, reporter config, app short name |
| Reporter fields | DB model/migration columns, API response schemas, protobuf messages, marshmallow/pydantic schemas |
| Schema namespace | Existing `.ksl` files or `schema/{app}.star` sources, Kessel SDK namespace config, app identifier |
| V2 permission names | `KesselPermission` or equivalent classes, Kessel SDK permission constants |
| Contingent permissions | Cross-service API calls, stacked `@access` decorators, permission dependencies |

## Changelog

- 2026-11: Default to the unified Starlark schema for permissions-only services. New Step 1.5 decides `schema_source` (`unified` vs `ksl`) from the Step 1 classification: a service that needs no `public type` for any asset type now gets a single `schema/{namespace}.star` for starlark-unified-schema instead of a hand-authored `.ksl` plus inventory-api JSON Schema. Added Step 5A (Starlark generation rules and extension call syntax), split the old Step 5 into 5A/5B, gated Step 4 and the inventory-api validation on the legacy path, added a Starlark compile check to Step 8.5, added `unified_schema_path` input, extended the Step 0 existing-namespace check to cover both repos, and branched the README next-steps and testing links. Format details live in `reference.md`.
- 2026-09: Test-day user feedback: Step 9 now shows an unresolved schema decisions tracker after the output summary — surfaces all TBD/unknown/deferred items (unconfirmed v2 names, deferred reporter fields, unknown permissions, unconfirmed platform_default) with one-line impact descriptions so the team knows what to fix before opening PRs.
- 2026-09: Test-day gap fixes: KSL generation now checks for existing namespace in rbac-config and generates only additive content; README must list any asset_types[] entries omitted from generated schemas with explicit rationale; types inferred from codebase but absent from asset_types[] require EM confirmation before schemas are generated; Step 7 checks for existing deployed roles file before scaffolding; Step 2 Q&A explicitly distinguishes reporter payload from domain model, with guidance to check ReportResource call sites; Step 3b checks for existing v2 permission names (SDK constants, feature flags, KSL context) before applying naming convention; Step 1 asks "new type vs workspace extension" when interview language is ambiguous; Step 7 confirms platform_default on viewer role with EM before finalizing; all URL inputs (codebase_ref, rbac_config_path, inventory_api_path) are cloned locally before analysis.
- 2026-09: Updated the follow-up implementation topic guidance to use the shared workflow in `AGENTS.md`.
- 2026-08: Step 9 now offers 3–5 contextually relevant follow-up implementation topics from `context/implementation-topics.json` when the user is not immediately proceeding to migration, so users are guided toward implementation next steps without leaving the conversation.
- 2026-08: Step 9 extended with Gate 2 — after presenting schema output, offers to invoke migration immediately, write a `migrate-context.md` context file for later, or skip. `migrate-context.md` added to Outputs table. Context file carries v2 permission name mapping, applied patterns, and open questions so `onboarding-migrate-rbac-v1` can skip re-derivation.
- 2026-07: Steps 0/3a/3d/5/7/8/8.5 — tool preflight (ksl, jq, jsonschema); multi-namespace KSL consolidation; concrete extension-point question (omit when unsure, new services have no dependents); three KSL generation rules (no redundant top-level declarations, no rbac.ksl-owned permissions, top-level for workspace-level only); roles.json must not use _comment (additionalProperties:false); real testing/validation resource links in README; structural validation step after generation.
- 2026-07: Initial version.

Assisted-by: Claude (Anthropic)
