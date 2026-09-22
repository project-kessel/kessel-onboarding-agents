---
name: onboarding-interview-conduct
description: >
  Runs structured Kessel onboarding intake (Phase 0/1): provider, service, Jira
  project, feature epic, wave, tech stack, asset types, UI access checks, v1
  permissions. Outputs ServiceProfile JSON and narrative summary. Used by
  onboarding-interview agent.
---

# Onboarding interview conduct

Structured Phase 0/1 intake with the service EM and tech lead. Produces a draft `ServiceProfile` for pattern suggestion and dedup.

## When to use

- At the start of every new Kessel service onboarding — this is the first skill the `onboarding-interview` agent runs via `/kessel-onboarding:interview`.
- For both live Q&A sessions (EM/tech lead present) and headless sessions where intake notes or a repo reference are supplied up front.

## Configuration

Read `~/.config/kessel-onboarding/config.json`. Use `artifacts_dir` for output paths. If missing, direct the user to [docs/configuration.md](../../docs/configuration.md).

## Inputs

| Input | Required | Notes |
|-------|----------|-------|
| `service_name` | yes | From command or positional arg |
| `provider_name` | no | Infer or ask |
| `feature_epic_key` | no | `--feature-epic` flag |
| `intake_notes` | no | Pasted doc or file for `--headless` |
| `codebase_ref` | no | GitHub/GitLab URL, local path, or archive to the service's repo — see [Step 1.5](#step-15--codebase-reference-optional) |
| `headless` | no | Skip live Q&A |
| `test_mode` | no | When `true`, activates the Kessel blindfold during codebase analysis and routes all artifacts to `{artifacts_dir}/test/{slug}/profiles/` instead of `{artifacts_dir}/profiles/` |

## Execution

### Step 0 — Load reference

Read [reference.md](reference.md) for schema and question script. Optionally read [examples/activation-keys-profile.json](../../examples/activation-keys-profile.json) for shape.

### Step 1 — Session setup (Gate 0)

Confirm with EM:

- Single service intake vs provider-wide context
- EM is present or explicitly delegated
- Home Jira project

**Rule:** Do not infer or default intake type or EM presence. These determine session scope and must be explicitly confirmed before proceeding to Step 2. If the user skips or does not answer, re-ask — do not assume.

If `--headless` with `intake_notes`, skip to Step 3 after parsing notes.

**Upfront orientation — show before Group 1 (UX: "going in blind" feedback):** After session setup is confirmed, briefly orient the EM before asking any questions:

> "This intake covers **7 question groups** across two phases. I'll ask one question at a time and wait for your answer before moving on:
>
> **Phase 0 (4 groups):** Jira linkage · Contacts · Kickoff & credentials · UI access checks
> **Phase 1 (3 groups):** Tech stack & asset types · Permissions & inventory · Ephemeral & targets
>
> Where I've analysed your repo, I'll show my draft and ask you to confirm or correct it. Any TBD or unknown answers are fine — I'll flag them all at the end.
>
> Ready? I'll start with the first question."

This sets expectations for length and the confirm-or-correct pattern without surprising users mid-session.

### Step 1.5 — Codebase reference (optional)

Ask once, before Group 1: "Do you have a link to the service's repo (GitHub or GitLab URL), a local path, or an archive I can look at? Pointing me at the code lets me draft answers for UI access checks, tech stack, asset types, and v1 permissions, which you then confirm or correct rather than dictating from scratch."

- If the EM/tech lead has nothing to share, record `codebase_ref` as none and proceed to Group 1 with no analysis — this step never blocks the interview.
- If provided, record it as `interview.codebase_ref` and attempt analysis with whatever tools are available in this session. **If `codebase_ref` is a GitHub or GitLab URL, clone the repository to a local temp directory (`work/codebase/`) before reading any files — do not fetch individual files via URL.** Use the local clone path for all codebase reads. If cloning or access fails — private repo, no matching tool, unreadable archive — say so and fall back to asking the affected groups cold; do not stall the session waiting for access.
- **Never invent file contents or guess at a repo's structure.** Only draft an inferred value for a field when the agent actually read a matching file; otherwise leave that field unknown and ask the normal question in Step 2.

**What to look for, by field:**

