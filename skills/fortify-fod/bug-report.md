# FoD Skill — Bug Report (2026-04-05)

Schema verification of `skills/fortify-fod/` against MCP tool definitions.

---

## Confirmed Bugs (Schema Mismatch)

### B1: `--filters-param` does NOT exist on `issue_list` (CRITICAL) ✅ FIXED
- **Affects**: SKILL.md, list-filter-vulnerabilities.md, vulnerability-summary.md, remediation-workflow.md, run-sast-scan.md, run-sca-scan.md, run-dast-scan.md
- **Wrong**: `--filters-param "severityString:Critical"`, `--filters-param "category:SQL Injection"`, `--filters-param "audited:false"`, `--filters-param "suppressed:true"`
- **Schema**: `issue_list` has NO `--filters-param`. Only client-side `query` with fields: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`
- **Correct**: Use `query { "severity": "Critical" }` for severity, `query { "category": "SQL Injection" }` for category. For suppressed/fixed issues use `--include "suppressed"` or `--include "fixed"`. No query field for `audited`.
- **Impact**: ALL server-side filter examples throughout the skill are broken

### B2: `issue_list` query uses non-existent fields ✅ FIXED
- **Affects**: list-filter-vulnerabilities.md, vulnerability-summary.md
- **Wrong fields**: `primaryLocationFull`, `foundDate`, `severityString`
- **Valid query fields**: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`
- **Correct**: `primaryLocationFull` → `location`; `severityString` → `severity`; `foundDate` → not available (use `introducedDate` for client-side filtering)
- **Verified**: Live MCP test confirmed `query {"primaryLocationFull": "..."}` is silently ignored (no filtering); `query {"location": "ecs.tf.*"}` correctly filters results

### B3: `release_get` param should NOT have `--` prefix ✅ FIXED
- **Affects**: SKILL.md (Parameter Formats table), find-release.md, list-filter-vulnerabilities.md, remediation-workflow.md, run-sast-scan.md, run-sca-scan.md, run-dast-scan.md, vulnerability-summary.md
- **Wrong**: `--qualifiedReleaseNameOrId "MyApp:MyRelease"`
- **Schema**: `qualifiedReleaseNameOrId` is positional (no `--` prefix)
- **Correct**: `qualifiedReleaseNameOrId "MyApp:MyRelease"`

### B4: `app_get` uses wrong parameter name ✅ FIXED
- **Affects**: SKILL.md (Parameter Formats table), find-release.md
- **Wrong in SKILL.md**: Parameter Formats table claims `app_get` uses `--qualifiedReleaseNameOrId`
- **Wrong in find-release.md**: `fcli_fod_app_get app-name-or-id "MyApp"`
- **Schema**: `app_get` uses `appNameOrId` (positional, camelCase, no `--`)
- **Correct**: `fcli_fod_app_get appNameOrId "MyApp"`

### B5: `action_package` uses wrong parameter names (CRITICAL) ✅ FIXED
- **Affects**: SKILL.md, run-sast-scan.md, run-sca-scan.md
- **Wrong**: `output-file "sast-package.zip"`, `include "src/**"`, `exclude "test/**"`
- **Schema**: Required params are `--sc-client-version`, `--source-dir`, `--output`. No `include`/`exclude` params exist. Has `--extra-opts` for additional options.
- **Correct**: `fcli_fod_action_package --output "sast-package.zip" --source-dir "." --sc-client-version "auto" --rel "MyApp:MyRelease"`
- **Verified (2026-04-05)**: MCP call with `--output`, `--source-dir`, `--sc-client-version` accepted by tool (params valid). `output-file`/`include`/`exclude` do not exist in schema.
- **FCLI Bug found (2026-04-05)**: Action script unconditionally evaluates `package.ossEnabled` via SpEL (`${package.ossEnabled?' -oss':''}`). If `--rel` is not provided (or no FoD session is active), `package` is null and the action crashes with `EL1007E`. `--rel` is listed as optional in the schema but is **effectively required** for the action to succeed. Workaround: always supply `--rel` and ensure an active FoD session.

