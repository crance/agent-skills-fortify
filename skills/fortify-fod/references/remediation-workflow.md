# Remediation Workflow and Recommendations
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to provide remediation recommendations and guidance for fixing security vulnerabilities. This includes prioritizing AI-generated code fixes from Fortify Aviator when available.

## Critical: Prioritize Aviator Code Fixes
**Fortify Aviator** provides AI-generated code fix suggestions that appear in issue comments. When providing remediation guidance:

1. **ALWAYS check for Aviator suggestions first**
2. **Present Aviator code fixes before general recommendations**
3. **Aviator suggestions include specific code changes** - not just advice
4. **Look for `userName: "Fortify Aviator"` in comments array**

---

## Workflow Steps

### Step 1 - Select Release
First, identify the release to analyze. If uncertain, see [Finding Releases](find-release.md).

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
```
**Expected**: Release details including ID

---

### Step 2 - Retrieve Issues with Full Details
Get issues with all relevant information including comments and recommendations:

**For Critical issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  --embed "details,recommendations,history"
```

**For High issues:**
```tool
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:High"
  --embed "details,recommendations,history"
```

**Expected**: Issues with:
- `details` - Full vulnerability explanation and general recommendations
- `recommendations` - Additional remediation guidance
- `history` - Tracking of status changes and reviews
- `comments` - User comments and **Aviator code fix suggestions** (included by default in response)

**Note:** The `comments` field is automatically included in issue responses - it is NOT an embed parameter. Valid embed parameters are: `allData`, `summary`, `details`, `recommendations`, `history`, `requestResponse`, `headers`, `parameters`, `traces`.

---

### Step 3 - Identify Aviator Suggestions
**Check the `comments` array for Fortify Aviator entries:**

Example response with Aviator suggestion:
```json
{
  "records": [
    {
      "issueId": 38994,
      "category": "Insecure Dockerfile Configuration",
      "severityString": "High",
      "primaryLocationFull": "Dockerfile:5",
      "recommendation": "It is good practice to run your containers as a non-root user when possible...",
      "comments": [
        {
          "issueId": 38994,
          "seqNumber": 0,
          "auditTime": "2025-10-10T08:27:49.643+00:00",
          "comment": "Here's a suggested modification:\n\nIn the Dockerfile, add the following lines after the RUN apt-get update instruction:\n\nRUN groupadd -r appuser && useradd -r -g appuser appuser\nUSER appuser\n\nThis creates a non-privileged user and switches to it.",
          "userName": "Fortify Aviator"
        }
      ]
    }
  ]
}
```

---

### Step 4 - Present Remediation Guidance (Aviator First!)

#### When Aviator Suggestion Exists:
**Present in this order:**

1. **Aviator Code Fix** (PRIORITY)
   ```
   🤖 Fortify Aviator Recommendation:
   
   In the Dockerfile, add the following lines after the RUN apt-get update instruction:
   
   RUN groupadd -r appuser && useradd -r -g appuser appuser
   USER appuser
   
   This creates a non-privileged user and switches to it.
   ```

2. **General Recommendation** (SECONDARY)
   ```
   General Guidance:
   It is good practice to run your containers as a non-root user when possible...
   ```

#### When No Aviator Suggestion:
**Present general recommendation only:**
```
Recommendation:
It is good practice to run your containers as a non-root user when possible...
```

---

### Step 5 - Handle Multiple Issues
When multiple issues are returned, prioritize presentation:

1. **Issues WITH Aviator suggestions** (show first)
2. **Issues WITHOUT Aviator suggestions** (show after)

**Example Summary Format:**
```
Critical Vulnerabilities Requiring Attention
============================================

🤖 ISSUES WITH AI-GENERATED CODE FIXES:

1. Insecure Dockerfile Configuration (High) - Dockerfile:5
   Aviator Fix: Add non-root user configuration...
   [Show full Aviator comment]

2. SQL Injection (Critical) - UserService.java:45
   Aviator Fix: Replace string concatenation with PreparedStatement...
   [Show full Aviator comment]

---

OTHER ISSUES:

3. Path Traversal (Medium) - FileHandler.java:102
   Recommendation: Validate and sanitize file paths...
   
4. Weak Cryptography (Low) - EncryptionUtil.java:33
   Recommendation: Use AES-256 instead of DES...
```

---

### Step 6 - Update Issue Status (Optional)
After reviewing or fixing issues, update their analysis status:

```tool
fcli_fod_issue_update 
  --release "MyApp:MyRelease"
  issue-filters "issueId:38994"
  analysis-status "In Remediation"
  comment "Applied Aviator's suggested fix"
```

**Common Analysis Status Values**:
- `Not an Issue` - False positive
- `In Remediation` - Currently being fixed
- `Reviewed` - Has been examined
- `Fixed` - Issue resolved

---

### Step 7 - Track Audit History
View the history of changes for an issue by including the `history` embed parameter:

The `history` data shows who changed what and when:

