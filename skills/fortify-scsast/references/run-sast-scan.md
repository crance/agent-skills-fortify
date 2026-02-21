# Run SAST Scan Workflow

## Use Case
Package source code and run a SAST scan on ScanCentral sensors, monitor progress, and retrieve vulnerability results from SSC.

## Keywords
run scan, SAST, package, upload, start scan, monitor progress, scan my code, package and scan, upload for scanning, SAST analysis, security scan, code analysis, static analysis, version mismatch, scan stuck, scan pending, scan won't start, client version, sensor version, troubleshoot scan

## Prerequisites
- Active SSC session with ScanCentral client auth token
- Valid SSC application version (target for results)
- Source code directory accessible
- ScanCentral sensors available with matching version

## Complete Workflow

### Step 1: Verify Authentication
Check that SSC session is active with ScanCentral access:

**Tool:** `fcli_ssc_session_list`

**Parameters:**
```json
{}
```

**Expected output:** Active session with `expired: false`

**If expired or missing:** Ask user to run locally:
```bash
fcli ssc session login --url https://ssc.example.com --user <username> --password <password> --sc-sast-url https://scsast.example.com/scancentral-ctrl --client-auth-token <TOKEN>
```

### Step 2: Verify Target Application Version
Ensure the SSC application version exists before starting the scan:

**Tool:** `fcli_ssc_appversion_get`

**Parameters:**
```json
{
  "appVersionNameOrId": "MyApp:1.0"
}
```

**Expected output:** Application version details with `id` field

**If not found:** Ask user to create the application version in SSC first, or verify the correct application:version name.

### Step 3: Package Source Code
Create a scan package from the source directory:

**Tool:** `fcli_ssc_action_package`

**Parameters:**
```json
{
  "--sc-client-version": "latest",
  "--source-dir": ".",
  "--output": "package.zip"
}
```

**Parameter Details:**
- `--sc-client-version` (**required**): Use `"latest"` (recommended) to automatically install the most recent client version that matches your sensors
- `--source-dir` (**required**): Path to source code (use "." for current directory)
- `--output` (**required**): Output package file path

**Expected output:** Package file created successfully at specified path

**Common issues:**
- Missing build files: Ensure project is in a buildable state

**Note:**
- Packaging source code may result in long response from build tools. Once `"status" : "completed"` is received, ignore the packaging output and do not include in context anymore.

#### Alternative: Explicit Version Control
**Only use this if you need precise control over which client and sensor versions are used.**

Check sensor versions first:

**Tool:** `fcli_sc_sast_sensor_list`

**Parameters:**
```json
{}
```

**Steps:**
1. Identify the highest sensor version (e.g., "25.4" from "25.4.0.0047")
2. Package with explicit client version using `--sc-client-version: "25.4"`
3. Start scan with explicit sensor version: `--sensor-version: "25.4"` (see Step 4)

**⚠️ Note:** When specifying explicit versions, you must use the SAME version for both `--sc-client-version` and `--sensor-version`.

### Step 4: Start SAST Scan
Submit the package to ScanCentral sensors:

**Tool:** `fcli_sc_sast_scan_start`

**Parameters:**
```json
{
  "--publish-to": "MyApp:1.0",
  "--file": "package.zip"
}
```

**Parameter Details:**
- `--publish-to` (**required**): SSC application:version to publish results (format: "App:Version")
- `--file` (**required**): Path to package created in Step 3
- `--sensor-pool` (optional): Specific sensor pool UUID
- `--sensor-version` (optional): Omit for auto-match (recommended). Only specify if you need explicit version control.

**Expected output:** JSON response containing `scanJobToken` (UUID)

**CRITICAL:** Capture the `scanJobToken` from the response - you'll need it for monitoring!

