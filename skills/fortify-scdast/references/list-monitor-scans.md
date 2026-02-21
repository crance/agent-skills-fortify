# List and Monitor Scans

## Use Case
List all scans, filter by criteria, monitor progress, and manage scan execution including pause/resume capabilities.

## Keywords
list scans, monitor progress, check status, pause scan, resume scan, scan management, view scans, track scans

## Prerequisites
- Active SSC session (check with `fcli_ssc_session_list`)

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

### 2. Filter Scans by Status (Server-Side)

Use `server-queries` parameter for efficient server-side filtering:

**Running Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Running"
  }
}
```

**Completed Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Complete"
  }
}
```

**Paused Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Paused"
  }
}
```

**Failed Scans:**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Failed"
  }
}
```

---

### 3. Filter by Scan Name (Contains Match)

**Tool:** `fcli_sc_dast_scan_list`

**Parameters:**
```json
{
  "--server-queries": "name~Production"
}
```

**Explanation:** The `~` operator performs a "contains" match. This will return all scans with "Production" in their name.

**Examples:**
- `"name~Security"` → Scans containing "Security"
- `"name~Feb"` → Scans containing "Feb"
- `"name~MyApp"` → Scans containing "MyApp"

---

### 4. Multiple Filters (Combined)

Combine multiple filters with comma separation:

```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "--server-queries": "status=Complete,name~Production"
  }
}
```

**Example Combinations:**
- `"status=Running,name~Security"` → Running scans containing "Security"
- `"status=Complete,policy=Standard"` → Completed scans using Standard policy

---

### 5. Client-Side Filtering (Complex Expressions)

Use `query` parameter for complex filtering with SpEL expressions:

**Example: Filter by scan version**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": "scanVersion > 3"
  }
}
```

**Example: Multiple conditions**
```json
{
  "tool": "fcli_sc_dast_scan_list",
  "parameters": {
    "query": "status == 'Complete' && scanVersion > 5"
  }
}
```

**When to Use:**
- Server-side (`--server-queries`): Preferred for performance, simple key-value filtering
- Client-side (`query`): Complex expressions not supported by server-side filtering

---

### 6. Get Specific Scan Details

**Tool:** `fcli_sc_dast_scan_get`

**Parameters:**
```json
{
  "scanIds": "12345"
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

**Tool:** `fcli_sc_dast_scan_wait_for`

**Wait Until Complete:**
```json
{
  "tool": "fcli_sc_dast_scan_wait_for",
  "parameters": {
    "scanIds": "12345",
    "--until": "status=Complete",
    "--timeout": "7200"
  }
}
```

**Wait Until Failed:**
```json
{
  "tool": "fcli_sc_dast_scan_wait_for",
  "parameters": {
    "scanIds": "12345",
    "--until": "status=Failed",
    "--timeout": "7200"
  }
}
```

**Wait While Running:**
```json
{
  "tool": "fcli_sc_dast_scan_wait_for",
  "parameters": {
    "scanIds": "12345",
    "--while": "status=Running",
    "--timeout": "7200"
  }
}
```

**Parameters:**
- `--until`: Continue until condition becomes true
- `--while`: Continue while condition remains true
- `--timeout`: Maximum wait time in seconds (7200 = 2 hours)

**Behavior:**
- Tool polls scan status periodically
- Returns when condition met or timeout reached
- Useful for automation and CI/CD pipelines

---

### 8. Pause Running Scan

**Tool:** `fcli_sc_dast_scan_pause`

**Parameters:**
```json
{
  "scanIds": "12345"
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
  "scanIds": "12345"
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
      "--server-queries": "status=Running"
    }
  },
  {
    "step": "2. Get details of specific scan",
    "tool": "fcli_sc_dast_scan_get",
    "parameters": {
      "scanIds": "12345"
    }
  },
  {
    "step": "3. Pause scan for maintenance",
    "tool": "fcli_sc_dast_scan_pause",
    "parameters": {
      "scanIds": "12345"
    }
  },
  {
    "step": "4. Verify paused",
    "tool": "fcli_sc_dast_scan_get",
    "parameters": {
      "scanIds": "12345"
    }
  },
  {
    "step": "5. Resume after maintenance",
    "tool": "fcli_sc_dast_scan_resume",
    "parameters": {
      "scanIds": "12345"
    }
  },
  {
    "step": "6. Monitor until complete",
    "tool": "fcli_sc_dast_scan_wait_for",
    "parameters": {
      "scanIds": "12345",
      "--until": "status=Complete",
      "--timeout": "7200"
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
2. Verify filter syntax (e.g., `status=Running` not `status:Running`)
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
- ✅ Use `scan_wait_for` instead of manual polling
- ✅ Set realistic timeouts (DAST scans take 30min-2hrs typically)
- ✅ Use pause/resume for maintenance windows
- ✅ Monitor failed scans regularly
- ✅ Use descriptive scan names for easy filtering

**DO NOT:**
- ❌ Poll status manually when `scan_wait_for` is available
- ❌ Use client-side filtering for large datasets
- ❌ Pause scans unnecessarily (affects completion time)
- ❌ Set very short timeouts for `scan_wait_for`
- ❌ Ignore pagination when `totalRecords` is null

