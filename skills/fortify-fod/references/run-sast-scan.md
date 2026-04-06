# Running SAST Scans
**Prerequisites:** Authentication verified (see SKILL.md)

## Contents
- [Use Case](#use-case)
- [Workflow Steps](#workflow-steps)
- [Complete Example Flow](#complete-example-flow)
- [Troubleshooting](#troubleshooting)
- [Key Differences from SCA Scans](#key-differences-from-sca-scans)

## Use Case
You need to run a Static Application Security Testing (SAST) scan to analyze source code for security vulnerabilities.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to scan. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"
```
**Expected**: Release details including ID and available assessment types

---

### Step 2 - Check Scan Configuration
Before starting a scan, check if SAST scan settings are already configured:

```tool
fcli_fod_sast_scan_get_config --release "MyApp:MyRelease"
```

**Expected**: 
- If configured → Scan settings with language, build tool, framework details
- If not configured → Error or empty response

---

### Step 3 - Configure Scan Settings (If Not Already Configured)
If Step 2 shows no configuration, **ask the user** for the following information:
- **Technology Stack** (e.g., Java/J2EE, Python, JavaScript, .NET / C#)
- **Language Level** (e.g., `1.8` for Java 8, `3.9` for Python 3.9 — optional)
- **Source Code Location** (which directories to include/exclude)
- **Assessment Type** (e.g., Static Assessment, Static Assessment+)
- **Entitlement Frequency** (`Subscription` or `SingleScan`)
- **Audit Preference** (`Automated` or `Manual`)
- **Open Source Analysis** (whether to include OSS/SCA scanning in this SAST scan)

**Never infer these settings from the workspace** - always prompt the user.

Once you have the information, configure the scan:

```tool
fcli_fod_sast_scan_setup 
  --release "MyApp:MyRelease"
  --technology-stack "Java/J2EE"
  --language-level "1.8"
  --assessment-type "Static Assessment"
  --entitlement-frequency "Subscription"
  --audit-preference "Automated"
  --oss
```

**Required Parameters**:
- `--release`: Release identifier (e.g., `"MyApp:MyRelease"`)
- `--assessment-type`: Assessment type (e.g., `"Static Assessment"`, `"Static Assessment+"`)
- `--entitlement-frequency`: Entitlement frequency — `"Subscription"` or `"SingleScan"`
- `--audit-preference`: Audit preference — `"Automated"` or `"Manual"`

**Optional Parameters**:
- `--technology-stack`: Technology stack (e.g., `"Java/J2EE"`, `"Python"`, `"JavaScript"`, `".NET / C#"`)
- `--language-level`: Language/framework version (e.g., `"1.8"` for Java 8, `"3.9"` for Python 3.9)
- `--oss`: Enable Open Source Analysis (boolean flag) - includes SCA/OSS scanning in the SAST scan

**Expected**: Confirmation that scan settings are configured

---

### Step 4 - Package Source Code
Package the application source code for upload:

```tool
fcli_fod_action_package 
  --output "sast-package.zip"
  --source-dir "."
  --sc-client-version "auto"
  --rel "MyApp:MyRelease"
```

**Parameters**:
- `--output`: Path for the generated zip file
- `--source-dir`: Root directory of the source code to package
- `--sc-client-version`: ScanCentral client version to use (use `"auto"` to auto-detect)
- `--rel`: Release identifier (e.g., `"MyApp:MyRelease"`) — required in practice even though listed as optional
- `--extra-opts`: Additional ScanCentral client options (e.g., `-bt none` for no build tool)

**Expected**: Package file created (e.g., `sast-package.zip`)

> ⚠️ **FCLI Bug**: The action script unconditionally evaluates `package.ossEnabled` to determine OSS flags. If `--rel` is omitted or no FoD session is active, the action crashes with a SpEL null-reference error. Always supply `--rel` and ensure an active FoD session.

**Note**: Ask the user which source directory to use if uncertain.

---

### Step 5 - Upload Package and Start Scan
Upload the packaged source code and initiate the SAST scan:

```tool
fcli_fod_sast_scan_start 
  --release "MyApp:MyRelease"
  --file "sast-package.zip"
```

**Expected**: Scan initiated with scan ID returned

Example response:
```json
{
  "scanId": 12345,
  "status": "Queued",
  "analysisStatusType": "Pending"
}
```

---

### Step 6 - Monitor Scan Progress
Monitor the scan until completion. **Use the scan ID returned from Step 5.** You can either:

#### Option A: Blocking Wait

**Tool:** `fcli_fod_sast_scan_wait_for`

```json
{
  "releaseQualifiedScanOrIds": "1391852:18001127",
  "--until": "all-match",
  "--while": "any-match",
  "--any-state": "Completed",
  "--timeout": "1h"
}
```

**Parameter Details:**
- `releaseQualifiedScanOrIds` (**required**): Use `<release-id>:<scan-id>` from `releaseAndScanId` field of `sast_scan_start` response
- `--until` (**required by MCP schema**): `"any-match"` or `"all-match"` — NOT a state name
- `--while` (**required by MCP schema**): `"any-match"` or `"all-match"`
- `--any-state`: Scan state to wait for (`Completed`, `Canceled`)
- `--timeout`: Maximum wait time with unit suffix (e.g., `"30m"`, `"1h"`)

> ⚠️ **MCP Schema Warning:** The MCP schema requires both `--until` AND `--while` simultaneously, but these are mutually exclusive options in the actual FCLI CLI. As a result, calls through the MCP tool may consistently return `exitCode: 2` with no output. **If this occurs, fall back to Option B (polling).**

#### Option B: Polling (Recommended for MCP usage)

**Tool:** `fcli_fod_sast_scan_get`

```json
{
  "releaseQualifiedScanOrId": "1391852:18001127"
}
```

Poll every 60–120 seconds until `analysisStatusType` is `Completed`.

**Scan Status Values**:
- `Queued` → Waiting to start
- `In Progress` → Currently scanning
- `Completed` → Scan finished successfully
- `Failed` → Scan encountered an error

**Expected**: Eventually reaches `Completed` status

---

### Step 7 - Retrieve Scan Results
Once the scan completes, list the vulnerabilities found:

```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details,recommendations"
  query {"severity": "Critical"}
```

**Expected**: List of security vulnerabilities with details and recommendations

For a summary of findings, see [Vulnerability Summary](vulnerability-summary.md).

For remediation guidance, see [Remediation Workflow](remediation-workflow.md).

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Check existing configuration
fcli_fod_sast_scan_get_config --release "MyApp:MyRelease"

# 4. Configure scan (if needed) - after asking user
fcli_fod_sast_scan_setup 
  --release "MyApp:MyRelease"
  language "Java"
  build-tool "Maven"

# 5. Package source code
fcli_fod_action_package 
  output-file "sast-package.zip"
  include "src/**,pom.xml"

# 6. Start scan
fcli_fod_sast_scan_start 
  --release "MyApp:MyRelease"
  --file "sast-package.zip"

# 7. Poll until completed (recommended for MCP)
fcli_fod_sast_scan_get releaseQualifiedScanOrId "<release-id>:<scan-id>"
# Repeat every 60-120s until analysisStatusType = "Completed"
# Alternative (may return exitCode 2 via MCP due to schema constraints):
# fcli_fod_sast_scan_wait_for releaseQualifiedScanOrIds "<release-id>:<scan-id>" --until "all-match" --while "any-match" --any-state "Completed" --timeout "1h"

# 8. View results
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details"
  query {"severity": "Critical"}
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Release not found" | Verify release name with `fcli_fod_release_list` (see [Finding Releases](find-release.md)) |
| "Scan not configured" | Run `fcli_fod_sast_scan_setup` after asking user for settings |
| "Package file not found" | Verify `output-file` path from `fcli_fod_action_package` |
| "Scan failed" | Get scan details with `fcli_fod_sast_scan_get` to see error message |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Key Differences from SCA Scans

- **SAST**: Analyzes source code structure and logic
- **SCA**: Analyzes open source dependencies and libraries
- **Packaging**: Same for both (use `fcli_fod_action_package`)
- **Results**: SAST finds code vulnerabilities; SCA finds vulnerable components
- **Combined Scanning**: Use `--oss` flag in `fcli_fod_sast_scan_setup` to enable Open Source Analysis in a SAST scan

For SCA/OSS scanning, see [Run SCA Scan](run-sca-scan.md).
