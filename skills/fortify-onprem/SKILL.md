---
name: fortify-onprem
description: "Use this skill whenever the user wants to list and filter application security findings, run SAST or DAST scans, discover applications and versions, and manage security assessments using Fortify on-premises products: Software Security Center (SSC), ScanCentral SAST (SC-SAST), and ScanCentral DAST (SC-DAST). Triggers include: any mention of 'SSC', 'ScanCentral', 'SC-SAST', 'SC-DAST', 'list vulnerabilities', 'run SAST scan', 'run DAST scan', 'list applications', 'DAST scan', 'web scan', 'dynamic scan', and similar requests for on-premises Fortify products."
compatibility: Requires Fortify MCP configured for SSC, SC-SAST, and/or SC-DAST with an authenticated SSC session.
metadata:
  author: crance
  version: "0.0.1"
---

# Fortify On-Premises Skill
Combined skill for Fortify on-premises products via Model Context Protocol (MCP):
- **SSC** — Application security findings, vulnerability management
- **SC-SAST** — Static application security testing at scale
- **SC-DAST** — Dynamic application security testing

## Parameter Formats
| Parameter | Format | Example |
|-----------|--------|---------|
| `appVersionNameOrId` / `--appversion` / `--publish-to` | `"<App>:<Version>"` — case-sensitive, colon-separated | `"MyApp:1.0"` |
| `--filter` (SSC issues) | `"<FilterType>:<Value>"` — discover via `issue_list_filters` first | `"Folder:Critical"` |
| `--filterset` | Filter set title or ID | `"Security Auditor View"` |
| `--embed` (SSC issues) | Comma-separated values | `"details,auditHistory"` |
| `--by` (SSC issue_count) | Group name from `issue_list_groups` | `"Folder"`, `"Category"` |
| `--sc-client-version` (SC-SAST) | Version string or `latest` | `"latest"` (recommended) |
| `--settings` (SC-DAST) | CICD token or numeric ID | `"MY_APP_SETTINGS"` |
| `--mode` (SC-DAST) | Scan mode string | `"CrawlOnly"`, `"CrawlAndAudit"`, `"AuditOnly"` |
| `--login-macro` (SC-DAST) | Login macro ID (optional) | `"98765"` |
| `--until` (SC-DAST wait) | Wait condition | `"status=Complete"` |
| `--server-queries` (SC-DAST) | Server-side text search | `"searchText=keyword"` (only verified working form) |
| `scanIds` (SC-DAST) | Scan identifier | Numeric ID from list/get |
| `scanJobToken` / `scanJobTokens` (SC-SAST) | UUID string | `"550e8400-e29b-41d4-a716-446655440000"` |

## Authentication

**All operations require an active SSC session.** Always verify before any operation:
```tool
fcli_ssc_session_list with refresh-cache=true
```
- If `Expired` = `No` → proceed
- If expired → ask user to run locally (see commands below)

### SSC Only
```bash
fcli ssc session login --url "<SSC_URL>" -u "<user>" -p "<pass>"
```

### SC-SAST (SSC + additional SC-SAST parameters)
```bash
fcli ssc session login --url "<SSC_URL>" -u "<user>" -p "<pass>" \
  --sc-sast-url "<SC_SAST_URL>" --client-auth-token <TOKEN>
```
- `--client-auth-token` is obtained from SSC Administration → Settings → ScanCentral Client

### SC-DAST (standard SSC auth — no extra parameters)
```bash
fcli ssc session login --url "<SSC_URL>" -u "<user>" -p "<pass>"
```

**Note:** Reference workflows assume authentication has been verified.

## Filtering

### SSC Issues
- **Prefer `--filter`** for server-side filtering (fastest, smallest payloads)
- Discover available filters with `issue_list_filters` before applying
- Optionally use `query` as a client-side post-filter when needed

### SC-SAST Scans
- Use `query` JSON parameter for client-side filtering: `{"status": "RUNNING"}`

### SC-DAST Scans
- **Use client-side `query` JSON** for all status and name filtering: `{"scanStatusTypeDescription": "Running"}`, `{"name": ".*MyApp.*"}`
- `--server-queries "searchText=keyword"` is the only verified server-side option (full-text search across scan fields)
- `--server-queries "status=X"` does **not** filter by status (confirmed non-functional)
- `--server-queries "name~X"` uses an unsupported operator (returns exitCode 2)

## Pagination

### SSC
- `pagination.hasMore` = true → use `pagination-offset` for next page
- `pagination.jobToken` present → use `fcli_ssc_mcp_job` (see [Background Job Handling](references/mcp-job-wait.md))

### SC-DAST
- `pagination.totalRecords` is null → use `fcli_sc_dast_mcp_job` with `operation: "wait"` and the `job_token`, then retry

## Error Recovery
| Error | Recovery |
|-------|----------|
| "Session expired" | Ask user to run appropriate `fcli ssc session login` command locally |
| "Application version not found" | Run `app_list` → `appversion_list` to discover correct names |
| "Unknown filter" | Run `issue_list_filters` to discover valid filters |
| "No sensors available" (SC-SAST) | Use `sensor_pool_list` to check pool availability |
| "Version mismatch" (SC-SAST) | Repackage with `--sc-client-version "latest"`, omit `--sensor-version` |
| "Scan settings not found" (SC-DAST) | Run `scan_settings_list` to find valid settings/CICD tokens |