### B6: `sast_scan_setup` uses non-existent params; missing required (CRITICAL) ✅ FIXED
- **Affects**: run-sast-scan.md
- **Wrong**: `language "Java"`, `build-tool "Maven"`, `framework "Spring Boot"`, `assessment-type "Static Assessment"` (missing `--`)
- **Non-existent params**: `language`, `build-tool`, `framework`
- **Schema**: Use `--technology-stack`, `--language-level`. Required: `--assessment-type` (needs `--`), `--entitlement-frequency`, `--audit-preference`
- **Correct example**: `fcli_fod_sast_scan_setup --release "MyApp:MyRelease" --technology-stack "Java/J2EE" --language-level "1.8" --assessment-type "Static Assessment" --entitlement-frequency "Subscription" --audit-preference "Automated"`
- **⚠️ MANUAL VERIFY** — modifies scan configuration

### B7: `sast_scan_start` / `oss_scan_start` missing `--` on `file` ✅ FIXED
- **Affects**: run-sast-scan.md, run-sca-scan.md
- **Wrong**: `file "sast-package.zip"`
- **Correct**: `--file "sast-package.zip"`
- **Verified (2026-04-06)**: Negative test — `file` (no `--`) returns `"must have required property '--file'"`. Positive test — `--file` accepted and scan started (scanId: 18001138, Demo:1.0.0).

### B8: `*_scan_wait_for` multiple issues (CRITICAL) ✅ FIXED
- **Affects**: run-sast-scan.md, run-sca-scan.md
- **Issues**:
  1. `scan-id "12345"` → positional param is `releaseQualifiedScanOrIds` (not `scan-id`)
  2. `until "COMPLETED"` → `--until` takes `"any-match"` or `"all-match"`, NOT state names; state goes in `--any-state`
  3. `timeout "3600"` → needs unit suffix: `"1h"` or `"3600s"`
  4. Both `--until` AND `--while` REQUIRED simultaneously per schema, but they are mutually exclusive in CLI logic → **non-functional via MCP** (same bug as onprem `SC-SAST`/`SC-DAST` `scan_wait_for`)
- **Verified (2026-04-06)**: Live MCP test confirmed — passing both `--until` and `--while` returns `exitCode 2` ("mutually exclusive"); passing only `--until` returns MCP schema error ("must have required property '--while'"). Tool is confirmed non-functional via MCP.
- **Fix applied**: Removed broken `wait_for` Option A from Step 6 in both files. Polling via `*_scan_get` is now the primary documented approach. Correct `wait_for` params preserved in a collapsed `<details>` block for reference.
- **Correct params** (for future use if MCP schema bug is fixed): `releaseQualifiedScanOrIds "<release-id>:<scan-id>"  --until "all-match"  --any-state "Completed"  --timeout "1h"`

### B9: `dast_scan_setup_website` wrong params; missing required (CRITICAL) ✅ FIXED
- **Affects**: run-dast-scan.md
- **Wrong**: `url "https://example.com"` → `--site-url`; `scan-type "Standard"` → doesn't exist; `assessment-type` → `--assessment-type` (missing `--`)
- **Missing required**: `--entitlement-frequency`
- **Correct**: `fcli_fod_dast_scan_setup_website --release "MyApp:MyRelease" --site-url "https://example.com" --assessment-type "Dynamic+ Website Assessment" --entitlement-frequency "Subscription"`
- **Verified (2026-04-06)**: Negative test — omitting `--entitlement-frequency` returns `"must have required property '--entitlement-frequency'"`. Positive test — `--site-url`, `--assessment-type`, `--entitlement-frequency` all accepted by CLI. `url`/`scan-type` do not exist in schema.
- **Additional finding**: `dast_scan_setup_website` (and `setup-api`, `setup-workflow`) configure **DAST Automated** scans only (per FCLI docs). Standard non-automated "Dynamic Website Assessment" is incompatible — FoD returns HTTP 422 `"Assessment type ID must have the DAST Automated Assessment type"`. Only automated DAST types (e.g. `Dynamic+ Website Assessment`) work.

