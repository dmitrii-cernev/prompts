# Agent task: split NAM UAT Helm database configuration

Implement repository configuration changes only. Work in the actual repositories available to you; inspect their instructions and working-tree state before editing. Preserve unrelated user changes.

## Goal and scope

UAT1 and UAT2 will use independent database groups:

| Environment | Logical group | Database placement |
|---|---:|---|
| UAT1 | 1 | Existing loader/discrepancy and other current non-vector responsibilities remain unchanged. |
| UAT1 | 2 | Existing `vectors` shard remains. Other current non-vector responsibilities remain unchanged. |
| UAT1 | 3 | Existing linkage and other current non-vector responsibilities remain unchanged. |
| UAT2 | 4 | `data_loader`, `discrepancy`, `amc_gfcid`, and one `vectors` shard. |
| UAT2 | 5 | `id_linkage`, `id_mapping`,  `id_history` (including schema `linkage_service_history`),and one `vectors` shard. |

“Group N” means the existing NAM UAT logical connection endpoint for that group. Resolve the actual endpoint strings from existing UAT1 values, especially the query-router application and migration repositories, rather than inventing addresses.

This task changes values files, not live databases. Creating configuration does not create schemas, move existing records, or make a live shard removal safe. Do not deploy, execute migrations, clear data, change LightSpeed manifests, commit or push as part of this task.

## File selection

Use the existing repository convention:

- existing: `helm/nam-uat-1-values.yaml`;
- new sibling: `helm/nam-uat-2-values.yaml`.

Use the actual checked-out naming convention. If a repository instead contains literal `nam-uat-1.yaml`, create its matching `nam-uat-2.yaml`. Search for all matching UAT1 values files in each target repository, including additional directories, and process every applicable pair. Avoid generated/build/vendor files. Do not create duplicate naming variants for the same chart.

Only repositories containing an actual matching UAT1 file are eligible. Report missing repositories or missing UAT1 files rather than manufacturing configurations for them. If a UAT2 counterpart already exists, reconcile it carefully instead of overwriting unrelated environment-specific settings.

## Target repositories

Organization: `CitiInternalLS1`.

Repository prefix: `178774.wmt-globalcusth.`. The table gives the exact repository suffix; concatenate the prefix and suffix to obtain the full repository name.

Process these 20 repositories, subject to the UAT1-file eligibility check:

| # | Repository suffix |
|---:|---|
| 1 | `customer-search-query-router` |
| 2 | `customer-search-query-router-db` |
| 3 | `customer-search-amc-gfcid` |
| 4 | `customer-search-discrepancy-job` |
| 5 | `customer-search-ids-linkage-db` |
| 6 | `customer-search-data-loader-db` |
| 7 | `cs-ids-history-service` |
| 8 | `customer-search-clean-up` |
| 9 | `customer-search-xcid-to-cmid-db` |
| 10 | `customer-search-tie-breaker-db` |
| 11 | `customer-search-real-time-loader` |
| 12 | `customer-search-data-loader` |
| 13 | `customer-search-discrepancy-db` |
| 14 | `customer-search-discrepancy-api` |
| 15 | `customer-search-ids-linkage` |
| 16 | `customer-search-amc-gfcid-db` |
| 17 | `customer-search` |
| 18 | `cs-ids-history-events-persistor` |
| 19 | `customer-search-ids-history-db` |
| 20 | `gprs-tool-set` |

For example: `CitiInternalLS1/178774.wmt-globalcusth.customer-search-query-router`.

## Transformation rules

### Existing UAT1 files: minimal diff

1. Find every connection to database `vectors` on groups 4 and 5.
2. Remove those whole vector connection entries from UAT1.
3. Keep the vector connection to group 2, including its existing credentials/options.
4. Preserve the ordering of surviving entries.
5. Leave every other UAT1 setting unchanged, including non-vector database connections, application settings, resources, pools, replicas, source settings and environment labels.
6. A UAT1 file without group-4/group-5 vector connections must remain byte-for-byte unchanged.

Do not apply a global hostname replacement to UAT1. If a connection cannot be confidently identified as a `vectors` connection, resolve its meaning from the chart/configuration code or report a blocker before modifying it.

### New UAT2 files: derive from the ORIGINAL UAT1 configuration

Capture/read the original UAT1 content before removing vector entries. Create UAT2 from that original baseline, then apply the following database-target changes:

| Connection purpose/database | UAT2 target |
|---|---|
| `vectors` | Keep groups 4 and 5 only; remove group 2. |
| `data_loader` | Group 4. |
| `discrepancy` | Group 4. |
| `amc_gfcid` | Group 4. |
| `id_history`, including schema `linkage_service_history` | Group 5. |
| `id_linkage` | Group 5. |
| `id_mapping` | Group 5. |

Include the application, job and Flyway connections for all these databases in UAT2. Keep their existing UAT1 connections unchanged. Preserve exact database/schema identifiers: `linkage_service_history` is a schema, not a replacement database name, and the AMC/GFCID identifier is `amc_gfcid`.

The UAT2 databases/schemas will need independent provisioning and population. Prepare their values files here; actual database recreation and migration execution remain outside this configuration-only task.

For query-router and query-router-db, the final vector shard lists must be:

- UAT1: group 2 only;
- UAT2: groups 4 and 5 only, retaining their original relative order.

Preserve all original database/schema names, ports, tablespaces, connection options and unrelated settings. In particular, keep `environment: uat` unless the actual project requires a different environment-specific value; do not blindly substitute `uat` with `uat2`.

