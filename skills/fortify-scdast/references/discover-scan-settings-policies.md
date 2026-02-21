# Discover Scan Settings and Policies

## Use Case
Discover available scan configurations and policies before starting a scan. This is essential for understanding what scanning options are available and ensuring scans are configured correctly.

## Keywords
scan settings, scan policy, configuration, CICD token, scan setup, available settings, scan options, policies

## Prerequisites
- Active SSC session (check with `fcli_ssc_session_list`)
- SC-DAST environment configured with settings and policies by administrator

---

## Understanding Settings vs Policies

### Scan Settings
**What They Are:**
- Complete scan configurations including target URLs, authentication, and crawl parameters
- Created and managed by administrators in SC-DAST UI
- Can be referenced by CICD token or numeric ID
- Contain all technical details needed to run a scan

**What They Include:**
- Target URL(s) to scan
- Authentication credentials/macros
- Crawl scope and exclusions
- Performance settings
- SSC application version mapping

### Scan Policies
**What They Are:**
- Pre-defined attack patterns and vulnerability checks
- Determine which security tests are run
- Control scan depth and aggressiveness

**Common Policies:**
- **Standard:** Balanced coverage, reasonable duration
- **Aggressive:** Deep scanning, longer duration
- **Quick:** Fast basic scan, shorter duration
- **Custom:** Organization-specific policies

**Key Point:** Settings define WHERE and HOW to scan, Policies define WHAT to test for.

---

## Discovery Operations

### 1. List All Scan Settings

**Tool:** `fcli_sc_dast_scan_settings_list`

**Parameters:**
```json
{}
```

**Expected Output:**
```json
{
  "data": [
    {
      "id": "12345",
      "name": "Production Web App Settings",
      "cicdToken": "PROD_WEB_APP",
      "url": "https://app.example.com",
      "sscApplicationVersionId": "100",
      "description": "Production website configuration"
    },
    {
      "id": "12346",
      "name": "Staging Environment Settings",
      "cicdToken": "STAGING_WEB_APP",
      "url": "https://staging.example.com",
      "sscApplicationVersionId": "101",
      "description": "Pre-production testing"
    }
  ],
  "pagination": {
    "totalRecords": 2
  }
}
```

**Key Information to Extract:**
- **cicdToken:** Human-readable identifier for automation
- **id:** Numeric identifier
- **url:** Target application URL
- **sscApplicationVersionId:** Where findings will be imported

---

### 2. Filter Settings by Name

**Tool:** `fcli_sc_dast_scan_settings_list`

**Parameters:**
```json
{
  "--server-queries": "name~Production"
}
```

**Use Cases:**
- Find settings for specific application: `"name~MyApp"`
- Find settings for environment: `"name~Production"`, `"name~Staging"`
- Find settings by team: `"name~TeamA"`

---

### 3. Get Specific Setting Details

**By CICD Token (Recommended for Automation):**
```json
{
  "tool": "fcli_sc_dast_scan_settings_get",
  "parameters": {
    "scanSettingsCicdTokenOrId": "PROD_WEB_APP"
  }
}
```

**By Numeric ID:**
```json
{
  "tool": "fcli_sc_dast_scan_settings_get",
  "parameters": {
    "scanSettingsCicdTokenOrId": "12345"
  }
}
```

**Expected Output:**
```json
{
  "id": "12345",
  "name": "Production Web App Settings",
  "cicdToken": "PROD_WEB_APP",
  "url": "https://app.example.com",
  "sscApplicationVersionId": "100",
  "allowedHosts": ["app.example.com", "*.example.com"],
  "excludedPaths": ["/admin/*", "/api/internal/*"],
  "loginMacro": "98765",
  "maxCrawlDuration": "3600",
  "maxPages": "10000",
  "description": "Production website configuration"
}
```

**Detailed Information Available:**
- Target URLs and allowed hosts
- Excluded paths and patterns
- Authentication configuration
- Crawl limits and timeouts
- SSC mapping

---

### 4. List All Scan Policies

**Tool:** `fcli_sc_dast_scan_policy_list`

**Parameters:**
```json
{}
```