Example response:
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000",
  "status": "QUEUED"
}
```

**Common issues:**
- "No sensors available": Check sensor pools with `sensor_pool_list`
- "Version mismatch": Verify sensor version matches `sensor-version`
- "Package file not found": Verify path from Step 3

### Step 5: Monitor Scan Progress
Choose one of two monitoring approaches:

#### Option A: Blocking Wait (Recommended for immediate results)
Wait for scan to complete with timeout:

**Tool:** `fcli_sc_sast_scan_wait_for`

**Parameters:**
```json
{
  "scanJobTokens": "550e8400-e29b-41d4-a716-446655440000",
  "--until": "COMPLETED",
  "--timeout": "3600"
}
```

**Parameter Details:**
- `scanJobTokens` (**required**): The UUID from Step 4 output
- `--until` (**required**): Target state ("COMPLETED", "PUBLISHED", or "FAILED")
- `--timeout` (**required**): Maximum wait time in seconds (3600 = 1 hour)

**Expected output:** Scan reaches target state or timeout occurs

**Timing guidance:**
- Small projects: 5-15 minutes
- Medium projects: 15-30 minutes
- Large projects: 30-60 minutes
- Set timeout conservatively (recommend 3600 seconds)

#### Option B: Polling (For async monitoring)
Periodically check scan status:

**Tool:** `fcli_sc_sast_scan_status`

**Parameters:**
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Scan states:**
- `QUEUED`: Waiting for sensor
- `RUNNING`: Analysis in progress
- `COMPLETED`: Scan finished, results available
- `PUBLISHED`: Results published to SSC
- `FAILED`: Scan encountered error

Poll every 60-120 seconds until status is `COMPLETED` or `PUBLISHED`.

### Step 6: Retrieve Vulnerability Results
After scan completes and publishes, view issues in SSC:

**Tool:** `fcli_ssc_issue_list`

**Parameters:**
```json
{
  "--appversion": "MyApp:1.0",
  "--embed": "details,recommendations"
}
```

**Parameter Details:**
- `--appversion` (**required**): Same value used in `--publish-to` (Step 4)
- `--embed` (optional): Include additional details (details, recommendations, audits, etc.)
- `--filter` (optional): Server-side filtering (e.g., `FILTER[severity]:Critical,High`)

**Expected output:** List of security issues found in the scan

**Additional filtering options:**
```json
{
  "--appversion": "MyApp:1.0",
  "--embed": "details,recommendations",
  "--filter": "FILTER[severity]:Critical"
}
```

## Alternative: Download FPR File
If you need the raw FPR file instead of SSC issues:

**Tool:** `fcli_sc_sast_scan_download`

**Parameters:**
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000",
  "--fpr": "output.fpr"
}
```

**When to use:**
- Need offline analysis
- Importing to another system
- Archiving scan results

## Error Handling

### Authentication Errors
```
Error: No active SSC session
Recovery: Ask user to run fcli ssc session login with --client-auth-token
```

### Packaging Errors
```
Error: Build files not found
Recovery: Verify source directory contains buildable code, check for build.gradle, pom.xml, etc.
```

### Scan Failures
```
Error: Scan failed during analysis
Recovery: Download scan logs with scan_download --log, check for build issues or analysis errors
```

### Publish Failures
```
Error: Failed to publish to SSC
Recovery: Verify SSC application version exists, check connectivity, review SSC permissions
```

## Complete Example Flow

**1. Check authentication:**
```json
{
  "tool": "fcli_ssc_session_list",
  "parameters": {}
}
```

**2. Verify target application:**
```json
{
  "tool": "fcli_ssc_appversion_get",
  "parameters": {
    "appVersionNameOrId": "WebApp:main"
  }
}
```

**3. Package with latest client version (auto-matches sensors):**
```json
{
  "tool": "fcli_ssc_action_package",
  "parameters": {
    "--sc-client-version": "latest",
    "--source-dir": ".",
    "--output": "webapp-scan.zip"
  }
}
```

**4. Start scan (capture scanJobToken from output):**
```json
{
  "tool": "fcli_sc_sast_scan_start",
  "parameters": {
    "--publish-to": "WebApp:main",
    "--file": "webapp-scan.zip"
  }
}
```
**Note:** `--sensor-version` omitted - auto-selected by ScanCentral

**Output:** `{"scanJobToken": "abc123...", "status": "QUEUED"}`

