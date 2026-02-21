# List and Monitor Scans Workflow

## Use Case
View scan history, check status of existing scans, monitor scan progress, and download scan artifacts.

## Keywords
list scans, scan history, check status, monitor scan, scan progress, running scans, view scans, find scans, scan details, track scan

## Prerequisites
- Active SSC session with ScanCentral client auth token

## Workflow Operations

### Operation 1: List All Scans
View scan history with optional filtering:

**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{}
```

**Basic listing** (no filters):
- Returns recent scans
- Default sorting (most recent first)
- Limited results (use pagination for more)

**Expected output:** Array of scan objects with basic information

### Operation 2: List Scans with Filters
Use client-side filtering to find specific scans:

#### Filter by Status
**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "query": {"status": "RUNNING"}
}
```

**Available status values:**
- `QUEUED` - Waiting for sensor
- `RUNNING` - Scan in progress
- `COMPLETED` - Scan finished
- `PUBLISHED` - Results published to SSC
- `FAILED` - Scan encountered error
- `CANCELED` - Scan was canceled

#### Filter by Application Version
**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "query": {"publishToApplicationVersion": "MyApp:1.0"}
}
```

#### Combine Multiple Filters
**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "query": {"status": "COMPLETED", "publishToApplicationVersion": "MyApp:.*"}
}
```

**Query syntax:**
- JSON object format with field-value pairs
- Supports regex patterns in values
- Multiple fields act as AND logic
- Pass as JSON parameter (not command-line flag)

### Operation 3: Get Detailed Scan Information
Include embedded scan details in the listing:

**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "--embed": "scSastScan"
}
```

**Embedded data includes:**
- Scan configuration settings
- Sensor version used
- Publish target details
- Timing information
- Error messages (if failed)

**When to use `embed`:**
- Need troubleshooting information
- Want to see scan configuration
- Analyzing historical scan patterns

### Operation 4: Check Specific Scan Status
Monitor a specific scan using its job-token:

**Tool:** `fcli_sc_sast_scan_status`

**Parameters:**
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Parameter Details:**
- `scanJobToken` (**required**): UUID from scan start operation

**Expected output:** Current scan status with state information

**Use for:**
- Polling individual scan progress
- Verifying scan completion
- Checking error status

**Response includes:**
- Current status (QUEUED, RUNNING, COMPLETED, etc.)
- Progress percentage (if available)
- Start/end times
- Error messages (if failed)

### Operation 5: Wait for Scan Completion
Block until a scan reaches desired state:

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
- `scanJobTokens` (**required**): UUID of scan to monitor
- `--until` (**required**): Target state to wait for
  - `"COMPLETED"` - Wait for scan analysis to finish
  - `"PUBLISHED"` - Wait for results to publish to SSC
  - `"FAILED"` - Wait for failure (useful in error handling)
- `--timeout` (**required**): Maximum wait time in seconds

**Timeout recommendations:**
- Small projects: 900 seconds (15 minutes)
- Medium projects: 1800 seconds (30 minutes)
- Large projects: 3600 seconds (60 minutes)
- Enterprise projects: 7200 seconds (2 hours)

**Expected behavior:**
- Polls scan status periodically
- Returns when target state reached
- Throws error if timeout exceeded
- Returns error if scan fails

### Operation 6: Download Scan Artifacts
Retrieve FPR files or logs after scan completion:

#### Download FPR File
**Tool:** `fcli_sc_sast_scan_download`

**Parameters:**
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000",
  "--fpr": "results.fpr"
}
```

**When to download FPR:**
- Need offline analysis in Audit Workbench
- Archiving scan results
- Importing to non-SSC systems
- Detailed analysis tools

#### Download Scan Logs
**Tool:** `fcli_sc_sast_scan_download`

**Parameters:**
```json
{
  "scanJobToken": "550e8400-e29b-41d4-a716-446655440000",
  "--log": "scan.log"
}
```

**When to download logs:**
- Troubleshooting failed scans
- Understanding scan performance
- Debugging configuration issues
- Support ticket creation

**Prerequisites for download:**
- Scan must be in COMPLETED or FAILED state
- Cannot download from RUNNING or QUEUED scans

## Pagination for Large Result Sets

When dealing with many scans, use pagination:

### Step 1: Initial Request
**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "pagination-offset": 0
}
```

**Parameter Details:**
- `pagination-offset`: Starting offset for results (default: 0)

### Step 2: Check Response
Response includes pagination information:
```json
{
  "data": [...],
  "pagination": {
    "offset": 50,
    "hasMore": true
  }
}
```

### Step 3: Fetch Next Page
**Tool:** `fcli_sc_sast_scan_list`

**Parameters:**
```json
{
  "pagination-offset": 50
}
```

### Step 4: Continue
Repeat with incremented offset until `hasMore: false`

## Monitoring Multiple Scans

### Scenario: Track All Running Scans
**1. List all running scans:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {"status": "RUNNING"},
    "--embed": "scSastScan"
  }
}
```

