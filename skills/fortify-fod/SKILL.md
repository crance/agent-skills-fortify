---
name: fortify-fod
description: "use this skill whenever the user wants to list and filter application security findings, run SAST/SCA/DAST scans, discover applications and releases, and manage security scanning using Fortify on Demand (FoD). Triggers include: any mention of 'FoD', 'Fortify on Demand', 'list vulnerabilities', 'run SAST scan', 'run SCA scan', 'run DAST scan', 'list applications', 'list releases', 'package source code', 'security scan', and similar requests indicating interaction with FoD for application security scanning and vulnerability management."
compatibility: Requires Fortify MCP configured for FoD, and authenticated session to FoD.
metadata:
  author: crance
  version: "0.0.1"
---

# Fortify on Demand (FoD) Skill
Fortify on Demand (FoD) integration via Model Context Protocol (MCP).

## When to Use This Skill
- List applications and releases
- Run security scans (SAST, SCA, DAST, MAST)
- List security issues/vulnerabilities with filtering by severity, category, etc.
- Count issues grouped by severity, category, etc.
- Manage scan configurations and monitor scan progress
- Generate and download security reports

## Parameter Formats
Common formats and examples for key parameters:
| Parameter | Format | Example |
|-----------|-------------|-------------|
| `--release` | `"<App>[:<MicroService>]:<Release>"` - case-sensitive, colon-separated (for `*_list`, `*_scan_setup`, `*_scan_start`, `*_scan_get_config` tools) |  `"MyApp:MyRelease"` or `"MyApp:MyService:MyRelease"` |
| `qualifiedReleaseNameOrId` | `"<App>[:<MicroService>]:<Release>"` - positional param, case-sensitive, colon-separated (for `release_get` tool) |  `"MyApp:MyRelease"` or `"MyApp:MyService:MyRelease"` |
| `appNameOrId` | Application name or ID - positional param, camelCase (for `app_get` tool) | `"MyApp"` or `"5011"` |
| `releaseQualifiedScanOrId` | Scan ID or qualified scan ID (for `*_scan_get` tools) - **Always use scan ID returned from `*_scan_start` or from `*_scan_list`** | `"12345"` or `"MyApp:MyRelease:12345"` |
| `--include` | Control which issue statuses to include. By default, only `visible` issues returned. Comma-separated values: `visible`, `fixed`, `suppressed` | `"visible,fixed"` or `"suppressed"` |
| `--embed` | Comma-separated values to include additional data. Valid values: `allData`, `summary`, `details`, `recommendations`, `history`, `requestResponse`, `headers`, `parameters`, `traces` | `"details,recommendations,history"` |
| `file` | Path to packaged zip or report output | `"package.zip"`, `"report.pdf"` |

## Authentication
**All operations require authentication.** Always verify session before any operation:
```tool
fcli_fod_session_list refresh-cache=true
```
- If `Expired` = `No` → proceed
- If expired → ask user to run locally: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>`
- When running any FoD tool, if authentication error occurs, prompt user to re-authenticate locally.

**Note:** Reference workflows assume authentication has been verified.

## Domain-Specific Guidance

### Scan Workflows: Always Check Settings First
Before starting any scan, follow this sequence:
1. **Check existing scan configuration** using `*_scan_get_config` command
2. **If not configured** → Always ask user for required settings (language, build tool, framework, etc.)
3. **Never infer settings** from workspace - build tools, language versions, and frameworks must be user-confirmed
4. **Package source code** (SAST/SCA only) using `fcli_fod_action_package`
5. **Upload and start scan** using appropriate `*_scan_start` command
6. **Monitor progress** using `*_scan_wait_for` or periodic `*_scan_get` calls

### Packaging Requirements
- **SAST scans**: Package source code with `fcli_fod_action_package`
- **SCA/OSS scans**: Package source code with `fcli_fod_action_package` (same as SAST)
- **DAST scans**: No packaging needed - scans live running application
- **MAST scans**: Upload mobile app binary (APK/IPA file)
- **Note**: To enable Open Source Analysis in a SAST scan, use `--oss` flag in `fcli_fod_sast_scan_setup`

### Filtering: Use `query` for Client-Side, `--include` for Status
- **Use `query`** for client-side filtering by valid fields: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`
- **Use `--include`** to control issue status visibility: `visible` (default), `fixed`, `suppressed`
- **`--filters-param` does NOT exist** — do not use it; it will fail
- Common examples: `query {"severity": "Critical"}`, `query {"category": "SQL Injection"}`, `--include "suppressed"`

