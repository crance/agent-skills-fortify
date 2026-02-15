# Running SCA/OSS Scans
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to run a Software Composition Analysis (SCA) or Open Source Scan (OSS) to identify vulnerabilities in third-party libraries and open source components used by your application.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to scan. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
```
**Expected**: Release details including ID and available assessment types

---

### Step 2 - Check Available Assessment Types
Verify that OSS/SCA scanning is available for this release:

```tool
fcli_fod_release_list_assessment_types --release "MyApp:MyRelease"
```

**Expected**: List of available assessment types including "Open Source Analysis" or similar

---

### Step 3 - Ask User for Scan Details (Optional)
You can ask the user for the following information to understand the application better:
- **Package Manager(s)** (e.g., Maven, npm, pip, NuGet, Gradle)
- **Dependency Files** (e.g., pom.xml, package.json, requirements.txt, packages.config)
- **Source Code Location** (which directories contain dependencies)
- **Build Artifacts** (if applicable - JAR files, node_modules, etc.)

This helps determine what to include in the package.

---

### Step 4 - Package Source Code
Package the application source code and dependencies for upload:

```tool
fcli_fod_action_package 
  output-file "sca-package.zip"
  include "src/**,pom.xml,package.json,requirements.txt"
  exclude "test/**,*.log"
```

**Critical Parameters**:
- `output-file`: Path for the generated zip file
- `include`: Glob patterns for source files AND dependency manifests (pom.xml, package.json, etc.)
- `exclude`: Glob patterns for files to exclude (optional)

**Expected**: Package file created (e.g., `sca-package.zip`)

**Important**: Make sure to include dependency manifest files (pom.xml, package.json, etc.) for accurate component detection.

---

### Step 5 - Upload Package and Start SCA Scan
Upload the packaged source code and initiate the SCA/OSS scan:

```tool
fcli_fod_oss_scan_start 
  --release "MyApp:MyRelease"
  file "sca-package.zip"
```

**Expected**: Scan initiated with scan ID returned

Example response:
```json
{
  "scanId": 67890,
  "status": "Queued",
  "analysisStatusType": "Pending"
}
```

---

### Step 6 - Monitor Scan Progress
Monitor the scan until completion. **Use the scan ID returned from Step 5.** You can either:

**Option A: Wait for completion (blocking)**
```tool
fcli_fod_oss_scan_wait_for 
  scan-id "67890"
  until "COMPLETED"
  timeout "3600"
```

**Option B: Check status periodically (non-blocking)**
```tool
fcli_fod_oss_scan_get releaseQualifiedScanOrId "67890"
```
> **Note**: The parameter name is `releaseQualifiedScanOrId`, and you must provide the **scan ID** (e.g., `"67890"`), not the release name. Get the scan ID from the `oss_scan_start` response in Step 5.

**Scan Status Values**:
- `Queued` → Waiting to start
- `In Progress` → Currently scanning
- `Completed` → Scan finished successfully
- `Failed` → Scan encountered an error

**Expected**: Eventually reaches `Completed` status

---

### Step 7 - List Detected Open Source Components
Once the scan completes, view the open source components that were detected:

```tool
fcli_fod_oss_scan_list_components --release "MyApp:MyRelease"
```
> **Note**: Use the same release name from Step 1.

**Expected**: List of open source libraries/components with version numbers

Example output:
```json
{
  "records": [
    {
      "componentName": "log4j",
      "componentVersion": "2.14.1",
      "licenseName": "Apache-2.0",
      "vulnerabilityCount": 3,
      "highestSeverity": "Critical"
    },
    {
      "componentName": "spring-core",
      "componentVersion": "5.3.10",
      "vulnerabilityCount": 0,
      "highestSeverity": "None"
    }
  ]
}
```

---

### Step 8 - Retrieve Vulnerability Results
List the vulnerabilities found in the open source components:

```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details,recommendations"
  --filters-param "severityString:Critical"
```

**Expected**: List of vulnerabilities in third-party libraries with remediation advice

For a summary of findings, see [Vulnerability Summary](vulnerability-summary.md).

For remediation guidance, see [Remediation Workflow](remediation-workflow.md).

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Check available assessment types
fcli_fod_release_list_assessment_types --release "MyApp:MyRelease"

# 4. Package source code
fcli_fod_action_package 
  output-file "sca-package.zip"
  include "pom.xml,src/**"

# 5. Start OSS/SCA scan
fcli_fod_oss_scan_start 
  --release "MyApp:MyRelease"
  file "sca-package.zip"

# 6. Wait for completion
fcli_fod_oss_scan_wait_for "67890" until "COMPLETED"

# 7. List detected components
fcli_fod_oss_scan_list_components --release "MyApp:MyRelease"

# 8. View vulnerabilities
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Release not found" | Verify release name with `fcli_fod_release_list` (see [Finding Releases](find-release.md)) |
| "No components detected" | Verify dependency files (pom.xml, package.json) are included in package |
| "Package file not found" | Verify `output-file` path from `fcli_fod_action_package` |
| "Scan failed" | Get scan details with `fcli_fod_oss_scan_get` to see error message |
| "Missing dependency manifest" | Include package manager files: pom.xml, package.json, requirements.txt, etc. |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Key Differences from SAST Scans

- **SCA/OSS**: Analyzes third-party libraries and open source components
- **SAST**: Analyzes your application's source code
- **Packaging**: Same for both (use `fcli_fod_action_package`)
- **Results**: SCA finds vulnerable libraries; SAST finds code vulnerabilities
- **Detection**: SCA uses `oss_scan_list_components` to show what was found
- **Note**: To run Open Source Analysis as part of a SAST scan, use `--oss` flag in `fcli_fod_sast_scan_setup`

For SAST scanning, see [Run SAST Scan](run-sast-scan.md).

---

## Common Package Managers and Dependency Files

| Language/Platform | Package Manager | Dependency Files to Include |
|-------------------|-----------------|----------------------------|
| Java | Maven | `pom.xml`, `**/pom.xml` |
| Java | Gradle | `build.gradle`, `settings.gradle` |
| JavaScript/Node | npm | `package.json`, `package-lock.json` |
| JavaScript/Node | Yarn | `package.json`, `yarn.lock` |
| Python | pip | `requirements.txt`, `setup.py`, `Pipfile` |
| .NET | NuGet | `packages.config`, `*.csproj` |
| Ruby | Bundler | `Gemfile`, `Gemfile.lock` |
| Go | Go Modules | `go.mod`, `go.sum` |
| PHP | Composer | `composer.json`, `composer.lock` |

**Important**: Always include both the manifest file (package.json) and lock file (package-lock.json) when available for more accurate version detection.