| Field | Signals to check |
|-------|-------------------|
| `tech_stack.lang` / `framework` | Manifest files: `pom.xml`/`build.gradle` (Java), `package.json` (Node), `go.mod` (Go), `requirements.txt`/`pyproject.toml` (Python), `Gemfile` (Ruby) |
| `tech_stack.auth` | Check for **two distinct SDK uses**: (1) auth/permission SDK — `Check`, `CheckForUpdate`, `ListAllowedWorkspaces`, Kessel SDK auth imports used for access control; (2) inventory SDK — `ReportResource`, `DeleteResource`, inventory client imports used for resource reporting. If only inventory SDK is present with no auth integration, record: "No auth integration; Kessel inventory SDK present for resource reporting." Do not conflate the two.|
| `ui_access_checks` | Frontend source calling `/rbac/v1/access` (or equivalent) — grep for the literal path |
| `asset_types[]` | Domain/model classes, DB migrations, or API resource names that look like customer-managed resources |
| `v1_permissions` | `rbac-config`-style permission definition files (e.g. `permissions.yaml`, access-config manifests) |

For each field with a match, draft the value with a one-line rationale citing what was found (e.g. "`pom.xml` has Quarkus + a `kessel-sdk` dependency → lang=Java, framework=Quarkus, auth=Kessel SDK"). Carry these drafts into Step 2 — see the confirmation rule below.

#### Test mode: codebase analysis rules

When `--test-mode` is active, apply a **Kessel blindfold** during codebase analysis. The goal is to simulate running this interview against a service that has not yet integrated with Kessel, even if the actual codebase already has.

**Ignore entirely — pretend these do not exist:**

| What to ignore | Examples |
|---|---|
| Kessel SDK dependency | `kessel-sdk`, `kessel-sdk[auth]`, `github.com/project-kessel/inventory-client-go` in any manifest |
| Kessel client code | `lib/kessel.py`, `internal/kessel/`, any file initialising `ClientBuilder` or `Kessel{}` |
| Kessel permission classes | `KesselPermission`, `KesselResourceType`, `KesselResourceTypes`, `HostKesselResourceType`, etc. |
| Kessel feature flags | `FLAG_RBAC_WORKSPACES`, `bypass_kessel`, `BYPASS_KESSEL`, `kessel_auth_enabled`, any flag gating Kessel behaviour |
| Existing Kessel Check/List calls | `Check(`, `CheckBulk(`, `CheckForUpdate(`, `ListAllowedWorkspaces(`, `list_workspaces(` via Kessel SDK |
| Inventory reporting calls | `ReportResource(`, `report_resource(`, outbox publishers sending to Kessel inventory |
| `@access` decorators backed by Kessel | Decorators that reference `KesselResourceTypes.*` |
| Kessel SDK workspace calls | `list_workspaces(` from `kessel.rbac.v2` — Kessel SDK method, not the RBAC service REST API |

**Still analyse — these reflect the pre-Kessel state:**

Every service onboarding to Kessel will almost certainly already be integrated with the RBAC v1 service. That integration code is exactly the migration surface — read all of it.

| What to keep | Why |
|---|---|
| Language, framework (minus Kessel SDK) | Tech stack is independent of Kessel integration |
| Domain model classes, DB migrations | These determine what asset types exist |
| **All RBAC service integration code (v1 and v2)** | Services use both RBAC v1 and v2 REST APIs — read both fully, not just constants |
| v1 RBAC permission enums/constants | `RbacPermission`, `RbacResourceType`, `inventory:hosts:read` |
| v1 RBAC middleware and permission-checking logic | Full middleware file (e.g. `lib/middleware.py`, `internal/rbac/`) — read the entire RBAC v1 enforcement flow |
| v1 RBAC client calls | `get_rbac_filter()`, `get_rbac_url()`, `rbac_permission_denied()`, any function calling `/api/rbac/v1/access/` |
| v2 RBAC REST API calls | `get_rbac_workspace_by_id()`, `get_rbac_workspaces()`, `RBAC_V2_ROUTE`, any function calling `/api/rbac/v2/workspaces/` — the RBAC service's own workspace API. Many services call this directly to look up workspace IDs; it is RBAC service integration, not Kessel. Reveals whether groups/workspaces are already a concept in the service. |
| Permission-checking decorators (RBAC-backed) | `@rbac_permission`, `@permission_required`, or any decorator that calls the RBAC service (v1 or v2) — not Kessel-backed ones |
| Existing rbac-config permissions/roles files for this service | `permissions/{app}.json`, `roles/{app}.json` in rbac-config — these are the permission definitions being migrated |
| `/rbac/v1/access` route calls | Determines `ui_access_checks` value and permission surface |
| API endpoints and handlers | Reveals the permission surface |
| ClowdApp config, IQE test plugin | Determines ephemeral/Bonfire answer |
| CMDB/CIAM credential config | Service account status (env vars for auth, not Kessel-specific) |

