---
name: onboarding-migrate-rbac-v1
description: "Migrate an HCC service's RBAC v1 call sites to Kessel v2. Finds v1 RBAC call sites in the service codebase, classifies each against the KSL-016 adoption patterns, fills schema gaps, and writes Kessel v2 replacement code for review. Use after the onboarding interview and schema-design skills, or standalone via /kessel-onboarding:migrate-rbac-v1."
disable-model-invocation: false
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Agent
  - AskUserQuestion
  - Skill
argument-hint: "--rbac <path> --rbac-config <path> [--unified-schema <path>] [--inventory-api <path>] [--profile <path>] <service-repo>"
---

# Migrate RBAC v1 Call Sites to Kessel v2

This skill turns a v1-only or partially-migrated service into one that
authorizes via Kessel. It finds every v1 call site, works out the
correct v2 adoption pattern, and writes the actual replacement code for
review — not just a report.

**Self-contained by design.** Everything needed to run this skill —
the KSL-016 pattern catalog, code templates, and a minimal schema
scaffold — is embedded in [reference.md](reference.md) or linked to a
stable public/canonical URL. It does not assume any other skill,
plugin, or agent is installed. If you (or a teammate you hand this
skill to) *do* have the `kessel-onboarding` plugin available, this
skill uses it opportunistically for a better schema-design experience
(Phase 3) — but it works fully without it.

It does not create Jira tickets or PRs. Changes are written to the
working tree, uncommitted, so the user reviews via `git diff` and hands
off to whatever branch/PR workflow they normally use.

Read [reference.md](reference.md) in full before Phase 2 — it has the
pattern catalog, language-agnostic call shapes, the minimal schema
scaffold, and a gotchas checklist. reference.md is deliberately
service- and language-independent: it points to the official Kessel
SDKs and APIs as the source of truth rather than hardcoding any one
service's implementation.

## Arguments

Parse `$ARGUMENTS` for:

- `--rbac <path-or-url>` (required) — local path or git URL to
  `insights-rbac`. Reference for scope config and workspace lookup
  APIs (`/v2/workspaces?type=default|root`).
- `--rbac-config <path-or-url>` (required unless `--profile` already
  supplies v2 permission names for every target) — local path or git
  URL to `rbac-config`. Used to check whether v2 permissions already
  exist for this service (in `.ksl` or released KSIL `.json`), and for
  the `permissions.json` / `roles.json` half of schema generation.
