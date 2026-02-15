# Running SAST Scans
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to run a Static Application Security Testing (SAST) scan to analyze source code for security vulnerabilities.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to scan. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
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
- **Language** (e.g., Java, C#, Python, JavaScript, Go)
- **Build Tool** (e.g., Maven, Gradle, MSBuild, npm, pip)
- **Framework** (e.g., Spring Boot, .NET Core, Django, React)
- **Source Code Location** (which directories to include/exclude)
- **Assessment Type** (e.g., Static Assessment, Static Assessment+)
- **Open Source Analysis** (whether to include OSS/SCA scanning in this SAST scan)

**Never infer these settings from the workspace** - always prompt the user.

Once you have the information, configure the scan:

```tool
fcli_fod_sast_scan_setup 
  --release "MyApp:MyRelease"
  language "Java"
  build-tool "Maven"
  framework "Spring Boot"
  assessment-type "Static Assessment"
  --oss
```

**Optional Parameters**:
- `--oss`: Enable Open Source Analysis (boolean flag) - includes SCA/OSS scanning in the SAST scan

**Expected**: Confirmation that scan settings are configured

---

### Step 4 - Package Source Code
Package the application source code for upload:

```tool
fcli_fod_action_package 
  output-file "sast-package.zip"
  include "src/**"
  exclude "test/**,*.log,node_modules/**"
```

**Parameters**:
- `output-file`: Path for the generated zip file
- `include`: Glob patterns for files to include
- `exclude`: Glob patterns for files to exclude (optional)

**Expected**: Package file created (e.g., `sast-package.zip`)

**Note**: Ask the user which directories to include/exclude if uncertain.

---

### Step 5 - Upload Package and Start Scan
Upload the packaged source code and initiate the SAST scan:

```tool
fcli_fod_sast_scan_start 
  --release "MyApp:MyRelease"
  file "sast-package.zip"
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

**Option A: Wait for completion (blocking)**
```tool
fcli_fod_sast_scan_wait_for 
  scan-id "12345"
  until "COMPLETED"
  timeout "3600"
```

**Option B: Check status periodically (non-blocking)**
```tool
fcli_fod_sast_scan_get releaseQualifiedScanOrId "12345"
```
> **Note**: The parameter name is `releaseQualifiedScanOrId`, and you must provide the **scan ID** (e.g., `"12345"`), not the release name. Get the scan ID from the `scan_start` response in Step 5.

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
  --filters-param "severityString:Critical"
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
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

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
  file "sast-package.zip"

# 7. Wait for completion
fcli_fod_sast_scan_wait_for scan-id "12345" until "COMPLETED"

# 8. View results
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details"
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
