# Listing and Filtering Vulnerabilities
**Prerequisites:** Authentication verified (see SKILL.md)

## Contents
- [Use Case](#use-case)
- [Workflow Steps](#workflow-steps)
- [Complete Example Flow](#complete-example-flow)
- [Combining Filters](#combining-filters)
- [Filter by Analysis Status](#filter-by-analysis-status)
- [Common Query Patterns](#common-query-patterns)
- [Understanding Issue Fields](#understanding-issue-fields)
- [Troubleshooting](#troubleshooting)
- [Related Workflows](#related-workflows)
- [Best Practices](#best-practices)

## Use Case
You need to list security vulnerabilities/issues for a release, optionally filtering by severity, category, or other criteria.

## Workflow Steps

### Step 1 - Select Release
First, identify the release to query. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"
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
      "introducedDate": "2026-02-14",
      "primaryLocationFull": "src/main/java/UserService.java:45"
    }
  ]
}
```

---

### Step 3 - Filter by Severity
Use the `query` parameter for client-side filtering by severity:

**Filter by Critical severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "Critical"}
```

**Filter by High severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "High"}
```

**Filter by Medium severity:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "Medium"}
```

**Expected**: Only issues matching the severity level

---

### Step 4 - Filter by Category
Filter issues by vulnerability category:

```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"category": "SQL Injection"}
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
  query {"severity": "Critical"}
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
  query {"severity": "Critical"}
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
  query {"severity": "Critical"}
  pagination-offset 50
```

**Continue until `pagination.hasMore` = false**

---

### Step 7 - Client-Side Post-Filtering (Optional)
Use `query` parameter for additional client-side filtering:

**Filter by file location pattern:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query { "location": ".*Controller\.java.*" }
```

**Filter by introduced date** (client-side only — `introducedDate` is NOT a valid `query` field):
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
```
Then filter the response records by `introducedDate` ≥ desired date.

**Note**: Valid `query` fields are: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`. `--filters-param` does NOT exist and will fail. `primaryLocationFull`, `foundDate`, and `severityString` are response fields only — they cannot be used in `query`.

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. List Critical issues with full details
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "Critical"}
  --embed "details,recommendations"

# 4. List High issues with history
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "High"}
  --embed "history"

# 5. Filter by specific category
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"category": "SQL Injection"}
  --embed "details"
```

---

## Combining Filters

Combine `query` fields to filter on multiple dimensions simultaneously:

```tool
fcli_fod_issue_list
  --release "MyApp:MyRelease"
  query {"severity": "Critical", "category": "SQL Injection"}
  --embed "details"
```

**Note**: Only `query` fields are combinable: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`.

---

## Filter by Analysis Status

Filter issues by their review/analysis status:

**Show only unreviewed issues** (retrieve all, filter client-side by `audited` field — no query field exists):
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
```
Then filter the response records where `"audited": false`.

**Show suppressed issues** (use `--include`):
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --include "suppressed"
```

**Show fixed issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --include "fixed"
```

**Note**: `audited` is NOT a valid `query` field. To filter by audit status, retrieve all issues and check the `audited` boolean field in the response. `--include` controls which _statuses_ (visible/fixed/suppressed) are returned, not the `audited` field.

---

## Common Query Patterns

### Pattern 1: Critical and High Issues Only
```tool
# Get Critical issues
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "Critical"}
  --embed "details"

# Get High issues
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"severity": "High"}
  --embed "details"
```

### Pattern 2: New Issues Since Last Review
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
```
Then filter response records where `"audited": false` client-side.

### Pattern 3: Specific Vulnerability Type
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  query {"category": "Cross-Site Scripting"}
  --embed "details,recommendations"
```

---

## Understanding Issue Fields

Key fields in issue responses:

| Field | Description | Example |
|-------|-------------|---------|
| `issueId` | Unique issue identifier | `12345` |
| `category` | Vulnerability type | `"SQL Injection"` |
| `severityString` | Severity level (response only — use `severity` in `query`) | `"Critical"` |
| `primaryLocationFull` | Full file path and line (response only — use `location` in `query`) | `"UserService.java:45"` |
| `introducedDate` | When issue was first detected | `"2026-02-14"` |
| `audited` | Has been reviewed | `true` / `false` |
| `suppressed` | Has been suppressed | `true` / `false` |
| `primaryTag` | Current analysis status | `"Exploitable"`, `"Suspicious"` |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Release not found" | Verify release name with `fcli_fod_release_list` (see [Finding Releases](find-release.md)) |
| "No issues found" | Verify scans have completed; check scan status |
| "Invalid filter" | `query` only supports these fields: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker` |
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
- Use `query {"severity": "Critical"}` to filter by severity (valid fields: `category`, `foundInReleases`, `instanceId`, `location`, `severity`, `visibilityMarker`)
- Include `--embed "details"` when you need recommendations
- Handle pagination for large result sets
- Filter by severity to prioritize critical issues

❌ **DO NOT:**
- Use `--filters-param` — it does NOT exist on `issue_list` and will fail
- Retrieve all issues without filtering when only specific severities are needed
- Forget to check `pagination.hasMore` for additional results
- Assume issue IDs are sequential or predictable