## Decision Tree: Choosing the Right Approach
| User Intent | Action |
|------------|--------|
| "list/show vulnerabilities" | `issue_list` with `--filter` + `--embed details` — see [List, Filter and Query Issues](references/list-filter-query-issues.md) |
| "count / summary / breakdown" | `issue_count` with `--by` — see [Summarise and Count Issues](references/issue-summary.md) |
| "find app / which version" | `app_list` → `appversion_list` — see [List and Find Application Versions](references/list-application-version.md) |
| "run SAST scan" | 1. `action_package` → 2. `scan_start` → 3. `scan_wait_for` → 4. `issue_list` — see [Run SAST Scan](references/run-sast-scan.md) |
| "list / check SAST scans" | `fcli_sc_sast_scan_list` with optional `query` filter — see [List and Monitor SAST Scans](references/sast-list-monitor-scans.md) |
| "run DAST scan" | 1. `scan_settings_list` → 2. `scan_start` → 3. `scan_wait_for` — see [Run DAST Scan](references/run-dast-scan.md) |
| "list / check DAST scans" | `fcli_sc_dast_scan_list` with optional `query` filter — see [List and Monitor DAST Scans](references/dast-list-monitor-scans.md) |
| "discover scan settings/policies" | `scan_settings_list`, `scan_policy_list` — see [Discover Scan Settings and Policies](references/discover-scan-settings-policies.md) |
| "pause/resume DAST scan" | `scan_pause` / `scan_resume` with `scanIds` — see [List and Monitor DAST Scans](references/dast-list-monitor-scans.md) |

## Best Practices
**DO:**
- ✅ Always verify session before any operation
- ✅ Use `--filter` for SSC issue server-side filtering
- ✅ Discover filters with `issue_list_filters` before applying
- ✅ Use `"latest"` for `--sc-client-version` when packaging (SC-SAST)
- ✅ Omit `--sensor-version` for auto-matching (SC-SAST modern clients)
- ✅ Use client-side `query` JSON for SC-DAST status/name filtering; `--server-queries "searchText=keyword"` for full-text search only
- ✅ Capture `scanJobToken` from SC-SAST `scan_start` response
- ✅ Use `--publish-to` on SC-SAST scans so results appear in SSC
- ✅ Use `--embed` to get additional details in a single API call (SSC)
- ✅ Set appropriate timeouts on `scan_wait_for` (SAST: 15-60 min; DAST: 30 min-2 hrs)

**Do NOT:**
- ❌ Guess application/version names — ask the user
- ❌ Prompt user for credentials — ask user to run `fcli ssc session login` locally
- ❌ Assume filter names exist — always run `issue_list_filters` first
- ❌ Use `--sc-client-version "auto"` — use `"latest"` instead (avoids outdated defaults)
- ❌ Skip `--publish-to` on SC-SAST scans — results won't appear in SSC
- ❌ Start DAST scans without verifying scan settings are configured to auto-publish results to SSC
- ❌ Use `--overwrite-scan-mode` or `--priority-scan-mode` (these parameters do not exist) — use `--mode` with `"CrawlOnly"`, `"CrawlAndAudit"`, or `"AuditOnly"` instead
- ❌ Poll scan status manually — use `scan_wait_for` instead

## References
### Example Workflows
| Workflow | Use When User Says... |
|----------|----------------------|
| [List and Find Application Versions](references/list-application-version.md) | "list applications", "show app versions", "what applications are available" |
| [List, Filter and Query Issues](references/list-filter-query-issues.md) | "list vulnerabilities", "show issues", "filter by severity", "include suppressed" |
| [Summarise and Count Issues](references/issue-summary.md) | "count issues", "show summary", "breakdown by severity", "OWASP mapping" |
| [Provide Recommendations](references/provide-recommendations.md) | "show recommendations", "how to fix", "remediation advice" |
| [Background Job Handling](references/mcp-job-wait.md) | When `pagination.jobToken` appears in SSC responses |
| [Run SAST Scan](references/run-sast-scan.md) | "run SAST scan", "scan my code", "package and scan", "static analysis" |
| [List and Monitor SAST Scans](references/sast-list-monitor-scans.md) | "list SAST scans", "check scan status", "SAST scan history", "monitor SAST" |
| [Run DAST Scan](references/run-dast-scan.md) | "run DAST scan", "web scan", "dynamic scan", "application scan" |
| [List and Monitor DAST Scans](references/dast-list-monitor-scans.md) | "list DAST scans", "DAST scan status", "pause scan", "resume scan" |
| [Discover Scan Settings and Policies](references/discover-scan-settings-policies.md) | "scan settings", "available policies", "CICD token", "scan configuration" |

### External Resources
- [FCLI Documentation](https://fortify.github.io/fcli/)
- [SSC Documentation](https://www.microfocus.com/documentation/fortify-software-security-center/)