### B10: `dast_scan_setup_api` wrong params; missing required (CRITICAL) ✅ FIXED
- **Affects**: run-dast-scan.md
- **Note**: Configures **DAST Automated** API scans only (per FCLI docs). Only `"DAST Automated"` assessment type is compatible — standard types (e.g. `"Dynamic+ API Assessment"`) are rejected with HTTP 422 (same finding as B9).
- **Wrong**: `api-spec-file "openapi.json"` → `--file`; `base-url "https://..."` → doesn't exist (use `--api-url` for URL-based specs)
- **Missing required**: `--type`, `--assessment-type`, `--entitlement-frequency`
- **Correct (URL-based)**: `fcli_fod_dast_scan_setup_api --release "MyApp:MyRelease" --type "OpenApi" --api-url "https://api.example.com/openapi.json" --assessment-type "DAST Automated" --entitlement-frequency "Subscription"`
- **Correct (file-based)**: `fcli_fod_dast_scan_setup_api --release "MyApp:MyRelease" --type "OpenApi" --file "openapi.json" --assessment-type "DAST Automated" --entitlement-frequency "Subscription"`
- **Verified (2026-04-06)**: Negative test — omitting `--type` returns `"must have required property '--type'"`. Positive test 1 — `--api-url` + `--type "OpenApi"` + `--assessment-type "DAST Automated"` accepted (exitCode 0, `sourceType: "Url"`). Positive test 2 — `--file` + `--type "OpenApi"` + `--assessment-type "DAST Automated"` accepted (exitCode 0, `fileId: 1077`, `sourceType: "FileId"`). `api-spec-file`/`base-url` do not exist in schema.

### B11: `dast_scan_setup_workflow` wrong params; missing required (CRITICAL) ✅ FIXED
- **Affects**: run-dast-scan.md
- **Note**: Configures **DAST Automated** workflow-driven scans only. Only `"DAST Automated"` is a valid assessment type — standard types (e.g. `"Dynamic+ Website Assessment"`) are rejected with HTTP 422 (same finding as B9/B10).
- **Wrong**: `workflow-file "recorded-session.webmacro"` → `--file`; `url "https://..."` → doesn't exist
- **Missing required**: `--assessment-type`, `--entitlement-frequency`
- **Correct**: `fcli_fod_dast_scan_setup_workflow --release "MyApp:MyRelease" --file "recorded-session.webmacro" --assessment-type "DAST Automated" --entitlement-frequency "Subscription"`
- **Verified (2026-04-06)**: Negative test 1 — `workflow-file` (no `--`) returns `"must NOT have additional properties"`. Negative test 2 — omitting `--assessment-type` returns `"must have required property '--assessment-type'"`. Negative test 3 — `Dynamic+ Website Assessment` returns HTTP 422 `"Assessment type ID must have the DAST Automated Assessment type"`. Positive test — `--file "zws1.webmacro"` + `--assessment-type "DAST Automated"` + `--entitlement-frequency "Subscription"` accepted (exitCode 0, fileId: 1079, `setupType: "Workflow"`, Zerobank:2).

### B12: `dast_scan_upload_file` missing `--` prefix; missing required
- **Affects**: run-dast-scan.md
- **Wrong**: `file "openapi.json"` → `--file`
- **Missing required**: `--dast-file-type`
- **Correct**: `fcli_fod_dast_scan_upload_file --release "MyApp:MyRelease" --file "openapi.json" --dast-file-type "OpenAPIDefinition"`
- **⚠️ MANUAL VERIFY** — uploads file

### B13: `issue_update` wrong params; missing required (CRITICAL) ✅ FIXED
- **Affects**: remediation-workflow.md
- **Wrong**: `issue-filters "issueId:38994"` → `--vuln-ids "38994"`; `analysis-status "In Remediation"` → `--dev-status "In Remediation"` or `--auditor-status`; `comment "..."` → `--comment "..."`
- **Missing required**: `--user`
- **Correct**: `fcli_fod_issue_update --release "MyApp:MyRelease" --vuln-ids "38994" --dev-status "In Remediation" --comment "Applied Aviator fix" --user "developer@example.com"`
- **Verified (2026-04-06)**: Negative test — missing `--user` returns `"must have required property '--user'"`. Positive test — `--vuln-ids "291818525" --dev-status "In Remediation" --comment "..." --user "cho"` accepted (exitCode 0, updateCount 1, WebGoat:v2025.3). `issue-filters`/`analysis-status`/`comment` (without `--`) do not exist in schema.