**Expected Output:**
```json
{
  "data": [
    {
      "id": "1",
      "name": "Standard",
      "description": "Balanced security testing with reasonable scan time",
      "aggressiveness": "Medium",
      "estimatedDuration": "2-4 hours"
    },
    {
      "id": "2",
      "name": "Aggressive",
      "description": "Comprehensive security testing with longer scan time",
      "aggressiveness": "High",
      "estimatedDuration": "4-8 hours"
    },
    {
      "id": "3",
      "name": "Quick",
      "description": "Fast basic security scan",
      "aggressiveness": "Low",
      "estimatedDuration": "1-2 hours"
    }
  ],
  "pagination": {
    "totalRecords": 3
  }
}
```

---

### 5. Get Specific Policy Details

**By Policy Name:**
```json
{
  "tool": "fcli_sc_dast_scan_policy_get",
  "parameters": {
    "policyNameOrId": "Standard"
  }
}
```

**By Policy ID:**
```json
{
  "tool": "fcli_sc_dast_scan_policy_get",
  "parameters": {
    "policyNameOrId": "1"
  }
}
```

**Expected Output:**
```json
{
  "id": "1",
  "name": "Standard",
  "description": "Balanced security testing with reasonable scan time",
  "aggressiveness": "Medium",
  "estimatedDuration": "2-4 hours",
  "vulnerabilityChecks": [
    "SQL Injection",
    "Cross-Site Scripting (XSS)",
    "Authentication Issues",
    "Session Management",
    "Information Disclosure",
    "..."
  ],
  "maxAttacksPerPage": "100",
  "crawlDepth": "10"
}
```

---

### 6. List Available Sensors

**Tool:** `fcli_sc_dast_sensor_list`

**Parameters:**
```json
{}
```

**Expected Output:**
```json
{
  "data": [
    {
      "id": "sensor-1",
      "name": "DAST Sensor Pool 1",
      "status": "Enabled",
      "capacity": "5",
      "activeScans": "2",
      "version": "23.2.0"
    },
    {
      "id": "sensor-2",
      "name": "DAST Sensor Pool 2",
      "status": "Enabled",
      "capacity": "10",
      "activeScans": "7",
      "version": "23.2.0"
    }
  ],
  "pagination": {
    "totalRecords": 2
  }
}
```

**When to Check:**
- Before starting scans to verify capacity
- Troubleshooting "no sensors available" errors
- Load balancing across sensor pools

---

### 7. Get Specific Sensor Details

**Tool:** `fcli_sc_dast_sensor_get`

**Parameters:**
```json
{
  "sensorNameOrId": "sensor-1"
}
```

**Expected Output:**
```json
{
  "id": "sensor-1",
  "name": "DAST Sensor Pool 1",
  "status": "Enabled",
  "capacity": "5",
  "activeScans": "2",
  "version": "23.2.0",
  "lastHeartbeat": "2026-02-16T14:30:00Z",
  "queuedScans": "1"
}
```

---

## Complete Discovery Workflow

Before starting any scan, run this discovery workflow:

```json
[
  {
    "step": "1. Check authentication",
    "tool": "fcli_ssc_session_list",
    "parameters": {
      "refresh-cache": true
    }
  },
  {
    "step": "2. List all scan settings",
    "tool": "fcli_sc_dast_scan_settings_list",
    "parameters": {}
  },
  {
    "step": "3. Get details of target settings",
    "tool": "fcli_sc_dast_scan_settings_get",
    "parameters": {
      "scanSettingsCicdTokenOrId": "PROD_WEB_APP"
    }
  },
  {
    "step": "4. List available policies",
    "tool": "fcli_sc_dast_scan_policy_list",
    "parameters": {}
  },
  {
    "step": "5. Get details of selected policy",
    "tool": "fcli_sc_dast_scan_policy_get",
    "parameters": {
      "policyNameOrId": "Standard"
    }
  },
  {
    "step": "6. Check sensor availability",
    "tool": "fcli_sc_dast_sensor_list",
    "parameters": {}
  }
]
```

---

## CICD Token vs ID

### CICD Token (Recommended for Automation)
**Advantages:**
- Human-readable: `"PROD_WEB_APP"` vs `"12345"`
- Stable across environments
- Self-documenting in scripts
- Less likely to change

**Example:**
```json
{
  "tool": "fcli_sc_dast_scan_start",
  "parameters": {
    "--settings": "PROD_WEB_APP",
    "--name": "Weekly Security Scan"
  }
}
```

### Numeric ID
**Advantages:**
- Guaranteed unique
- Slightly faster lookup

**Example:**
```json
{
  "tool": "fcli_sc_dast_scan_start",
  "parameters": {
    "--settings": "12345",
    "--name": "Weekly Security Scan"
  }
}
```