- `--unified-schema <path-or-url>` (optional) — local path or git URL to
  [`starlark-unified-schema`](https://github.com/project-kessel/starlark-unified-schema),
  the current-generation schema source. Checked alongside `--rbac-config`
  in Phase 3 for an existing v2 permission mapping, and used as the
  target shape when Phase 3 must generate a permissions-only schema.
  If omitted, Phase 3 still checks `~/dev/starlark-unified-schema` and
  `../starlark-unified-schema` before falling back to the reference
  scaffold.
- `--inventory-api <path-or-url>` (optional) — local path or git URL to
  `inventory-api`. Only needed if Phase 3 has to generate resource
  schemas (native / native-ws-list / default-workspace patterns).

**URL inputs — clone before reading:** For each argument that accepts a path-or-url (`--rbac`, `--rbac-config`, `--unified-schema`, `--inventory-api`, and the positional `<service-repo>`): if the value is a GitHub or GitLab URL, clone the repository to a local temp directory (`work/rbac/`, `work/rbac-config/`, `work/unified-schema/`, `work/inventory-api/`, `work/codebase/`) before any analysis. Do **not** fetch individual files via URL. Use the local clone path for all subsequent reads. If cloning fails, stop and report the error — do not fall back to URL fetching.
- `--profile <path>` (optional) — path to a ServiceProfile JSON. See
  "Bridging from an onboarding profile" below.
- `--context <path>` (optional) — path to a `migrate-context.md` file
  written by the schema-design skill's Gate 2 "migrate later" option.
  When present, takes precedence over `--profile` and pre-populates
  `--rbac-config`, `--inventory-api`, the v2 permission name mapping,
  and the pattern classification — skipping all re-derivation. See
  "Bridging from a migration context file" below.
- One positional argument — the service codebase to migrate. May be omitted
  if `--context` already records `codebase_ref`.

If required arguments are missing, prompt with `AskUserQuestion`.

## Bridging from an onboarding profile

If a service has already been through an onboarding interview and/or
schema-design process (whether via the `kessel-onboarding` plugin or a
manual equivalent), it may have a **ServiceProfile JSON** describing
its asset types, v1 permissions, and chosen adoption patterns. This
skill can start from that instead of re-deriving everything from
scratch — it's the bridge from "onboarding decided what to do" to
"here's the actual code."

If `--profile <path>` is given, read it. Accept any JSON with (at
least) this shape — this is a minimal, tool-agnostic contract, not tied
to any specific plugin's internal schema:

```json
{
  "asset_types": ["activation_key"],
  "v1_permissions": {
    "items": [
      { "app": "content-sources", "resource": "activation_keys", "verb": "read" }
    ]
  },
  "patterns": [
    {
      "id": "default-workspace",
      "asset_types": ["activation_key"],
      "confidence": "high",
      "rationale": "..."
    }
  ],
  "interview": { "codebase_ref": "path-or-url-to-service-repo" }
}
```

When present:
- Phase 1 still scans the repo (code is the source of truth for what's
  actually there), but **cross-checks** findings against
  `v1_permissions.items` instead of starting blind — flag anything the
  profile lists that the scan doesn't find (may be already migrated or
  removed) and anything the scan finds that the profile doesn't list
  (drift — profile is stale, or scan needs a wider search pattern).
- Phase 2 uses `patterns[]` directly instead of re-running the KSL-016
  heuristics, but still validates each against the decision tree in
  reference.md and flags mismatches rather than silently trusting a
  stale or low-confidence entry.
- If the profile's `confidence` for a pattern is `low`, treat it as
  unclassified for Phase 4 purposes — ask, don't assume.

If no `--profile` is given, none of the above changes — the skill
runs Phases 1–2 entirely from repo discovery, as documented below.

## Bridging from a migration context file

If `--context <path>` is given, read the `migrate-context.md` file written
by the schema-design skill. This is the highest-fidelity starting point —
it contains the v2 permission name mapping already extracted from the
generated KSL, the pattern classification, all repo paths, and any open
questions flagged during schema design.

When present:
- Extract `--rbac-config`, `--unified-schema`, `--inventory-api`,
  `codebase_ref` from the context file; command-line flags override these if
  also supplied.
- Read the context's **Schema source** line (`unified` or `ksl`). It records
  which generation schema-design targeted; carry it into Phase 3 rather than
  re-deciding, and report it in Phase 5. An older context file written before
  this field existed implies `ksl` — re-run the Phase 3 step 3a check rather
  than assuming.
- Phase 1 still scans the repo (code is the source of truth), but uses the
  context's v2 permission mapping to skip Phase 3 schema lookup — the
  mappings are read directly from the context's "v2 permission name mapping"
  section.
- Phase 2 uses the context's "Patterns applied" section instead of
  re-running the decision tree. Still validate high-confidence entries;
  flag medium/low for confirmation.
- Phase 4 uses the exact v2_perm names from the context — never re-derives
  them from rbac-config when the context already has them.
- Surface any "Open questions" from the context file at the start of Phase 4
  and resolve them before writing code.

`--context` takes precedence over `--profile` when both are supplied.
If neither is given, the skill runs fully from repo discovery.

## Setup

URL inputs are cloned in the Arguments section above — use those resolved local paths (`work/rbac/`, `work/rbac-config/`, `work/unified-schema/`, `work/inventory-api/`, `work/codebase/`) for all repo reads. For local path inputs, verify the path exists. Record all resolved paths before proceeding.