### B14: `--fod-session` wrongly listed as required for all tools ✅ FIXED
- **Affects**: SKILL.md (Parameter Formats table)
- **Wrong**: `--fod-session` "REQUIRED for all tools"
- **Correct**: `--fod-session` is **optional for all tools** — FCLI auto-uses the active default session when omitted. Row removed from Parameter Formats table to match on-prem convention (which omits `--ssc-session` entirely).
- **Verified (2026-04-06)**: Negative test 1 — `session_list` called without `--fod-session` → exitCode 0, session listed. Negative test 2 — `app_list` called without `--fod-session` → exitCode 0, apps returned. Positive test — `app_list` called with `--fod-session "default"` → exitCode 0, same results.

### B15: `--release` format omits microservice segment ✅ FIXED
- **Affects**: SKILL.md (Parameter Formats table)
- **Wrong**: `"<App>:<Release>"`
- **Schema**: `"<application>[:<microservice>]:<release>"`
- **Correct**: `"<App>[:<MicroService>]:<Release>"` — both rows updated in Parameter Formats table
- **Verified (2026-04-06)**: Positive test — `release_get qualifiedReleaseNameOrId "nonexistent:service:release"` returned `"Cannot find release nonexistent:service:release"` (format parsed correctly, not rejected as invalid). 2-part format `"MyApp:MyRelease"` remains valid for apps without microservices.

---

## Operations Requiring Manual Verification

These operations involve mutations (starting scans, modifying config, uploading files) and cannot be verified by schema alone:

| Tool | Operation Type | Bug IDs | Notes |
|------|---------------|---------|-------|
| `sast_scan_setup` | Config change | B6 | Completely wrong params; test corrected syntax |
| `sast_scan_start` | Start scan | B7 | `--file` prefix; verify scan triggers |
| `sast_scan_wait_for` | Wait/poll | B8 | Likely non-functional (same `--until`/`--while` schema bug as onprem) |
| `oss_scan_start` | Start scan | B7 | Same `--file` prefix issue |
| `oss_scan_wait_for` | Wait/poll | B8 | Same `--until`/`--while` issue |
| `dast_scan_setup_website` | Config change | B9 | Wrong params + missing required |
| `dast_scan_setup_api` | Config change | B10 | Wrong params + missing required |
| `dast_scan_setup_workflow` | Config change | B11 | Wrong params + missing required |
| `dast_scan_start` | Start scan | — | Schema looks OK (only requires `--release`); verify behavior |
| `dast_scan_upload_file` | File upload | B12 | Missing `--dast-file-type` |
| `issue_update` | Data mutation | B13 ✅ | Wrong params + missing required — VERIFIED FIXED |}
| `action_package` | Create package | B5 | Completely wrong params |

---

## Summary

| Severity | Count | Description |
|----------|-------|-------------|
| CRITICAL | 7 | B1, B5, B6, B8, B9, B10, B11 — core workflows completely broken |
| Medium | 5 | B2, B3, B4, B7, B12 — wrong names/prefixes; will fail at runtime |
| Low | 2 | B14 ✅, B15 ✅ — minor inaccuracies |
| **Total** | **15** | |

### Files Affected
| File | Bug Count | Bug IDs |
|------|-----------|---------|
| SKILL.md | 5 | B1, B3, B4, B5, B14 ✅, B15 ✅ |
| list-filter-vulnerabilities.md | 4 | B1, B2, B3 |
| vulnerability-summary.md | 4 | B1, B2, B3 |
| remediation-workflow.md | 3 | B1, B3, B13 |
| run-sast-scan.md | 6 | B1, B3, B5, B6, B7, B8 |
| run-sca-scan.md | 5 | B1, B3, B5, B7, B8 |
| run-dast-scan.md | 6 | B1, B3, B9, B10, B11, B12 |
| find-release.md | 2 | B3, B4 |
