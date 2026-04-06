# List and Monitor Scans

## Use Case
List all scans, filter by criteria, monitor progress, and manage scan execution including pause/resume capabilities.

## Keywords
list scans, monitor progress, check status, pause scan, resume scan, scan management, view scans, track scans

## Prerequisites
- Active SSC session (check with `fcli_ssc_session_list`)

## Contents
- [Workflow Operations](#workflow-operations) — list, filter, pause, resume scans
- [Complete Example: Monitor and Manage Workflow](#complete-example-monitor-and-manage-workflow)
- [Pagination Handling](#pagination-handling)
- [Troubleshooting](#troubleshooting)
- [Tips for Effective Monitoring](#tips-for-effective-monitoring)
- [Best Practices](#best-practices)

---

## Workflow Operations

### 1. List All Scans

**Tool:** `fcli_sc_dast_scan_list`

**Parameters:**
```json
{}
```

**Expected Output:**
List of all scans with:
- Scan ID
- Scan name
- Status (Pending, Running, Complete, Failed, Paused)
- Start time
- Target URL
- Policy used

**Example Response:**
```json
{
  "data": [
    {
      "id": "12345",
      "name": "Weekly Security Scan",
      "status": "Running",
      "startTime": "2026-02-16T10:30:00Z",
      "policy": "Standard"
    },
    {
      "id": "12344",
      "name": "Monthly Audit Scan",
      "status": "Complete",
      "startTime": "2026-02-15T08:00:00Z",
      "policy": "Aggressive"
    }
  ],
  "pagination": {
    "totalRecords": 2
  }
}
```

---

### 2. Filter Scans by Status

Use the client-side `query` parameter with the `scanStatusTypeDescription` regex field:

**Running Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Running" }
  }
}
```

**Completed Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Complete" }
  }
}
```

**Paused Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Paused" }
  }
}
```

**Failed Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Failed" }
  }
}
```

> **Note:** The `--server-queries` `status=X` syntax does not filter by status (confirmed non-functional). Use client-side `query.scanStatusTypeDescription` instead.

---

### 3. Filter by Scan Name (Contains Match)

**Tool:** `fcli_sc_dast_scan_list`

**Parameters:**
```json
{
  "query": { "name": ".*Production.*" }
}
```

**Explanation:** The `query.name` field accepts a regex pattern. Use `".*keyword.*"` for a contains match.

**Examples:**
- `{ "name": ".*Security.*" }` → Scans containing "Security"
- `{ "name": ".*Feb.*" }` → Scans containing "Feb"
- `{ "name": ".*MyApp.*" }` → Scans containing "MyApp"

---

### 4. Multiple Filters (Combined)

Combine status and name filters using the client-side `query` parameter:

```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Complete", "name": ".*Production.*" }
  }
}
```

**Example Combinations:**
- Status Running + name contains "Security": `"query": { "scanStatusTypeDescription": "Running", "name": ".*Security.*" }`
- Name only: `"query": { "name": ".*Production.*" }`
- Status + application: `"query": { "scanStatusTypeDescription": "Complete", "applicationName": ".*MyApp.*" }`

---

### 5. Client-Side Filtering (Complex Expressions)

Use `query` parameter for client-side filtering. The `query` value must be a `FcliScDastScanListQuery` JSON object. All string fields accept regex patterns.

**Available query fields:** `applicationName`, `applicationVersionName`, `criticalCount`, `highCount`, `id`, `lowCount`, `mediumCount`, `name`, `scanStatusTypeDescription`

**Example: Filter by scan status**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Running" }
  }
}
```

**Example: Multiple conditions**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": { "scanStatusTypeDescription": "Complete", "name": ".*Production.*" }
  }
}
```

**When to Use:**
- Client-side (`query`): Primary approach — regex matching on any field, multiple field conditions combined
- Server-side (`--server-queries "searchText=keyword"`): Only for full-text keyword search across all scan fields

---

### 6. Get Specific Scan Details

**Tool:** `fcli_sc_dast_scan_get`

**Parameters:**
```json
{
  "scanId": "12345"
}
```

**Expected Output:**
Detailed information including:
- Full scan configuration
- Current status and progress
- Findings summary
- Error messages (if any)
- Timestamps (start, end, last update)

---

### 7. Monitor Scan Progress (Blocking)

> ⚠️ **MCP Schema Bug:** The MCP schema requires both `--until` and `--while` simultaneously, but the underlying CLI treats them as mutually exclusive (`-u` OR `-w`). Providing both causes `exitCode: 2`. Use `scan_get` polling (Operation 6) as the reliable alternative.

**Recommended: Poll with `scan_get` (fully reliable)**
```json
{
  "tool": "fcli_sc_dast_scan_get",
  "parameters": {
    "scanId": "12345"
  }
}
```
Check `scanStatusTypeDescription`. Repeat every 30–60 seconds until it reaches `Complete`, `Failed`, or another terminal state.

**Terminal states:** `Complete`, `ForcedComplete`, `Interrupted`, `FailedToStart`, `ImportScanResultsFailed`, `FailedToImportScanFile`

---

**Best-Effort: `fcli_sc_dast_scan_wait_for` (may return exitCode 2)**

Wait until Complete:
```json
{
  "tool": "fcli_sc_dast_scan_wait_for",
  "parameters": {
    "scanIds": "12345",
    "--until": "any-match",
    "--while": "any-match",
    "--any-state": "Complete",
    "--timeout": "2h"
  }
}
```