Generated artifacts (schemas, migration report) go to `/tmp/migrate-v1-rbac/{service-name}/` — do not write generated files to the cloned repo directories.

## Phase 1: Discover v1 Call Sites and Kessel Gate

**1a. Kessel enablement gate.** Search the service codebase (excluding
tests, vendor, node_modules, `.git`) case-insensitively for:

```
KESSEL_ENABLED
KESSEL_URL
kessel
FLAG_.*KESSEL
feature_flag.*kessel
```

Read the matching settings/config files. Record: gate condition
(e.g. an `ENABLE_KESSEL` env var), config source (file:line), and
default value when unset. If nothing matches, the service has zero
Kessel integration — note that as the starting state, don't fail.

Before the v1 scan, also search for the source marker
KESSEL_ONBOARDING_PRODUCTION_AUTHZ: true. Record its file and line. This
marker means that nearby RBAC declarations and helpers are an intentional
production authorization surface, even if the implementation is incomplete.

**1b. v1 RBAC call sites.** Search for:

```
/api/rbac/v1/access
rbac.*v1.*access
make_rbac_url
has_rbac_permission
rbac_permission
check_access
x-rh-rbac-psk
RBAC_PSK
x-rh-rbac-client-id
RBAC_CLIENT_ID
make_rbac_request
rbac_request
```

For each match (production code only — exclude tests): file, line,
surrounding function/class, the permission string(s) checked, and the
v1 app namespace (e.g. `ros`, `inventory`).

If `--profile` was given, reconcile against `v1_permissions.items` now
(see "Bridging from an onboarding profile").

**1c. Trace bypass status.** For each v1 call site, walk up the call
chain: is it behind the Phase 1a gate condition?

- Behind the gate (v1 runs only when Kessel disabled) → **fallback
  path**. Leave alone unless the user wants v1 removed entirely once
  migration is verified (ask in Phase 5).
- Not behind the gate → **migration target** for Phase 4.
- Already has a working Kessel branch alongside the v1 branch (e.g.
  `if ENABLE_KESSEL: ... elif ENABLE_RBAC: ...`) → **partial
  migration**. Phase 4 audits the existing Kessel branch for
  correctness against reference.md's patterns rather than writing it
  from scratch — this is where real bugs hide (see reference.md's
  gotchas — e.g. a hardcoded "no, you don't have unrestricted access"
  fallback that never gets set `true`).

### 1d. Declared-but-unimplemented authorization

Every service is treated as production by default. Comments describing code as
a demo or stub are not sufficient evidence to conclude that no change is
needed. If the scan finds v1 permission constants, an RBAC route, a permission
helper, or an approved profile mapping but no executable request or check, do
not treat that finding as equivalent to "the service has no authorization
work." Record it as an incomplete authorization contract, inspect the
resource handlers, and implement v2 checks there when the endpoint mapping is
clear, preserving fail-closed behavior and the approved permission mapping.

Services may make this intent explicit with the source marker
KESSEL_ONBOARDING_PRODUCTION_AUTHZ: true. When present, the marker overrides
demo-only or stub wording in nearby comments: the declared authorization
surface must be migrated or the ambiguity must be surfaced. Never produce a
no-op migration report solely because a marked helper currently returns a
placeholder result.

## Phase 2: Classify Each Permission