```json
{
  "auditHistory": [
    {
      "issueId": 38994,
      "sequenceNumber": 0,
      "userName": "Fortify Aviator",
      "auditDateTime": "2025-10-10T08:27:49.643+00:00",
      "attributeName": "Analysis",
      "oldValue": null,
      "newValue": "Suspicious",
      "valueType": "LIST"
    },
    {
      "issueId": 38994,
      "sequenceNumber": 1,
      "userName": "developer@example.com",
      "auditDateTime": "2025-10-11T14:20:00.000+00:00",
      "attributeName": "Analysis",
      "oldValue": "Suspicious",
      "newValue": "In Remediation",
      "valueType": "LIST"
    }
  ]
}
```

**Note:** The API response field is named `auditHistory`, but the embed parameter to retrieve this data is `history` (e.g., `--embed "history"`).

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Get release details
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. Retrieve Critical issues with full details
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:Critical"
  --embed "details,recommendations,history"

# 4. Retrieve High issues with full details
fcli_fod_issue_list 
  --release "MyApp:MyRelease"
  --filters-param "severityString:High"
  --embed "details,recommendations,history"

# 5. Process results:
#    - Check each issue for comments[].userName == "Fortify Aviator"
#    - Present Aviator suggestions first
#    - Then present general recommendations
```

---

## Detecting Aviator Suggestions (Code Pattern)

When processing responses, use this logic:

```
For each issue:
  aviatorSuggestion = null
  
  For each comment in issue.comments:
    If comment.userName == "Fortify Aviator":
      aviatorSuggestion = comment.comment
      break
  
  If aviatorSuggestion exists:
    Present Aviator suggestion with 🤖 indicator
  Else:
    Present general recommendation from issue.details.recommendation
```

---

## Aviator Suggestion Characteristics

Fortify Aviator suggestions typically include:

1. **Specific file and line references**
2. **Exact code to add/change/remove**
3. **Before/after code snippets**
4. **Explanation of the fix**

**Example Aviator Comment Pattern:**
```
Here's a suggested modification:

In the [File], [action]:

[Old code or location]

[New code]

[Explanation of fix]
```

---

## Remediation Guidance by Category

### SQL Injection
**Aviator might suggest:**
- Using PreparedStatement instead of string concatenation
- Parameterized queries with specific syntax
- ORM query patterns

**General recommendation:**
- Use prepared statements or parameterized queries
- Validate and sanitize input
- Use ORM frameworks

### Cross-Site Scripting (XSS)
**Aviator might suggest:**
- Specific encoding functions (e.g., `html.escape()`)
- Framework-specific escaping methods
- Output encoding at specific locations

**General recommendation:**
- HTML-encode output
- Use Content Security Policy
- Validate and sanitize input

### Authentication/Authorization Issues
**Aviator might suggest:**
- Adding authentication checks
- Implementing role-based access control
- Specific framework authentication patterns

**General recommendation:**
- Implement proper authentication
- Use framework security features
- Follow principle of least privilege

---

## Updating Issue Status

Common workflows for issue management:

### Mark as False Positive
```tool
fcli_fod_issue_update 
  --release "MyApp:MyRelease"
  issue-filters "issueId:38994"
  analysis-status "Not an Issue"
  comment "This is a false positive because..."
```

### Mark as In Remediation
```tool
fcli_fod_issue_update 
  --release "MyApp:MyRelease"
  issue-filters "issueId:38994"
  analysis-status "In Remediation"
  comment "Following Aviator's suggested fix"
```

### Add Comment Without Status Change
```tool
fcli_fod_issue_update 
  --release "MyApp:MyRelease"
  issue-filters "issueId:38994"
  comment "Requires security review before implementing fix"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "No Aviator suggestions found" | Check `comments` array; Aviator may not have generated suggestions yet |
| "Cannot update issue" | Verify issue ID and user permissions |
| "Release not found" | See [Finding Releases](find-release.md) |
| "Missing details" | Ensure `--embed "details,recommendations,history"` is included |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Best Practices

✅ **DO:**
- **ALWAYS check for Aviator suggestions first**
- Present Aviator code fixes with visual indicator (🤖 or "AI-Generated Fix")
- Include both Aviator suggestions AND general recommendations when available
- Show specific code changes from Aviator comments
- Update issue status after applying fixes
- Track remediation progress via audit history

❌ **DO NOT:**
- Skip checking the comments array for Aviator suggestions
- Present only general recommendations when Aviator suggestions exist
- Bury Aviator suggestions below general guidance
- Ignore the userName field in comments
- Apply fixes without understanding the vulnerability
- Forget to mark issues as "In Remediation" when work begins

---

## Related Workflows

- **List and filter issues**: See [List and Filter Vulnerabilities](list-filter-vulnerabilities.md)
- **Count vulnerabilities**: See [Vulnerability Summary](vulnerability-summary.md)
- **Find correct release**: See [Finding Releases](find-release.md)

---

## Why Prioritize Aviator?

Fortify Aviator provides:
- **Specific code changes** vs general advice
- **Context-aware fixes** tailored to your codebase
- **Actionable remediation** developers can apply immediately
- **Time savings** by reducing research needed

**Always surface these valuable suggestions first!**