**5. Wait for completion:**
```json
{
  "tool": "fcli_sc_sast_scan_wait_for",
  "parameters": {
    "scanJobTokens": "abc123...",
    "--until": "COMPLETED",
    "--timeout": "3600"
  }
}
```

**6. View results:**
```json
{
  "tool": "fcli_ssc_issue_list",
  "parameters": {
    "--appversion": "WebApp:main",
    "--embed": "details,recommendations",
    "--filter": "FILTER[severity]:Critical,High"
  }
}
```

## Tips and Best Practices

1. **Use `"latest"` for simplicity**: Always use `--sc-client-version: "latest"` and omit `--sensor-version` - ScanCentral will auto-match the appropriate sensor.

2. **Timeout values**: Set generous timeouts - scans take longer than expected, especially for large codebases

3. **Job token tracking**: Store the scanJobToken from `scan_start` - you'll need it for all monitoring operations

4. **Publish target**: Always specify `--publish-to` to automatically publish results to SSC. Without it, you must manually retrieve FPR files.

5. **Incremental scans**: For faster feedback, consider scanning modules or components separately

6. **Sensor pools**: When multiple pools exist, specify `--sensor-pool` to target specific sensor configurations

7. **Results timing**: Don't query SSC issues until scan reaches COMPLETED or PUBLISHED state

8. **Error logs**: If scan fails, download logs using `scan_download` with `log` parameter for troubleshooting

## Troubleshooting Version Mismatches

### Problem: Scan Won't Start / Stuck in PENDING State

If your scan gets stuck in PENDING state and never starts, this is typically caused by a version mismatch between the client version used to package the code and the available sensor versions.

### Quick Fix: Use "latest" (Recommended)

The simplest solution is to repackage with `--sc-client-version: "latest"` and omit `--sensor-version`:

**1. Repackage:**
```json
{
  "tool": "fcli_ssc_action_package",
  "parameters": {
    "--sc-client-version": "latest",
    "--source-dir": ".",
    "--output": "package.zip"
  }
}
```

**2. Restart scan:**
```json
{
  "tool": "fcli_sc_sast_scan_start",
  "parameters": {
    "--publish-to": "MyApp:1.0",
    "--file": "package.zip"
  }
}
```

This automatically:
1. Installs the most recent ScanCentral Client (e.g., 25.4.0)
2. Packages with that client version
3. Auto-selects matching sensors when starting the scan

### Advanced: Explicit Version Control

If you need to specify exact versions for both client and sensors:

**Step 1: Check available sensor versions**
```json
{
  "tool": "fcli_sc_sast_sensor_list",
  "parameters": {}
}
```

Look for the `sensorVersion` field in the output:
```json
{
  "sensorVersion": "25.4.0.0047",
  "state": "ACTIVE"
}
```

**Step 2: Determine versions to use**
- Extract major.minor from sensor version (e.g., "25.4" from "25.4.0.0047")
- Use this version for BOTH `--sc-client-version` AND `--sensor-version`

**Step 3: Repackage and restart**
```json
{
  "tool": "fcli_ssc_action_package",
  "parameters": {
    "--sc-client-version": "25.4",
    "--source-dir": ".",
    "--output": "package.zip"
  }
}
```

```json
{
  "tool": "fcli_sc_sast_scan_start",
  "parameters": {
    "--sensor-version": "25.4",
    "--publish-to": "MyApp:1.0",
    "--file": "package.zip"
  }
}
```

### If Scan Already Failed: Diagnosis Workflow

**Check what client version was used in the failed scan:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "embed": "scSastScan",
    "limit": 5
  }
}
```

Look for the `clientVersion` field:
```json
{
  "jobToken": "4f219774-32f6-4971-bed0-7c405c25896a",
  "clientVersion": "24.2.0.0050",  // ⚠️ Mismatch - no 24.2 sensors available
  "sensorVersion": null,
  "jobState": "PENDING"
}
```

Compare with available sensor versions from Step 1. If they don't match, repackage with the correct version.