Response shows multiple scans with job-tokens

**2. Check each scan individually (if needed):**
```json
{
  "tool": "fcli_sc_sast_scan_status",
  "parameters": {
    "scanJobToken": "token-1"
  }
}
```

Repeat for token-2, token-3, etc.

**3. Wait for specific scans:**
```json
{
  "tool": "fcli_sc_sast_scan_wait_for",
  "parameters": {
    "scanJobTokens": "token-1",
    "--until": "COMPLETED",
    "--timeout": "3600"
  }
}
```

### Scenario: Find Failed Scans
**List failed scans with details:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {"status": "FAILED"},
    "--embed": "scSastScan"
  }
}
```

**Download logs for investigation:**
```json
{
  "tool": "fcli_sc_sast_scan_download",
  "parameters": {
    "scanJobToken": "failed-scan-token",
    "--log": "failure-analysis.log"
  }
}
```

### Scenario: View Scans for Specific Application
**List all scans published to application:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {"publishToApplicationVersion": "WebApp:.*"}
  }
}
```

**Filter to completed scans only:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {
      "status": "COMPLETED",
      "publishToApplicationVersion": "WebApp:.*"
    }
  }
}
```

## Error Handling

### No Scans Found
```
Error: Empty result set
Possible causes:
- No scans exist yet
- Filters too restrictive
- Session doesn't have access to scans

Recovery:
- Remove filters and try again
- Verify SSC session permissions
- Check scan history in SSC UI
```

### Invalid Job Token
```
Error: Scan not found for job-token
Possible causes:
- Token typed incorrectly
- Scan deleted from system
- Token from different SSC instance

Recovery:
- Verify token value (must be valid UUID)
- List scans to find correct token
- Check SSC instance connection
```

### Download Before Completion
```
Error: Cannot download - scan not complete
Possible causes:
- Scan still running
- Scan in queued state

Recovery:
- Wait for scan to complete
- Use scan_wait_for with timeout
- Check scan status first
```

### Timeout Exceeded
```
Error: Timeout waiting for scan
Possible causes:
- Scan taking longer than expected
- Scan stuck in queue
- Sensor issues

Recovery:
- Increase timeout value
- Check scan status manually
- Verify sensor availability
- Contact administrator if scan stuck
```

## Complete Examples

### Example 1: Monitor Recent Activity
**See what's currently happening:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {"status": "RUNNING|QUEUED"},
    "--embed": "scSastScan"
  }
}
```

**Check specific scan progress:**
```json
{
  "tool": "fcli_sc_sast_scan_status",
  "parameters": {
    "scanJobToken": "abc-123-def-456"
  }
}
```

Output shows progress: "RUNNING - 45% complete"

### Example 2: Historical Analysis
**List completed scans for an application:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {
      "publishToApplicationVersion": "MyApp:1.0",
      "status": "COMPLETED"
    },
    "pagination-offset": 0
  }
}
```

**Download multiple FPR files for comparison:**
```json
{
  "tool": "fcli_sc_sast_scan_download",
  "parameters": {
    "scanJobToken": "scan-1",
    "--fpr": "scan-1.fpr"
  }
}
```
```json
{
  "tool": "fcli_sc_sast_scan_download",
  "parameters": {
    "scanJobToken": "scan-2",
    "--fpr": "scan-2.fpr"
  }
}
```

### Example 3: Troubleshooting Failed Scans
**Find failed scans:**
```json
{
  "tool": "fcli_sc_sast_scan_list",
  "parameters": {
    "query": {"status": "FAILED"},
    "--embed": "scSastScan"
  }
}
```

Examine failure details in embedded data - look for error messages, sensor issues, configuration problems

**Download log for detailed analysis:**
```json
{
  "tool": "fcli_sc_sast_scan_download",
  "parameters": {
    "scanJobToken": "failed-token",
    "--log": "failure.log"
  }
}
```

Review log contents for root cause

## Tips and Best Practices

1. **Use filters**: Always filter scan lists when looking for specific scans - reduces data transfer and improves performance

2. **Embed selectively**: Only use `--embed: "scSastScan"` when you need detailed information - it increases response size

3. **Pagination strategy**: For large scan histories, use `pagination-offset` to control starting position (e.g., `pagination-offset: 50` for second page)

4. **Status checking frequency**: Poll scan status every 60-120 seconds to balance responsiveness and server load

5. **Timeout conservatively**: Set timeouts higher than expected scan duration - better to wait longer than to timeout prematurely

6. **Job token storage**: Save scanJobTokens from `scan_start` operations for later monitoring and reference

7. **Download timing**: Always verify scan is COMPLETED before attempting downloads

8. **Log analysis**: When scans fail, always download and review logs before opening support tickets

9. **Query syntax**: Test complex queries with simple filters first, then combine when working correctly

10. **Historical tracking**: Regularly export scan lists for audit trails and trend analysis