**Distinguishing RBAC service v2 from Kessel:** `/api/rbac/v2/workspaces/` is the RBAC service's own workspace REST API — keep it. `kessel.rbac.v2.list_workspaces()` is a Kessel SDK method that happens to call a similar endpoint via Kessel — ignore it. If in doubt, check the import: RBAC service calls use `requests` or `httpx`; Kessel SDK calls use the `kessel.*` package.

When a file contains both RBAC service logic and Kessel integration code (e.g. `lib/middleware.py`), read the entire RBAC service section (v1 and v2) and skip only the Kessel-specific sections. Do not skip the file.

Annotate drafted values with `(test mode — derived from pre-Kessel signals)` in the summary so the validation skill knows what was inferred vs asked cold.

**Wave inference for already-integrated services in test mode:** Even in test mode, the _presence_ (not content) of certain integration signals is visible and indicates the service is already beyond wave 2. If the codebase shows **all three** of: (a) Kessel SDK imports in any manifest (even if blindfolded from their usage), (b) feature flags with names matching `*rbac-v2*`, `*bypass_kessel*`, or similar gating patterns, and (c) production-grade OAuth2/OIDC credential configuration (not just env var stubs), default `program.wave` to `1` rather than `2`. Note the inference in the summary as `(test mode — wave 1 inferred from pre-Kessel integration signals)` and ask the EM to confirm.

### Step 2 — Interview (one fixed group per turn)

Ask fields one at a time, in the fixed group order below. **Send exactly one question per message and wait for the EM's answer before sending the next.** Never combine two questions into a single message — not within a group, and not across groups. This applies on both Claude and Codex, which cannot present per-question response inputs; a single-question-per-turn flow is the only format that works reliably across both platforms.

**After completing each group: show a brief progress line before moving on (UX: "no sense of progress" feedback).** After the last question in a group is answered, show:

> `✓ Group N/7 done — [N groups remaining] | TBD this group: [list fields, or "none"]`

After Group 4 (last Phase 0 group), add a phase checkpoint:

> `— Phase 0 complete. Starting Phase 1 (3 groups). —`

| Group | Fields | Phase |
|-------|--------|-------|
| 1. Jira linkage | `jira.feature_epic_key`, `program.wave` | 0 |
| 2. Contacts | `contacts.pm`, `contacts.em`, `contacts.tech_lead` | 0 |
| 3. Kickoff/credentials | `kickoff.docs_reviewed`, `credentials.cmdb_registered`, `credentials.service_account_status` | 0 |
| 4. UI access | `ui_access_checks` | 0 |
| 5. Tech stack & assets | `tech_stack.lang`/`framework`/`auth`, `asset_types` | 1 |
| 6. Permissions & inventory | `v1_permissions`, `inventory_reporting`, `inventory_migration_required` | 1 |
| 7. Ephemeral & targets | `ephemeral.uses_bonfire` (+ `ephemeral.pre_dev_tooling` only if false), `targets.stage`/`targets.prod` | 1 |

Skip a group entirely if every field in it is already known (from provider context, a prior service in the same multi-service session, or headless notes). In multi-service provider sessions, `jira.home_project` and provider-level fields (`provider.name`) are captured once during provider context and are not re-asked per service.

**Repo-drafted fields (groups 4, 5, 6):** if Step 1.5 produced a draft for a field in that group's turn, present the draft and its rationale in place of the raw question and ask the EM/tech lead to confirm or correct it — do not skip the group or auto-accept the draft. Record whatever they confirm (which may differ from the draft) as the final value, and mark that field `(confirmed from repo analysis)` in the narrative summary. A field with no draft in an otherwise-drafted group is still asked normally.

**Group 2 — contacts duplicate email check):** After recording contacts, if `contacts.em.email == contacts.tech_lead.email`, ask: "The EM and tech lead appear to share the same email address. Is there a separate technical lead we should contact for schema and implementation questions?" Record the clarification before moving to the next group.

**Group 5 mandatory follow-up — asset type ownership:**

**Rule: this question is required for every confirmed asset type. Do not skip it, even if the answer seems obvious from codebase analysis.**

After the EM confirms `asset_types[]`, ask for **each type individually**:

> "Is `{asset_type}` a new Kessel resource type your service will define and own, or does it correspond to an existing Kessel platform type — specifically `rbac.workspace` (e.g. groups, namespaces, projects that ARE the workspace hierarchy rather than living within it)?"

This catches the common case where an asset type derived from a v1 RBAC resource (e.g. `RbacResourceType.GROUPS`) maps to the existing `rbac.workspace` Kessel type rather than requiring a new `public type` definition. If this is not asked, schema-design will generate a `public type group` that shouldn't exist — and the mismatch will propagate through the KSL, inventory-api schema, and roles files.

