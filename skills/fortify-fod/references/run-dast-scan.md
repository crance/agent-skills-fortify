# Running DAST Scans
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to run a Dynamic Application Security Testing (DAST) scan to test a running application for security vulnerabilities by actively probing and interacting with the application.

## Overview
DAST scans are fundamentally different from SAST/SCA scans:
- **No source code packaging required** - scans a live, running application
- **Application must be accessible** - FoD needs network access to your app
- **Three scan types**: Website, API, or Workflow-driven

## Workflow Steps

### Step 1 - Select Release
First, identify the release to scan. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
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

#### Option A: Website DAST Scan
```tool
fcli_fod_dast_scan_setup_website 
  --release "MyApp:MyRelease"
  url "https://example.com"
  scan-type "Standard"
  assessment-type "Dynamic Assessment"
```

**Parameters**:
- `url`: Target website URL (must be accessible from FoD)
- `scan-type`: Standard, Express, or other available types
- `assessment-type`: Type of DAST assessment purchased

---

#### Option B: API DAST Scan
```tool
fcli_fod_dast_scan_setup_api 
  --release "MyApp:MyRelease"
  api-spec-file "openapi.json"
  base-url "https://api.example.com/v1"
```

**Parameters**:
- `api-spec-file`: Path to OpenAPI/Swagger specification
- `base-url`: Base URL for API endpoints

If you have the API spec file to upload:
```tool
fcli_fod_dast_scan_upload_file 
  --release "MyApp:MyRelease"
  file "openapi.json"
```

---

#### Option C: Workflow DAST Scan
```tool
fcli_fod_dast_scan_setup_workflow 
  --release "MyApp:MyRelease"
  workflow-file "recorded-session.webmacro"
  url "https://example.com"
```

**Parameters**:
- `workflow-file`: Browser automation/macro file
- `url`: Starting URL for the workflow

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
  --filters-param "severityString:Critical"
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
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Check existing DAST configuration
fcli_fod_dast_scan_get_config --release "MyApp:MyRelease"

# 4. Configure website DAST scan (after asking user for URL)
fcli_fod_dast_scan_setup_website 
  --release "MyApp:MyRelease"
  url "https://example.com"
  scan-type "Standard"

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
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Upload API specification
fcli_fod_dast_scan_upload_file 
  --release "MyApp:MyRelease"
  file "openapi.json"

# 4. Configure API DAST scan
fcli_fod_dast_scan_setup_api 
  --release "MyApp:MyRelease"
  api-spec-file "openapi.json"
  base-url "https://api.example.com/v1"

# 5. Start DAST scan
fcli_fod_dast_scan_start --release "MyApp:MyRelease"

# 6. Monitor and view results
fcli_fod_dast_scan_list --release "MyApp:MyRelease"
```

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
