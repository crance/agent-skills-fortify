# Listing and Filtering Vulnerabilities
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to list security vulnerabilities/issues for a release, optionally filtering by severity, category, or other criteria.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to query. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
```
**Expected**: Release details including ID

---

### Step 2 - List All Issues (Unfiltered)
Get a basic list of all vulnerabilities for the release:

```tool
fcli_fod_issue_list --release "MyApp:MyRelease"
```

**Expected**: List of issues with basic information

Example response:
```json
{
  "pagination": {
    "hasMore": false,
    "totalRecords": 42
  },
  "records": [
    {
      "issueId": 12345,
      "primaryTag": "Critical",
      "category": "SQL Injection",
      "severityString": "Critical",
      "foundDate": "2026-02-14T10:00:00Z",
      "primaryLocationFull": "src/main/java/UserService.java:45"
    }
  ]
}
```

---

### Step 3 - Filter by Severity (Server-Side)
Use the `--filters-param` parameter for efficient server-side filtering:

**Filter by Critical severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
```

**Filter by High severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:High"
```

**Filter by Medium severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Medium"
```

**Expected**: Only issues matching the severity level

---

### Step 4 - Filter by Category
Filter issues by vulnerability category:

```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "category:SQL Injection"
```

**Common Categories**:
- SQL Injection
- Cross-Site Scripting (XSS)
- Authentication Issues
- Authorization Issues
- Path Traversal
- Command Injection
- Insecure Cryptography
- Sensitive Data Exposure
- Security Misconfiguration

---

### Step 5 - Include Additional Details
Use `--embed` to include detailed information:

**Include vulnerability details and recommendations:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  --embed "details"
```

**Include recommendations and history:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --embed "details,recommendations,history"
```

**Embed Options**:
- `details` - Full vulnerability details and recommendations
- `recommendations` - Remediation guidance
- `history` - Status changes and audit trail

**Expected**: Enriched issue data with embedded fields

Example with details:
```json
{
  "records": [
    {
      "issueId": 12345,
      "category": "SQL Injection",
      "severityString": "Critical",
      "primaryLocationFull": "src/main/java/UserService.java:45",
      "details": {
        "summary": "SQL Injection vulnerability in user authentication",
        "explanation": "The application constructs SQL queries using unsanitized user input...",
        "recommendation": "Use parameterized queries or prepared statements to prevent SQL injection..."
      }
    }
  ]
}
```

---

### Step 6 - Handle Pagination
If there are many issues, handle pagination:

**First page:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
```

**Check pagination in response:**
```json
{
  "pagination": {
    "hasMore": true,
    "offset": 50,
    "totalRecords": 150
  }
}
```

**Get next page:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  pagination-offset 50
```

**Continue until `pagination.hasMore` = false**

---

### Step 7 - Client-Side Post-Filtering (Optional)
Use `query` parameter for additional client-side filtering:

**Filter by file path pattern:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query { "primaryLocationFull": ".*Controller\\.java.*" }
```

**Filter by found date:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query { "foundDate": "2026-02-.*" }
```

**Note**: Client-side filtering happens after data is retrieved, so prefer server-side `--filters-param` when possible.

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. List Critical issues with full details
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  --embed "details,recommendations"

# 4. List High issues with history
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:High"
  --embed "history"

# 5. Filter by specific category
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "category:SQL Injection"
  --embed "details"
```

---

## Combining Filters

You can combine multiple filters if supported:

```tool
fcli_fod_issue_list \n  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical,category:SQL Injection"
  --embed "details"
```

**Note**: Check FoD documentation for exact filter combination syntax.

---

## Filter by Analysis Status

Filter issues by their review/analysis status:

**Show only unreviewed issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "audited:false"
```

**Show only audited issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "audited:true"
```

**Show suppressed issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "suppressed:true"
```

---

## Common Query Patterns

### Pattern 1: Critical and High Issues Only
```tool
# Get Critical issues
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  --embed "details"

# Get High issues
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:High"
  --embed "details"
```

### Pattern 2: New Issues Since Last Review
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "audited:false"
  --embed "details"
```

### Pattern 3: Specific Vulnerability Type
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "category:Cross-Site Scripting"
  --embed "details,recommendations"
```

---

## Understanding Issue Fields

Key fields in issue responses:

| Field | Description | Example |
|-------|-------------|---------|
| `issueId` | Unique issue identifier | `12345` |
| `category` | Vulnerability type | `"SQL Injection"` |
| `severityString` | Severity level | `"Critical"` |
| `primaryLocationFull` | File and line number | `"UserService.java:45"` |
| `foundDate` | When issue was detected | `"2026-02-14T10:00:00Z"` |
| `audited` | Has been reviewed | `true` / `false` |
| `suppressed` | Has been suppressed | `true` / `false` |
| `primaryTag` | Current analysis status | `"Exploitable"`, `"Suspicious"` |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Release not found" | Verify release name with `fcli_fod_release_list` (see [Finding Releases](find-release.md)) |
| "No issues found" | Verify scans have completed; check scan status |
| "Invalid filter" | Check filter syntax: `"severityString:Critical"` (colon-separated) |
| "Pagination issues" | Track `pagination.offset` and `pagination.hasMore` properly |
| "Missing details" | Add `--embed "details"` to include full information |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Related Workflows

- **Count and summarize issues**: See [Vulnerability Summary](vulnerability-summary.md)
- **Get remediation guidance**: See [Remediation Workflow](remediation-workflow.md)
- **Find correct release**: See [Finding Releases](find-release.md)

---

## Best Practices

✅ **DO:**
- Use `--filters-param` for server-side filtering (faster, less data transfer)
- Include `--embed "details"` when you need recommendations
- Handle pagination for large result sets
- Filter by severity to prioritize critical issues

❌ **DO NOT:**
- Use client-side `query` when server-side `--filters-param` can achieve the same result
- Retrieve all issues without filtering when only specific severities are needed
- Forget to check `pagination.hasMore` for additional results
- Assume issue IDs are sequential or predictable
