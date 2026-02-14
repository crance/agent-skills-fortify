# Handling Background Jobs with mcp_job

## Use Case
When pagination indicates background loading is in progress (indicated by `pagination.jobToken`), you must wait for the job to complete before accessing all records.

## When to Use
- When `pagination.jobToken` is present in the response
- When `pagination.hasMore` is true but `pagination.totalRecords` is not yet available
- Background loading typically occurs with large datasets

## Workflow Pattern

### Step 1 - Detect Background Job
When calling tools like `fcli_ssc_issue_list`, check the pagination section:
```json
{
  "pagination": {
    "hasMore": true,
    "jobToken": "abc123-def456-ghi789",
    "totalRecords": null
  }
}
```

**Indicators**:
- ✅ `jobToken` is present
- ✅ `totalRecords` is `null` or missing
- ✅ `hasMore` is `true`

---

### Step 2 - Wait for Job Completion
Use the `mcp_job` tool with the wait operation:
```tool
fcli_ssc_mcp_job operation "wait" job_token "abc123-def456-ghi789"
```

**Parameters**:
- `operation`: Must be `"wait"`
- `job_token`: The token from `pagination.jobToken`

**Expected Result**:
```json
{
  "status": "completed",
  "message": "Job completed successfully"
}
```

---

### Step 3 - Retry Original Request
Once the job completes, call the original tool again without changing parameters:
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical"
```

**Expected**: Full results with `pagination.totalRecords` populated and complete data set available.

---

## Complete Example

**Scenario**: Listing issues from a large application version

**Call 1** - Initial request:
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical"
```

**Response 1**:
```json
{
  "data": [...],
  "pagination": {
    "hasMore": true,
    "jobToken": "xyz789-abc123",
    "totalRecords": null,
    "offset": 0,
    "limit": 50
  }
}
```

**Call 2** - Wait for background job:
```tool
fcli_ssc_mcp_job operation "wait" job_token "xyz789-abc123"
```

**Response 2**:
```json
{
  "status": "completed"
}
```

**Call 3** - Retry original request:
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical"
```

**Response 3**:
```json
{
  "data": [...],
  "pagination": {
    "hasMore": false,
    "totalRecords": 1234,
    "offset": 0,
    "limit": 50
  }
}
```

---

## Tool Name Variations
The actual MCP tool name depends on your MCP server configuration:
- Short form used in this skill: `fcli_ssc_mcp_job`
- Full form example: `mcp_fcli-ssc_fcli_ssc_mcp_job`

The short form is preferred for readability. When calling tools, use the pattern that matches your MCP server's naming convention (typically the MCP server name prefix is automatically handled).

---

## Common Mistakes to Avoid
❌ **Don't proceed with pagination before waiting**
```
# WRONG
Call issue_list → get jobToken → immediately call with offset
```

✅ **Do wait for job completion first**
```
# CORRECT
Call issue_list → get jobToken → wait → retry original call → then paginate
```

❌ **Don't modify parameters when retrying**
- Keep the same `--filter`, `--appversion`, and other parameters
- The job has precomputed results for your exact query

❌ **Don't wait indefinitely**
- If wait operation times out, inform the user that data loading is taking longer than expected
- SSC may still be processing the request

---

## References
- Related: [List, Filter and Query Issues](list-filter-query-issues.md) - Main workflow for querying issues
- Related: [Issue Summary](issue-summary.md) - How to count and summarize issues
