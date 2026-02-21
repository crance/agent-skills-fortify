# Run DAST Scan Workflow

## Use Case
Complete workflow for running a dynamic application security testing (DAST) scan from discovery to viewing results in SSC.

## Keywords
run scan, DAST, start scan, web scan, application scan, dynamic scan, security scan, execute scan, launch scan

## Prerequisites
- Active SSC session (check with `fcli_ssc_session_list`)
- Available scan settings and policies configured in SC-DAST
- **Scan settings configured to auto-publish results to SSC** (configured by administrator)
- Target web application URL configured in scan settings

## Workflow Steps

### Step 1: Check Authentication
**Tool:** `fcli_ssc_session_list`

**Parameters:**
```json
{
  "refresh-cache": true
}
```

**Expected Output:**  
Active session information with status. If expired, ask user to run:
```bash
fcli ssc session login --url <SSC_URL> -u <username> -p <password>
```

---

### Step 2: Discover Scan Settings
**Tool:** `fcli_sc_dast_scan_settings_list`

**Parameters:**
```json
{}
```

**Expected Output:**  
List of available scan settings with:
- CICD tokens (e.g., "MY_WEB_APP_SETTINGS")
- Numeric IDs
- Target URLs
- Configuration details

**Key Information to Extract:**
- Settings ID or CICD token to use for scan
- Target URL being scanned
- SSC application version mapping

---

### Step 3: Discover Scan Policies (Optional but Recommended)
**Tool:** `fcli_sc_dast_scan_policy_list`

**Parameters:**
```json
{}
```

**Expected Output:**  
List of available policies such as:
- "Standard" - Balanced coverage
- "Aggressive" - Deep scanning
- "Quick" - Fast basic scan
- Custom policies defined by organization

---

### Step 4: Start the Scan
**Tool:** `fcli_sc_dast_scan_start`

**Parameters:**
```json
{
  "--settings": "MY_WEB_APP_SETTINGS",
  "--name": "Weekly Security Scan - Feb 2026",
  "--policy": "Standard",
  "--overwrite-scan-mode": true
}
```

**Parameter Details:**
- `--settings` (**required**): CICD token or ID from Step 2
- `--name` (**required**): Descriptive scan name
- `--policy` (optional): Policy name or ID from Step 3
- `--overwrite-scan-mode` or `--priority-scan-mode` (optional): Boolean flags for scan mode (mutually exclusive)
- `--login-macro` (optional): Login macro ID if authenticated scanning required

**Expected Output:**  
Scan created successfully with:
- Scan ID (save this for monitoring)
- Initial status (typically "Pending" or "Running")
- Scan metadata

---

### Step 5: Monitor Scan Progress
**Tool:** `fcli_sc_dast_scan_wait_for`

**Parameters:**
```json
{
  "scanIds": "<scan-id-from-step-4>",
  "--until": "status=Complete",
  "--timeout": "7200"
}
```

**Parameter Details:**
- `scanIds`: Scan ID from Step 4
- `--until`: Condition to wait for ("status=Complete" or "status=Failed")
- `--timeout`: Maximum wait time in seconds (7200 = 2 hours)

**Expected Behavior:**
- Tool will poll scan status until condition met or timeout
- DAST scans can take 30 minutes to several hours depending on application size
- Adjust timeout based on expected scan duration

**Alternative: Check Status Without Blocking**
```json
{
  "tool": "fcli_sc_dast_scan_get",
  "parameters": {
    "scanIds": "<scan-id>"
  }
}
```

---

### Step 6: View Results in SSC

Once scan completes, findings automatically appear in SSC (if settings are configured for auto-publish).

Use SSC skill tools to view findings:

**Count Issues by Severity:**
```json
{
  "tool": "fcli_ssc_issue_count",
  "parameters": {
    "appversion": "<application:version>",
    "by": "Folder"
  }
}
```

**List Issues with Details:**
```json
{
  "tool": "fcli_ssc_issue_list",
  "parameters": {
    "appversion": "<application:version>",
    "filter": "CATEGORY:Security",
    "embed": "details"
  }
}
```

**Note:** If results don't appear in SSC, verify that scan settings are configured to auto-publish. Contact administrator if needed.

---

## Complete Example

```json
[
  {
    "tool": "fcli_ssc_session_list",
    "parameters": { "refresh-cache": true }
  },
  {
    "tool": "fcli_sc_dast_scan_settings_list",
    "parameters": {}
  },
  {
    "tool": "fcli_sc_dast_scan_policy_list",
    "parameters": {}
  },
  {
    "tool": "fcli_sc_dast_scan_start",
    "parameters": {
      "--settings": "PRODUCTION_WEB_APP",
      "--name": "Weekly Production Scan - Week 7",
      "--policy": "Standard",
      "--overwrite-scan-mode": true
    }
  },
  {
    "tool": "fcli_sc_dast_scan_wait_for",
    "parameters": {
      "scanIds": "12345",
      "--until": "status=Complete",
      "--timeout": "7200"
    }
  },
  {
    "tool": "fcli_ssc_issue_count",
    "parameters": {
      "appversion": "MyWebApp:1.0",
      "by": "Folder"
    }
  }
]
```

---

## Troubleshooting

### "Settings not found"
**Cause:** Invalid CICD token or settings ID  
**Solution:** Run `scan_settings_list` to verify available settings

### "Policy not found"
**Cause:** Invalid policy name or ID  
**Solution:** Run `scan_policy_list` to verify available policies

### "Scan hung" / Status stuck at "Running"
**Cause:** Scan encountered issues or application not responding  
**Solution:**
1. Use `scan_pause` to pause the scan
2. Investigate application/network issues
3. Either `scan_resume` to continue or restart with different settings

### "Results not appearing in SSC"
**Cause:** Scan settings not configured to auto-publish to SSC  
**Solution:** Contact administrator to verify scan settings include SSC application version mapping and auto-publish configuration

### "Session expired"
**Cause:** SSC session timed out during long-running scan  
**Solution:** Ask user to re-login: `fcli ssc session login --url <URL> -u <user> -p <pass>`

### Timeout occurred before scan completed
**Cause:** Scan taking longer than expected  
**Solution:**
1. Use `scan_get` to check current status
2. If still running, increase timeout and retry `scan_wait_for`
3. Consider pausing/resuming or using server-side monitoring

---

## Tips for Success

1. **Verify Settings:** Always list available settings before starting a scan
2. **Confirm Auto-Publish:** Ensure scan settings are configured to publish to SSC automatically
3. **Use Descriptive Names:** Include date/version in scan names for tracking
4. **Set Realistic Timeouts:** DAST scans typically take 30min-2hrs depending on app size
5. **Monitor Progress:** Use `scan_get` periodically if not using `scan_wait_for`
6. **Check SSC:** Verify findings appear in SSC after scan completes
7. **Leverage Pause/Resume:** Use pause capability for maintenance windows or to manage scan execution
