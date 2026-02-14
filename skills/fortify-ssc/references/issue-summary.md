# Summarise and count issues

## Use Case
Generate a summary report showing issue distribution by severity (Critical, High, Medium, Low) and by vulnerability category (SQL Injection, XSS, etc.) for management review.

## Workflow Steps
### Step 1 - Verify Authentication
```tool
fcli_ssc_session_list
```
**Expected**: `Expired` column shows `No`

---

### Step 2 - Verify if Application Version exists
```tool
fcli_ssc_appversion_get appVersionNameOrId "MyApp:MyRelease"
```
**Expected**: Application Version details with its ID

---

### Step 3 - Discover available grouping
```tool
fcli_ssc_issue_list_groups --appversion "MyApp:MyRelease"
```
**Look for**: Group types like:
- `Folder` - Severity folders (Critical/High/Medium/Low)
- `Category` - Vulnerability categories
- `Kingdom` - High-level vulnerability kingdoms
- `OWASP Top 10 2021` - OWASP mappings

---

### Step 4: Get Issue Count by Severity
```tool
fcli_ssc_issue_count --appversion "MyApp:MyRelease" --by "Folder"
```

**Expected Output**: Breakdown like:
```
Folder          Count
Critical        5
High            12
Medium          28
Low             45
```

---

### Step 5: Get Issue Count by Category
```tool
fcli_ssc_issue_count --appversion "MyApp:MyRelease" --by "Category"
```

**Expected Output**: Breakdown like:
```
Category                    Count
SQL Injection              3
Cross-Site Scripting       8
Path Manipulation          2
Weak Cryptography          4
...
```

---

### Step 6: Get Issue Count by OWASP Top 10
```tool
fcli_ssc_issue_count --appversion "MyApp:MyRelease" --by "OWASP Top 10 2021"
```

**Expected Output**: OWASP category mapping like:
```
OWASP Category                                Count
A01 Broken Access Control                    7
A02 Cryptographic Failures                   5
A03 Injection                               12
...
```

---

### Step 7: List Top Critical Issues
```tool
# First discover filters
fcli_ssc_issue_list_filters --appversion "MyApp:MyRelease"

# Then list critical issues with embedded details
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical" --embed "details,auditHistory"
```

**Result**: Detailed list of all critical issues with:
- File locations
- Detailed vulnerability information
- Audit history of changes

**Alternative - Get only top 10**:
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical" --embed "details"
```

---

## Report Template

**MyApp MyRelease Security Assessment Summary**

| Metric | Value |
|--------|-------|
| Total Issues | [from step 2] |
| Critical | [from step 4] |
| High | [from step 4] |
| Medium | [from step 4] |
| Low | [from step 4] |

**Top Vulnerability Categories:**
[List from step 5]

**OWASP Top 10 Distribution:**
[List from step 6]

**Critical Issues Requiring Immediate Attention:**
[Details from step 7]

---