If the EM confirms an asset type maps to `rbac.workspace`:

- Record it in the **narrative summary** (e.g. "group — maps to rbac.workspace; no new KSL type needed; permissions handled via rbac.ksl workspace type").
- **Annotate the pattern rationale** in `patterns[]` for that asset type with "uses existing rbac.workspace type — schema-design must NOT generate a public type definition". Schema-design Step 1 reads `patterns[]` as its authoritative source.
- If the previously suggested pattern (e.g. `native`) is inconsistent with the EM's answer, flag the inconsistency immediately and resolve it in this group's turn — do not carry an unresolved contradiction into suggest-patterns.

**Important scope limit:** This follow-up identifies whether an asset type uses an existing platform type. It does **not** independently determine whether the schema-design skill generates a KSL type definition or inventory-api schema — those decisions are made solely by schema-design Step 1 using `patterns[]` and `inventory_migration_required`. "Service-owned" alone does not imply a KSL `public type`; that depends on whether the pattern is `native` or `native-ws-list` specifically.

If codebase analysis in Step 1.5 found a `KesselResourceType` subclass with `namespace="rbac"` for an asset type, draft the answer as "maps to rbac.workspace" and ask the EM to confirm.

**Group 6 — wildcard resource permissions:** When presenting codebase-drafted `v1_permissions`, always include wildcard resource entries (`{app}:*:read`, `{app}:*:write`, `{app}:*:*`) if found in the analysis — do not filter them out as noise. These are meaningful grants (cross-resource-type access) that appear in `permissions.json` and are distinct from resource-specific wildcards like `{app}:{resource}:*`. If the codebase analysis found them, include them in the draft list. Explicitly prompt: "Does your app have any permissions that span all resource types — like `{app}:*:read` or `{app}:*:*`? These are often admin bypass grants."

**Group 6 — inventory reporting code-readiness:** When asking about `inventory_reporting` and `inventory_migration_required`, distinguish between two states: "Does your service currently report resources to Kessel Inventory in production?" (`inventory_reporting` field) AND "Does your service already have inventory reporting code written (e.g. ReportResource calls), even if it isn't active in production yet?" Record both answers. If reporting code exists but is not yet active, note this in the summary (e.g. "Inventory SDK wired but not yet active in production — migration still required") so the schema-design skill understands the service is code-ready.

**Group 6 — feature flag / dual-path strategy:** For services with existing RBAC v1 integration, ask: "Does your service use a feature flag to gate the Kessel v2 authorization path? If so: what is the flag name, what is its default state (on/off), and is there a bypass or fallback to the v1 path?" Record the flag name and bypass mechanism in `docs_gaps[]` if not otherwise captured. This prevents the feature flag strategy from being a consistent miss in schema-design and migration planning.

Rules:

- Infer `service.slug` from `service.name` (kebab-case).
- Default `jira.label` to `kessel-onboarding`.
- Default `program.path` to `self-service` (suggest-patterns may override).
- Set `interview.conducted_at` to current ISO-8601 timestamp.
- Record `interview.participants` from names provided.
- Leave `patterns`, `platform_gates`, and `dedup` empty or omit until downstream skills run.

Whenever the EM or tech lead asks a question the agent cannot answer from KesselDocs or the internal docs, record it in `docs_gaps[]` with enough context to reproduce the question. Do not interrupt the interview to research; capture and move on.

### Step 3 — Build ServiceProfile JSON

Assemble object per schema v1.3 (accept 1.0, 1.1, 1.2, or 1.3; new fields absent on earlier-version profiles are treated as null). Validate:

- `ui_access_checks` is one of `required`, `new`, `not_required`, `n/a`
- `program.wave` is 1, 2, or 3
- `v1_permissions.items` is non-empty array (or explicit "unknown — Phase 1 follow-up" in summary)

### Step 4 — Write artifacts

Output paths depend on `test_mode`:

| `test_mode` | Profile JSON | Narrative summary |
|---|---|---|
| `false` (default) | `{artifacts_dir}/profiles/{slug}-profile.json` | `{artifacts_dir}/profiles/{slug}-summary.md` |
| `true` | `{artifacts_dir}/test/{slug}/profiles/{slug}-profile.json` | `{artifacts_dir}/test/{slug}/profiles/{slug}-summary.md` |

Create the target directory if it does not exist.