### Pagination
- If `pagination.hasMore` = true → use `pagination-offset` for next page
- Continue until `pagination.hasMore` = false or `pagination.totalRecords` reached

## Error Recovery
| Error | Recovery |
|-------|----------|
| "Session expired" | Refer to flow in `Authentication` section |
| "Release not found" | Run `release_list` to discover correct names (see [Finding Releases](references/find-release.md)) |
| "Scan not configured" | Ask user for scan settings and run `*_scan_setup` |
| "Package required" | Run `fcli_fod_action_package` to package source code |

## Decision Tree: Choosing the Right Approach

| User Intent | Action |
|-------|----------|
| "run SAST scan" / "static analysis" | Check config → ask settings → package → `sast_scan_start` (see [SAST Workflow](references/run-sast-scan.md)) |
| "run SCA scan" / "open source scan" | Package → `oss_scan_start` (see [SCA Workflow](references/run-sca-scan.md)) |
| "run DAST scan" / "dynamic scan" | **DAST Automated only** — `setup-website/api/workflow` only support automated DAST types; manually-conducted Dynamic Assessments cannot be configured via MCP. Check config → ask settings → `dast_scan_start` (see [DAST Workflow](references/run-dast-scan.md)) |
| "list/show vulnerabilities" | `issue_list` with `query {"severity": "Critical"}` + `--embed details,recommendations` — see [List and Filter Vulnerabilities](references/list-filter-vulnerabilities.md) |
| "how many / count / summary" | `issue_list` and aggregate results client-side — see [Vulnerability Summary](references/vulnerability-summary.md) |
| "find release / which release" | `release_list` → `release_get` (see [Finding Releases](references/find-release.md)) |
| "show recommendations / how to fix" | `issue_list` with `--embed recommendations,history` → prioritize Aviator (see [Remediation](references/remediation-workflow.md)) |

## Best Practices
**DO:**
- ✅ Always verify authentication before operations
- ✅ Check scan configuration before starting SAST scans
- ✅ Always ask user for SAST scan settings (language, build tool, framework)
- ✅ Use `--oss` flag in `sast_scan_setup` to enable Open Source Analysis in SAST scans
- ✅ Use `query` for client-side filtering (valid fields: `severity`, `category`, `location`, `instanceId`, `foundInReleases`, `visibilityMarker`)
- ✅ Use `--include "suppressed"` or `--include "fixed"` to retrieve non-default issue statuses
- ✅ Use `--embed` to include details, recommendations, and history
- ✅ Prioritize Fortify Aviator code fix suggestions in remediation
- ✅ Use MCP tools over FCLI CLI directly
- ✅ Monitor long-running scans with `*_scan_wait_for`

**DO NOT:**
- ❌ Guess release names - always discover with `release_list` if uncertain
- ❌ Infer SAST scan settings from workspace - always ask user
- ❌ Skip SAST scan configuration validation
- ❌ Prompt user for credentials - ask user to run `fcli fod session login` locally
- ❌ Start scans without confirming settings with user
- ❌ Package source code for DAST scans (not needed)
- ❌ Use `dast_scan_setup_*` for non-automated (manually-conducted) DAST assessments — only **DAST Automated** assessment types are supported

## References
### Example Workflows
| Workflow | Use When User Says... |
|----------|----------------------|
| [Run SAST Scan](references/run-sast-scan.md) | "run SAST scan", "static analysis", "scan source code", "check for code vulnerabilities" |
| [Run SCA Scan](references/run-sca-scan.md) | "run SCA scan", "open source scan", "check dependencies", "OSS vulnerabilities", "software composition analysis" |
| [Run DAST Scan](references/run-dast-scan.md) | "run DAST scan", "dynamic scan", "test running application", "web application security test" |
| [List and Filter Vulnerabilities](references/list-filter-vulnerabilities.md) | "list vulnerabilities", "show security issues", "filter issues by severity", "critical vulnerabilities" |
| [Find Release](references/find-release.md) | "find release", "which release", "list releases", "search for application" |
| [Vulnerability Summary](references/vulnerability-summary.md) | "count vulnerabilities", "show summary", "breakdown by severity", "how many issues" |
| [Remediation Workflow](references/remediation-workflow.md) | "show recommendations", "how to fix", "remediation advice", "Aviator suggestions", "code fixes" |

### External Resources
- [FCLI Documentation](https://fortify.github.io/fcli/)
- [FoD Documentation](https://www.microfocus.com/documentation/fortify-on-demand/)
- [FoD User Guide](https://www.microfocus.com/documentation/fortify-on-demand/2410/FOD_Help_24.1.0/index.htm)