For each distinct v1 permission found in Phase 1b
(`{app}:{resource}:{verb}`), determine the v2 adoption pattern using
the decision tree in [reference.md](reference.md#pattern-catalog) — or
use `patterns[]` from `--profile` if provided, per the bridging section
above.

Record one row per permission: `{app}:{resource}:{verb}` → pattern →
confidence (high/medium/low) → rationale. If cardinality or
access-probability data needed to distinguish `native` from
`native-ws-list` is unknown, ask the user — don't guess; this affects
whether a per-resource check is fine or the service needs a
`workspace_id` column and pre-filtering.

## Phase 3: Check for Existing v2 Schema, Fill Gaps

Kessel has two schema generations in play. Check **both** — a service's
permissions may already be declared in either, and the newer one is not
visible from rbac-config alone.

| Generation | Source of truth | Shape |
|---|---|---|
| Current — **unified schema** | `starlark-unified-schema` (`--unified-schema`) | `schema/{app}.star`, Starlark, compiles to KSIL |
| Previous — **KSL** | `rbac-config` (`--rbac-config`) | `configs/{env}/schemas/src/{namespace}.ksl`, hand-authored |

For each v1 app namespace found in Phase 1b:

```
# Previous generation — hand-authored KSL in rbac-config
ls <rbac-config-path>/configs/*/schemas/src/{namespace}.ksl
grep -rn "add_v1_based_permission.*app:'{app}'" <rbac-config-path>/configs/*/schemas/src/

# RBAC registration — needed on both paths
ls <rbac-config-path>/configs/*/permissions/{app}.json
ls <rbac-config-path>/configs/*/roles/{app}.json

# Current generation — Starlark in starlark-unified-schema
ls <unified-schema-path>/schema/{namespace}.star
grep -rn "\"{app}\"" <unified-schema-path>/schema/*.star <unified-schema-path>/schema/*/reporters/*/*.star
```

A `{namespace}.json` in `rbac-config/configs/*/schemas/src/` is generated
KSIL released from `starlark-unified-schema`, not a hand-authored file.
Treat it as evidence the namespace is on the unified path and read the
`.star` source for the mapping; never propose editing it in place.

**If v2 permissions already exist in either generation:** extract the
v1→v2 name mapping and use those exact names in Phase 4 — never invent
new ones when a mapping already exists.

- From a `.ksl` file: the `v2_perm:'...'` argument of
  `add_v1_based_permission`, or `{app}_{resource}_{verb}` for
  `add_unified_permission`; also read `add_v1only_permission` and
  `add_contingent_permission`.
- From a `.star` file: the same extensions, called positionally with the
  reporter prepended —
  `add_v1_based_permission(reporter, app, resource, verb, v2_perm)`, so
  the v2 name is the **fifth** argument;
  `add_unified_permission(reporter, app, resource, verb)` yields
  `{app}_{resource}_{verb}`;
  `add_contingent_permission(reporter, first, second, contingent)` — the
  checkable name is `contingent`, never the `_assigned` intermediate.

Record which generation each mapping came from; Phase 5 reports it.

Before generating or delegating any Phase 3 schema output, or writing any
Phase 4 replacement code, request explicit user approval through the
available user-approval question capability. Summarize the planned files,
call sites, permission mapping, the schema source chosen in step 3a below
(unified Starlark vs legacy KSL) with its reason, and unresolved questions.
Offer exactly these choices:

- **Proceed** — continue to the planned writes.
- **Revise** — return to Phase 2/3 planning, update the proposed approach,
  and ask for approval again before writing.
- **Abort** — stop the migration without performing any writes.

Do not invoke a schema-design capability, generate a Phase 3 scaffold, or
perform a Phase 4 file-writing operation until the user selects **Proceed**.
If the plan or write targets change materially, request approval again before
writing. A **Revise** response returns to planning; an **Abort** response
ends the migration and is reported as stopped. This approval requirement
covers writes to generated artifacts, the service codebase, and the migration
report; no destructive or external write may bypass it.

**If missing for a namespace:**

First decide which generation to generate for, then generate.

**3a. Pick the schema source.** The rule matches the schema-design
skill's Step 1.5 — see
[reference.md](reference.md#choosing-the-schema-source) for the full
form:

- **No Phase 2 pattern for this namespace requires a service-owned
  resource type** (every permission classified `default-workspace`, or
  `native` / `native-ws-list` where the service is not reporting
  resources to inventory) → the service onboards **permissions only**.
  Generate a **unified Starlark** `{namespace}.star`. This is the
  default; it replaces both the `.ksl` and any inventory-api JSON
  Schema.
- **Any permission needs a per-resource check against a type the
  service owns in Kessel** (`native` / `native-ws-list` with resources
  reported to inventory) → generate the legacy pair (`.ksl` +
  inventory-api resource schema).
- **The namespace already exists as `.star`** → stay on the unified
  path, additive, regardless of the above.
- **The namespace already exists as hand-authored `.ksl`** → add to
  that `.ksl`. Do not split one namespace across both generations; KSL
  is not forward-compatible with Starlark extensions.

If the classification is `low` confidence for any permission in the
namespace, do not pick for the user — surface the choice in the
Phase 3 approval gate above.

**3b. Generate.**

1. Check whether a schema-design capability is available in this
   environment — e.g. a skill or plugin whose name contains
   `schema-design` or `onboarding` in the current skills/agents
   listing. If one is available, prefer it for a properly-vetted,
   Q&A-driven result:
   ```
   Skill(skill: "<discovered-skill-name>", args: "--codebase_ref <service-repo-path> --rbac_config_path <rbac-config-path> --unified_schema_path <unified-schema-path>")
   ```
2. **If none is available**, generate a minimal schema yourself using
   the scaffold in [reference.md](reference.md#minimal-schema-scaffold)
   matching the 3a verdict — the Starlark scaffold for permissions-only
   services, the KSL scaffold otherwise. Either produces just enough —
   the right permission declarations plus `permissions.json` and
   `roles.json` — to unblock Phase 4. It is intentionally lighter than
   a dedicated schema-design tool (no reporter-field modeling, no
   cross-service contingent-permission interview) — flag in Phase 5
   that a human should review it against
   [starlark-unified-schema](https://github.com/project-kessel/starlark-unified-schema)
   or [rbac-config](https://github.com/project-kessel/rbac-config)
   before merging.

Either way, write the output to
`/tmp/migrate-v1-rbac/{service-name}/schema/` — never write directly
into the `--rbac-config` or `--unified-schema` checkout.

Once a mapping exists (from either path), re-read it the same way as
the "already exist" case above and continue to Phase 4. The v2
permission names Phase 4 uses are identical either way — the generation
choice changes where the declaration lives, not what the service calls
`Check` with.

## Phase 4: Write the Replacement Code

For each migration target and partial migration from Phase 1c:

1. **Pick the call shape** from
   [reference.md](reference.md#call-shapes) matching the Phase 2
   pattern. The shapes there are language-agnostic pseudocode; realize
   them in the target language using its official Kessel SDK (types,
   imports, transport per reference.md's "Source of truth"). The shape
   is the same in any language: workspace lookup (if needed) → Kessel
   `Check`/`CheckForUpdate` or `StreamedListObjects` → interpret the
   result. Mirror the service's existing config/HTTP idioms; do not
   copy another service's implementation verbatim.

2. **Preserve the existing dual-path structure** if one exists (e.g.
   `ENABLE_RBAC` / `ENABLE_KESSEL` branches). Removing the v1 path
   entirely is a decision the user makes explicitly in Phase 5, not a
   silent side effect of fixing the Kessel path.

3. **Apply the gotchas checklist** in
   [reference.md](reference.md#gotchas) to the new or existing Kessel
   branch. In particular: "unrestricted access" must be detected
   correctly (a root-workspace binding means "can access everything",
   not "zero workspaces returned = deny"), and any special/ungrouped
   resource bucket the v1 path supported must still work.

4. **Write the change to the file directly** (Edit tool) — don't just
   print a diff in chat. Keep each edit minimal and scoped to its call
   site; don't refactor unrelated code.

5. Track, per call site: file:line touched, pattern applied, whether it
   was a new Kessel branch or a fix to an existing one, and any open
   question needing a human decision (e.g. "v1 `write` permission
   covers delete — confirm whether v2 should split this").

Do not attempt to run the service's test suite or fix compile/type
errors beyond what's visibly wrong in the diff — verification with the
user is Phase 5.

## Phase 5: Report and Handoff

Print a summary (under 30 lines) to the conversation:

- Call sites migrated/fixed, by file
- Patterns used, and any low-confidence classifications needing
  confirmation
- Where each v2 permission mapping came from — existing `.star`,
  existing `.ksl`, or generated in Phase 3
- Schema gaps filled in Phase 3: which generation was targeted
  (unified Starlark vs legacy KSL) and why, via which path (discovered
  skill vs. minimal scaffold) — vs. still open. On the unified path,
  state that the PR also needs the `SHIPPED_SCHEMA_FILES` and
  `KSL_RELEASE_FILES` Makefile entries, and that the schema change
  must land before this service's code
- Open questions surfaced in Phase 4
- Whether any v1 fallback paths were left in place intentionally, and
  whether a follow-up pass should remove them once the Kessel path is
  verified in stage

Write the full report to
`/tmp/migrate-v1-rbac/{service-name}/report.md`.

Ask the user (`AskUserQuestion`): review the diff now (`git diff`), or
hand off to their normal branch/PR workflow. Do not commit or push
anything yourself.

After the user responds, follow the user's intent directly. Continue a
requested diff review, normal branch/PR handoff, or another skill; answer an
unrelated question directly; and do not offer implementation topics in those
paths. Otherwise, read `context/implementation-topics.json` and offer 2–5
relevant follow-up implementation topics using the selection and presentation
pattern in `AGENTS.md`: include a natural-language lead-in, tailor the topics
to the service context, and end with a question inviting the user to choose a
topic or ask for something else. Good candidates at this stage:
`parity-testing`, `dual-path`, `testing`, `service-account`.

## Important Notes

- Never modify the `--rbac`, `--rbac-config`, `--unified-schema`, or
  `--inventory-api` repos — all generated schema output goes to
  `/tmp/migrate-v1-rbac/`.
- The schema generation choice (unified Starlark vs legacy KSL) never
  changes the service code. The v2 permission names Phase 4 writes are
  the same either way; only the declaration's home repo differs.
- Never stage, commit, or push in the service codebase — leave changes
  as unstaged working-tree edits so they are visible via `git diff`.
- If a v1 call site's pattern classification is `low` confidence, do
  not write speculative code for it — flag it in the Phase 5 report
  and ask instead.
- Use `Agent` with `subagent_type: Explore` for tracing complex call
  chains in Phase 1c.
- This skill has no hard dependency on any other skill or plugin. The
  Phase 3 "discovered skill" path is a nice-to-have, not a requirement
  — always fall back to reference.md's scaffold.

## Changelog

- 2026-11: Phase 3 now checks both schema generations. Added
  `--unified-schema <path-or-url>` for `starlark-unified-schema`, extraction
  of v2 permission names from positional `.star` extension calls alongside
  the KSL form, and step 3a — when a namespace needs no service-owned
  resource type, gap-filling generates a unified Starlark `{namespace}.star`
  instead of a `.ksl` plus inventory-api JSON Schema. `migrate-context.md`'s
  new `Schema source` line is honoured, and Phase 5 reports which generation
  each mapping came from. Phase 4 output is unchanged — the generation
  choice does not affect the service code.
- 2026-09: Treat services as production by default during authorization
  classification; require migration or surfaced ambiguity when markers or
  mappings exist without executable checks, and honor
  KESSEL_ONBOARDING_PRODUCTION_AUTHZ: true.

- 2026-09: Updated the follow-up implementation topic guidance to use the shared workflow in `AGENTS.md`.
- 2026-08: Phase 5 now offers 3–5 contextually relevant follow-up implementation topics from `context/implementation-topics.json` after the migration report is presented, so users are guided toward next steps (parity testing, dual-path, testing, etc.) without leaving the conversation.
- 2026-08: Added to kessel-onboarding plugin as `/kessel-onboarding:migrate-rbac-v1`. Added `--context <path>` argument accepting the `migrate-context.md` file written by schema-design Gate 2; context takes precedence over `--profile` and pre-loads v2 permission names and patterns to eliminate re-derivation. Added "Bridging from a migration context file" section.

Assisted-by: Claude (Anthropic)