Wait until Failed:
```json
{
  "tool": "fcli_sc_dast_scan_wait_for",
  "parameters": {
    "scanIds": "12345",
    "--until": "any-match",
    "--while": "any-match",
    "--any-state": "Failed",
    "--timeout": "2h"
  }
}
```

**Parameters:**
- `--until`: `"any-match"` or `"all-match"` — wait **until** scans reach the target state
- `--while`: `"any-match"` or `"all-match"` — required by MCP schema (mutually exclusive with `--until` in the CLI)
- `--any-state`: Target state value (e.g., `"Complete"`, `"Failed"`, `"Running"`)
- `--timeout`: Maximum wait time (e.g., `"1h"`, `"2h"`)

---

### 8. Pause Running Scan

**Tool:** `fcli_sc_dast_scan_pause`

**Parameters:**
```json
{
  "scanId": "12345"
}
```

**When to Use:**
- Application maintenance window started
- Server load too high
- Need to investigate application behavior
- Temporary network issues

**Expected Output:**
```json
{
  "id": "12345",
  "status": "Paused",
  "message": "Scan paused successfully"
}
```

**Note:** Paused scans can be resumed later from the same point.

---

### 9. Resume Paused Scan

**Tool:** `fcli_sc_dast_scan_resume`

**Parameters:**
```json
{
  "scanId": "12345"
}
```

**Expected Output:**
```json
{
  "id": "12345",
  "status": "Running",
  "message": "Scan resumed successfully"
}
```

**Note:** Scan continues from where it was paused, no need to restart.

---

## Complete Example: Monitor and Manage Workflow

```json
[
  {
    "step": "1. List all running scans",
    "tool": "fcli_sc_dast_scan_list",
    "parameters": {
      "query": { "scanStatusTypeDescription": "Running" }
    }
  },
  {
    "step": "2. Get details of specific scan",
    "tool": "fcli_sc_dast_scan_get",
    "parameters": {
      "scanId": "12345"
    }
  },
  {
    "step": "3. Pause scan for maintenance",
    "tool": "fcli_sc_dast_scan_pause",
    "parameters": {
      "scanId": "12345"
    }
  },
  {
    "step": "4. Verify paused",
    "tool": "fcli_sc_dast_scan_get",
    "parameters": {
      "scanId": "12345"
    }
  },
  {
    "step": "5. Resume after maintenance",
    "tool": "fcli_sc_dast_scan_resume",
    "parameters": {
      "scanId": "12345"
    }
  },
  {
    "step": "6. Monitor until complete",
    "comment": "Recommended: poll scan_get until scanStatusTypeDescription is Complete or Failed",
    "tool": "fcli_sc_dast_scan_get",
    "parameters": {
      "scanId": "12345"
    }
  }
]
```

---

## Pagination Handling

When listing many scans, you may encounter pagination with `totalRecords: null`:

### Step 1: Initial List Call
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {}
}
```

**Response:**
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

### Step 3: Retry List Call
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {}
}
```

**Response will now include full results:**
```json
{
  "data": [...],
  "pagination": {
    "totalRecords": 50
  }
}
```

---

## Troubleshooting

### "Cannot pause scan"
**Cause:** Scan status is not "Running"  
**Solution:** Check scan status with `scan_get`. Only running scans can be paused.

### "Cannot resume scan"
**Cause:** Scan status is not "Paused"  
**Solution:** Verify scan was paused. Check status with `scan_get`.

### "No scans found" with filters
**Cause:** Filter criteria too restrictive or no matching scans  
**Solution:**
1. Try without filters to see all scans
2. Use `query.scanStatusTypeDescription` for status filtering (e.g., `"query": { "scanStatusTypeDescription": "Running" }`) — `--server-queries "status=X"` does not work
3. Check for typos in scan names

### Pagination returns empty results
**Cause:** Job not completed yet  
**Solution:** Use `mcp_job` with `operation: "wait"` and `job_token` from response.

### Timeout waiting for scan
**Cause:** Scan taking longer than expected  
**Solution:**
1. Increase timeout value
2. Use `scan_get` to check progress
3. Consider pausing and investigating if stuck

---

## Tips for Effective Monitoring

### Regular Status Checks
**Polling Pattern:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Running"
  }
}
```
Run every 5-10 minutes to monitor active scans.

### Identify Long-Running Scans
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": "status == 'Running' && startTime < '2026-02-16T00:00:00Z'"
  }
}
```

### Track Failed Scans
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Failed"
  }
}
```
Review failed scans regularly to identify configuration issues.

### Organize by Project
Use consistent naming conventions and filter by name:
```json
{
  "--server-queries": "name~ProjectA"
}
```

---

## Best Practices

**DO:**
- ✅ Use server-side filtering (`--server-queries`) for better performance
- ✅ Use `scan_get` polling to monitor scan progress (reliable MCP method)
- ✅ Set realistic timeouts (DAST scans take 30min-2hrs typically)
- ✅ Use pause/resume for maintenance windows
- ✅ Monitor failed scans regularly
- ✅ Use descriptive scan names for easy filtering

**DO NOT:**
- ❌ Rely on `scan_wait_for` via MCP — it may return exitCode 2 due to a schema bug (use `scan_get` polling instead)
- ❌ Use client-side filtering for large datasets
- ❌ Pause scans unnecessarily (affects completion time)
- ❌ Set very short timeouts for `scan_wait_for`
- ❌ Ignore pagination when `totalRecords` is null