Summary format: see [reference.md](reference.md#narrative-summary-template).

### Step 5 — Present draft

Before requesting EM approval, render the **full profile summary table** in the conversation/chat. Include every captured ServiceProfile field, including unknown or `TBD` values, codebase-derived rationales, and any gaps flagged for Phase 1 follow-up. Writing the narrative summary file or returning its path is not a substitute for displaying the table. Do **not** run suggest-patterns or dedup here — the agent orchestrator calls those next.

**Unresolved decisions tracker — always show at end of Step 5 (UX: "unresolved decisions invisible" feedback):** After the full profile table, render a dedicated section:

> **⚠️ Unresolved decisions — review before provisioning**
>
> | Field | Value | Impact if left unresolved |
> |---|---|---|
> | {field with TBD/unknown/null/none} | {current value} | {one-line consequence} |
>
> _If all fields are resolved, show: "✅ No unresolved decisions — profile is complete."_

Populate this table from every field where the captured value is `null`, `"TBD"`, `"unknown"`, `"none"`, or an explicit "Phase 1 follow-up" note. Include a one-line consequence for each — e.g. "`jira.feature_epic_key = null` — blocks Jira provisioning until a feature epic key is supplied"; "`credentials.service_account_status = none` — Phase 3 blocked until CIAM service account is requested"; "`v1_permissions.items = unknown` — schema-design cannot generate KSL until permissions are identified." This gives the EM a clear action list rather than requiring them to comb through the profile for gaps.

## Headless mode

When `--headless` is set:

1. Parse `intake_notes` or attached file into profile fields.
2. If `codebase_ref` is also passed, run the Step 1.5 analysis and merge drafted values into the parsed notes for any field the notes left blank; since there is no live EM/tech lead to confirm, mark those fields `"TBD — confirm with EM (drafted from repo analysis)"` in summary rather than treating the draft as final.
3. Mark missing required fields as `"TBD — confirm with EM"` in summary only; use null-safe defaults in JSON where schema allows.
4. Write artifacts and return paths.

## Outputs

| Output | Path |
|--------|------|
| ServiceProfile JSON | `{artifacts_dir}/profiles/{slug}-profile.json` |
| Narrative summary | `{artifacts_dir}/profiles/{slug}-summary.md` |

Return both paths to the orchestrating agent.

## Changelog

- 2026-09: Test-day user feedback: Step 2 changed from "one group per turn" to "one question per turn" — each field is its own prompt, works on both Claude and Codex without requiring numbered-list responses; Step 1 shows an upfront 7-group orientation so users know what to expect; progress line shown after each group completes; Step 5 adds unresolved decisions tracker surfacing all TBD/unknown/null fields with one-line impact descriptions.
- 2026-09: Test-day gap fixes: Group 6 now asks for feature flag name, default state, and v1 fallback for services with existing RBAC v1 integration; Group 6 distinguishes inventory reporting code-readiness from production-active status; tech_stack.auth analysis now separately identifies auth SDK vs inventory SDK usage; test mode infers wave 1 when pre-Kessel signals indicate existing deep integration; Group 2 challenges identical EM and tech lead email; codebase_ref URLs are cloned to a local temp directory before any file reads.
- 2026-09: Gate 1 now requires the complete ServiceProfile summary table to be rendered in the conversation before EM approval, rather than only writing or reporting the artifact path.
- 2026-08: Added `test_mode` to inputs table; updated Step 4 to document conditional artifact routing (`{artifacts_dir}/profiles/` vs `{artifacts_dir}/test/{slug}/profiles/` when `test_mode = true`).
- 2026-08: Elevated Group 5 asset-type ownership question to a mandatory rule (must be asked for every asset type, not skipped); catches types like `group` that map to `rbac.workspace` rather than requiring a new `public type`. Expanded blindfold keep-list to cover all RBAC service integration code (v1 AND v2) with disambiguation rule: RBAC service REST calls (kept) vs Kessel SDK calls (ignored).
- 2026-08: Added Kessel blindfold rules to Step 1.5 for `--test-mode` — ignores existing Kessel SDK, permission classes, and client code; derives answers from pre-Kessel signals only; annotates drafted values with `(test mode)` marker for the validation skill.
- 2026-07: Group 5 — ask whether each asset type is service-owned or maps to an existing `rbac.workspace` type; passes ownership context to schema-design. Group 6 — always include wildcard resource permissions (`{app}:*:read`, `{app}:*:*`) in codebase-drafted permission lists.
- 2026-07: Added optional `codebase_ref` input and Step 1.5 for codebase-analysis-driven drafts (schema v1.3). Added `new` ui_access_checks value. Fixed question order to one group per turn.
- 2026-07: Initial version — accept schema v1.0/1.1/1.2; fixed 7-group interview order; docs-gap capture rule.

Assisted-by: Claude (Anthropic)
