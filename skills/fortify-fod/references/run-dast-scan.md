# Running DAST Scans
**Prerequisites:** Authentication verified (see SKILL.md)

## Contents
- [Use Case](#use-case)
- [Overview](#overview)
- [Workflow Steps](#workflow-steps)
- [Complete Example Flow (Website Scan)](#complete-example-flow-website-scan)
- [Complete Example Flow (API Scan)](#complete-example-flow-api-scan)
- [Troubleshooting](#troubleshooting)
- [Key Differences from SAST/SCA Scans](#key-differences-from-sastsca-scans)
- [Network Access Requirements](#network-access-requirements)

## Use Case
You need to run a Dynamic Application Security Testing (DAST) scan to test a running application for security vulnerabilities by actively probing and interacting with the application.

## Overview
DAST scans are fundamentally different from SAST/SCA scans:
- **No source code packaging required** - scans a live, running application
- **Application must be accessible** - FoD needs network access to your app
- **Three scan types**: Website, API, or Workflow-driven

> **DAST Automated scans only**: The `dast_scan_setup_website`, `dast_scan_setup_api`, and `dast_scan_setup_workflow` MCP tools configure **DAST Automated** scans exclusively. Manually-conducted Dynamic Assessments (e.g. standard `"Dynamic Website Assessment"`) **cannot** be set up via these tools — FoD will reject them with HTTP 422. Only automated DAST assessment types (e.g. `"Dynamic+ Website Assessment"`) are compatible. Use `fcli_fod_release_list_assessment_types` to confirm which automated types are available for a release.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to scan. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"
```
**Expected**: Release details including ID and available assessment types

---

### Step 2 - Check Existing DAST Configuration
Before starting a scan, check if DAST scan settings are already configured:

```tool
fcli_fod_dast_scan_get_config --release "MyApp:MyRelease"
```

**Expected**: 
- If configured → DAST scan settings with URL, scan type, and configuration details
- If not configured → Error or empty response

---

### Step 3 - Determine Scan Type and Ask User
**Always ask the user** for the following information based on scan type:

#### For Website Scans:
- **Target URL** (e.g., `https://example.com`)
- **Scan Type** (Standard, Express, etc.)
- **Authentication** (if login required - credentials, tokens)
- **Network Access** (confirm FoD can reach the URL)

#### For API Scans:
- **API Specification File** (OpenAPI/Swagger JSON or YAML)
- **Base URL** (API endpoint base URL)
- **Authentication** (API keys, OAuth tokens, etc.)
- **API Version** (if applicable)

#### For Workflow Scans:
- **Workflow/Macro File** (recorded browser automation)
- **Target URL** (starting point)
- **Authentication Details**

**Never infer these settings from the workspace** - always prompt the user.

---

### Step 4 - Configure DAST Scan Settings

#### Option A: Website DAST Scan (Automated Only)

> **Important**: `dast_scan_setup_website` configures **DAST Automated** scans only. Standard "Dynamic Website Assessment" (manually conducted by FoD testing team) is **not supported** by this tool. Only automated DAST assessment types (e.g. `Dynamic+ Website Assessment`) work. Use `fcli_fod_release_list_assessment_types` to find automated DAST types available for the release.

```tool
fcli_fod_dast_scan_setup_website 
  --release "MyApp:MyRelease"
  --site-url "https://example.com"
  --assessment-type "Dynamic+ Website Assessment"
  --entitlement-frequency "Subscription"
```

**Parameters**:
- `--site-url`: Target website URL (must be accessible from FoD)
- `--assessment-type`: Name of a **DAST Automated** assessment type — check available types with `fcli_fod_release_list_assessment_types` and filter for `scanType: "Dynamic"`
- `--entitlement-frequency`: `Subscription` or `SingleScan`

---

#### Option B: API DAST Scan

> **Important**: `dast_scan_setup_api` configures **DAST Automated** scans only. Only `"DAST Automated"` is a valid assessment type — standard API assessment types (e.g. `"Dynamic+ API Assessment"`) will be rejected with HTTP 422. Use `fcli_fod_release_list_assessment_types` to confirm availability.

**Option B1 — URL-based spec** (provide API spec via URL):
```tool
fcli_fod_dast_scan_setup_api 
  --release "MyApp:MyRelease"
  --type "OpenApi"
  --api-url "https://api.example.com/openapi.json"
  --assessment-type "DAST Automated"
  --entitlement-frequency "Subscription"
```

**Option B2 — File-based spec** (upload local API spec file):
```tool
fcli_fod_dast_scan_setup_api 
  --release "MyApp:MyRelease"
  --type "OpenApi"
  --file "openapi.json"
  --assessment-type "DAST Automated"
  --entitlement-frequency "Subscription"
```

**Parameters**:
- `--type`: API specification format — `OpenApi`, `Postman`, `GraphQL`, or `Grpc`
- `--api-url`: URL to the API specification (use instead of `--file` for URL-based specs)
- `--file`: Path to local API specification file (use instead of `--api-url` for local files; file is uploaded automatically)
- `--assessment-type`: Must be `"DAST Automated"` — standard API assessment types are not compatible
- `--entitlement-frequency`: `Subscription` or `SingleScan`

---

#### Option C: Workflow DAST Scan

> **Important**: `dast_scan_setup_workflow` configures **DAST Automated** workflow-driven scans only. Standard Dynamic Website Assessment types will be rejected with HTTP 422. Only `"DAST Automated"` is a valid assessment type. Use `fcli_fod_release_list_assessment_types` to confirm availability.

```tool
fcli_fod_dast_scan_setup_workflow 
  --release "MyApp:MyRelease"
  --file "recorded-session.webmacro"
  --assessment-type "DAST Automated"
  --entitlement-frequency "Subscription"
```

**Parameters**:
- `--file`: Path to the workflow/macro file (e.g. `.webmacro` recorded browser automation)
- `--assessment-type`: Must be `"DAST Automated"` — standard Dynamic types are not compatible
- `--entitlement-frequency`: `Subscription` or `SingleScan`

---

### Step 5 - Start DAST Scan
Once configuration is complete, start the DAST scan:

```tool
fcli_fod_dast_scan_start --release "MyApp:MyRelease"
```

**Expected**: Scan initiated with scan ID

Example response:
```json
{
  "scanId": 54321,
  "status": "Queued",
  "scanType": "Website"
}
```

**Note**: No packaging or file upload needed (except for API spec or workflow files during setup)

---

### Step 6 - Monitor Scan Progress
DAST scans typically take longer than SAST/SCA scans (minutes to hours depending on application size).

**Check scan status periodically:**
```tool
fcli_fod_dast_scan_list --release "MyApp:MyRelease"
```

**Expected**: List of DAST scans with current status

Example output:
```json
{
  "records": [
    {
      "scanId": 54321,
      "scanType": "Website",
      "status": "In Progress",
      "startedDateTime": "2026-02-14T10:30:00Z",
      "completedDateTime": null
    }
  ]
}
```

**Scan Status Values**:
- `Queued` → Waiting to start
- `In Progress` → Currently scanning
- `Paused` → Scan paused (can be resumed)
- `Completed` → Scan finished successfully
- `Cancelled` → Scan was cancelled
- `Failed` → Scan encountered an error

---

### Step 7 - Retrieve Scan Results
Once the scan completes, list the vulnerabilities found:

```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details,recommendations"
  query {"severity": "Critical"}
```

**Expected**: List of dynamic security vulnerabilities (XSS, SQL injection, authentication issues, etc.)

DAST vulnerabilities are different from SAST:
- **DAST finds**: Runtime behavior issues, authentication flaws, server misconfigurations
- **SAST finds**: Source code structural issues

For a summary of findings, see [Vulnerability Summary](vulnerability-summary.md).

For remediation guidance, see [Remediation Workflow](remediation-workflow.md).

---

## Complete Example Flow (Website Scan)

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Check existing DAST configuration
fcli_fod_dast_scan_get_config --release "MyApp:MyRelease"

# 4. Configure website DAST scan (Automated only — use fcli_fod_release_list_assessment_types to find valid automated type)
fcli_fod_dast_scan_setup_website 
  --release "MyApp:MyRelease"
  --site-url "https://example.com"
  --assessment-type "Dynamic+ Website Assessment"
  --entitlement-frequency "Subscription"

# 5. Start DAST scan (NO PACKAGING!)
fcli_fod_dast_scan_start --release "MyApp:MyRelease"

# 6. Monitor progress
fcli_fod_dast_scan_list --release "MyApp:MyRelease"

# 7. View results (when completed)
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details"
```

---

## Complete Example Flow (API Scan)

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Configure API DAST scan (upload spec inline via --file)
fcli_fod_dast_scan_setup_api 
  --release "MyApp:MyRelease"
  --type "OpenApi"
  --file "openapi.json"
  --assessment-type "DAST Automated"
  --entitlement-frequency "Subscription"

# 4. Start DAST scan
fcli_fod_dast_scan_start --release "MyApp:MyRelease"

# 5. Monitor and view results
fcli_fod_dast_scan_list --release "MyApp:MyRelease"
```

> **Advanced**: If you need to reuse the same spec file across multiple releases, pre-upload it once with `fcli_fod_dast_scan_upload_file --release "MyApp:MyRelease" --file "openapi.json" --file-type "OpenAPIDefinition"`, then reference the returned `fileId` via `--file-id` in `setup_api` instead of `--file`.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Release not found" | Verify release name with `fcli_fod_release_list` (see [Finding Releases](find-release.md)) |
| "DAST not configured" | Run appropriate `dast_scan_setup_*` command after asking user for details |
| "Cannot reach target URL" | Verify application is running and accessible from FoD network; check firewall rules |
| "Authentication failed" | Provide correct credentials or authentication tokens during setup |
| "API spec invalid" | Validate OpenAPI/Swagger specification file format |
| "Scan failed" | Get scan details from `fcli_fod_dast_scan_list` to see error message |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Key Differences from SAST/SCA Scans

| Aspect | DAST | SAST/SCA |
|--------|------|----------|
| **Target** | Running application | Source code |
| **Packaging** | Not required | Required with `fcli_fod_action_package` |
| **Access** | Needs network access to app | Needs source code files |
| **Duration** | Longer (hours) | Shorter (minutes) |
| **Findings** | Runtime vulnerabilities | Code structure issues |
| **Examples** | XSS, SQL injection, auth bypass | Buffer overflow, hardcoded secrets |

For SAST scanning, see [Run SAST Scan](run-sast-scan.md).

For SCA scanning, see [Run SCA Scan](run-sca-scan.md).

---

## Network Access Requirements

**Critical**: The target application must be accessible from Fortify on Demand's scanning infrastructure.

**Options for accessibility**:
- **Public internet**: Application deployed on public cloud/hosting
- **Site-to-Site VPN**: Secure tunnel between FoD and your network
- **FoD Sensor**: On-premise scanning agent (contact FoD support)

**Ask the user to confirm** network accessibility before starting DAST scans.