**Best Practice:** Use CICD tokens for automation and scripts, IDs are fine for one-off scans.

---

## Pagination Handling

When listing settings/policies with many results:

### Step 1: Initial List
```json
{
  "tool": "fcli_sc_dast_scan_settings_list",
  "parameters": {}
}
```

**If Response has `totalRecords: null`:**
```json
{
  "data": [],
  "pagination": {
    "totalRecords": null,
    "job_token": "abc123..."
  }
}
```

### Step 2: Wait for Job
```json
{
  "tool": "fcli_sc_dast_mcp_job",
  "parameters": {
    "operation": "wait",
    "job_token": "abc123..."
  }
}
```

### Step 3: Retry List
```json
{
  "tool": "fcli_sc_dast_scan_settings_list",
  "parameters": {}
}
```

---

## Troubleshooting

### "No settings found"
**Cause:** No scan settings configured or filtered out  
**Solution:**
1. Remove filters and list all settings
2. Ask administrator to create settings in SC-DAST UI
3. Verify access permissions

### "Policy not found"
**Cause:** Invalid policy name or ID  
**Solution:**
1. List all available policies
2. Verify policy name spelling (case-sensitive)
3. Use policy ID instead of name

### "No sensors available"
**Cause:** All sensors at capacity or disabled  
**Solution:**
1. List sensors to check status
2. Wait for running scans to complete
3. Contact administrator to enable more sensors

### Settings list returns empty with pagination
**Cause:** Job not completed  
**Solution:** Use `mcp_job` with `operation: "wait"` and `job_token`

---

## Use Cases

### Find Settings for Specific Application
```json
{
  "tool": "fcli_sc_dast_scan_settings_list",
  "parameters": {
    "--server-queries": "name~MyApplication"
  }
}
```

### Verify Settings Target Correct URL
```json
[
  {
    "tool": "fcli_sc_dast_scan_settings_get",
    "parameters": {
      "scanSettingsCicdTokenOrId": "PROD_WEB_APP"
    }
  }
]
```

Look for `url` field in response to verify target.

### Choose Appropriate Policy
```json
{
  "tool": "fcli_sc_dast_scan_policy_list",
  "parameters": {}
}
```

Review `estimatedDuration` and `aggressiveness` to choose policy that fits time window.

### Check Sensor Capacity Before Starting Scan
```json
{
  "tool": "fcli_sc_dast_sensor_list",
  "parameters": {}
}
```

Look for sensors with `activeScans < capacity` to ensure availability.

---

## Best Practices

**DO:**
- ✅ Always discover settings before starting scans
- ✅ Use CICD tokens for automation and scripts
- ✅ Verify settings target correct URL before scanning
- ✅ Check sensor capacity before starting multiple scans
- ✅ Review policy details to understand scan scope
- ✅ Use descriptive CICD tokens (e.g., `PROD_API_V2`)
- ✅ Filter settings list when many configurations exist
- ✅ Document which settings/policies are used for each application

**DO NOT:**
- ❌ Hardcode numeric IDs in automation scripts (use CICD tokens)
- ❌ Start scans without verifying settings exist
- ❌ Assume policy names are consistent across environments
- ❌ Ignore sensor capacity warnings
- ❌ Use wrong settings for application (verify URL)
- ❌ Forget to check SSC mapping in settings
- ❌ Skip discovery workflow to "save time"

---

## Integration with Scan Start

After discovery, use gathered information to start scan:

```json
[
  {
    "step": "1. Discover settings",
    "tool": "fcli_sc_dast_scan_settings_list",
    "parameters": {
      "--server-queries": "name~Production"
    }
  },
  {
    "step": "2. Verify settings details",
    "tool": "fcli_sc_dast_scan_settings_get",
    "parameters": {
      "scanSettingsCicdTokenOrId": "PROD_WEB_APP"
    }
  },
  {
    "step": "3. List policies",
    "tool": "fcli_sc_dast_scan_policy_list",
    "parameters": {}
  },
  {
    "step": "4. Start scan with discovered settings and policy",
    "tool": "fcli_sc_dast_scan_start",
    "parameters": {
      "--settings": "PROD_WEB_APP",
      "--name": "Weekly Production Scan - Feb 16",
      "--policy": "Standard",
      "--overwrite-scan-mode": true
    }
  }
]
```

This ensures you're using correct settings and appropriate policy for your scan needs.