Keep all logical stores: for example, individual/entity linkage entries must both remain and target group 5 in UAT2. Several entries sharing a host can represent different schemas/stores; do not deduplicate them merely because their hosts match.

### Locate ALL database connection forms

Inspect the full files and relevant templates/configuration code. Handle connection fields consistently regardless of representation, including:

- `dbs` and `flyway.dbs` lists;
- `postgres.host` or `postgres.hostname`;
- `DB_HOST`, `DB_NAME`, `DB_SCHEMA` environment fields;
- JDBC URLs and URL query parameters;
- loader/repository connection blocks;
- specialized keys such as history DB hosts or inconclusive-match DB hosts;
- nested individual/entity linkage stores.

### Additional databases beyond the approved placement table

The placement of `amc_gfcid`, `id_history` / `linkage_service_history`, and `id_mapping` is approved above; do not treat those placements as unresolved.

For any additional database not covered by the table:

1. Inspect actual repository configuration and code to identify what it is and which service owns it.
2. If its approved UAT2 placement is documented in the provided repository/user context, apply that placement and report the evidence.
3. Otherwise ask the user which of groups 4 or 5 should host it. Do not infer its target from its old group, repo name, or similarity to `id_linkage`.
4. You may complete unambiguous changes elsewhere, but report affected files as blocked and do not present a copied UAT2 file still pointing to groups 1–3 as complete.

Apply the same rule to configurations whose database is omitted or inherited: inspect merged values/templates first. An unresolved host must not silently remain a UAT1 connection inside a supposedly finished UAT2 file.

## Credentials

The user authorizes reusing existing UAT1 usernames/passwords or secret references. Take them from the existing local files; do not introduce new credentials or replace working references with dummy placeholders.

- Existing UAT1 credentials remain unchanged.
- For vector entries retained on groups 4 and 5 in UAT2, retain their original per-entry credentials.
- For retargeted non-vector entries, preserve existing credential/role conventions and inspect existing UAT1 entries for the destination group and matching database role.
- Do not blindly copy a host-specific group-1/group-3 password to group 4/5, or copy application credentials into a migration connection. If the correct destination credential for that role cannot be established from existing files, report it as a blocker for confirmation.
- Keep usernames, passwords and secret references paired correctly with connection options/roles.
- Reuse values only in the intended local configuration files. Redact passwords/tokens in reports, tool output and examples. Avoid broad commands that dump secret-bearing files to logs.
- Credential rotation or secret-management redesign is outside this task.

## Execution sequence and completion gates

### 1. Discover

- Locate all 20 repositories and applicable UAT1/UAT2 file pairs.
- Read each repository's local instructions and inspect existing working-tree changes.
- Inspect every connection in each UAT1 file, including inherited/default values where needed.
- Build a redacted table: repository, file, configuration key, database/schema, current group(s), UAT1 action, UAT2 target, unresolved dependency.
- Resolve endpoint strings and destination credential availability from existing files without printing secrets.

**Gate:** Every repository is accounted for as found, unavailable, or ineligible; every relevant connection has a known transformation or explicit blocker.

### 2. Edit

- Save the original UAT1 baseline for creating UAT2.
- Make only the permitted vector removals in UAT1.
- Create/reconcile matching UAT2 files with the approved database-target transformations.
- Preserve comments, quoting, order and formatting where possible; avoid whole-file serializer churn.
- Keep edits confined to the matching values files. If a shared default/template must change for the requested behavior, report that dependency instead of broadening the change silently.

**Gate:** Every unblocked eligible pair is updated, and files requiring unsupported placement/credential decisions are explicitly reported rather than guessed.

### 3. Verify locally without deploying

Use existing local tooling; do not build a new test/validation framework.

- Parse all changed/new YAML using available tools.
- Compare every UAT1 file to its original: only group-4/group-5 `vectors` entries may be removed. Non-vector-only UAT1 files must have no diff.
- Confirm query-router application and Flyway targets agree: UAT1 group 2; UAT2 groups 4 and 5.
- Confirm every resolved UAT2 connection targets only groups 4/5 according to the placement table.
- Confirm `data_loader`, `discrepancy`, `amc_gfcid` and `id_history` / `linkage_service_history` target group 4 in all representations.
- Confirm `id_linkage` and `id_mapping` target group 5 in all representations.
- Confirm application, background-job and Flyway connections agree for each database, including the data-loader's separate loader and repository connections.
- Confirm no credential mismatches, missing individual/entity entries, duplicate vector entries or accidental changes to schema/port/options.
- Where Helm is available, render the actual chart with each environment's base values and overrides. Check rendered connection targets locally and redact secret-bearing output. If rendering is unavailable, state that clearly.
- Inspect final diffs and run existing whitespace checks if available.

**Gate:** All completed files satisfy the routing/minimal-diff requirements, or are reported as blocked with a precise reason. A file with unresolved shared UAT1 DB access is not complete.

## Final report

Return a compact table covering all 20 target repositories:

| Repository | UAT1 file/action | UAT2 file/action | Checks run | Status/blocker |
|---|---|---|---|---|

Also list:

- any additional databases outside the approved placement table requiring a user decision;
- destination credential/role questions, without secret values;
- unavailable repositories/tools;
- any inherited configuration preventing UAT2 isolation;
- any existing non-database settings copied unchanged that still need a separate source/deployment-isolation review.

Finish by stating that this was a configuration-only change: deployment, database provisioning/reset, full population, and live shard-transition safety remain separate work. Actual database access is DBeaver-only; no command-line DB execution is authorized.
