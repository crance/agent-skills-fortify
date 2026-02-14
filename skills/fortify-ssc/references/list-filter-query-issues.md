# List issues and apply filtering

## Use Case
You need to review all critical vulnerabilities in the "MyApp:MyRelease" application to prioritize remediation efforts.

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

### Step 3 - Discover available filters
```tool
fcli_ssc_issue_list_filters --appversion "MyApp:MyRelease"
```
**Look for**: `Folder:Critical` in the output

---

### Step 4: List Critical Issues
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical" --embed "details"
```

**Result**: All critical severity issues with extended details like:
- Issue ID
- Fortify Priority (friority)
- Folder Name
- Location (file:line)
- Issue name/description
- Detailed vulnerability information (via --embed)

**With Audit History**:
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical" --embed "auditHistory,comments"
```

**Tip**: Use `--embed` to get additional information in a single API call instead of fetching details separately.

---

### Step 5: Get Critical Issue Count
```tool
fcli_ssc_issue_count --appversion "MyApp:MyRelease" --by "Folder"
```

**Result**: Count breakdown showing how many Critical, High, Medium, Low issues exist

---

## Alternative: Using Client-Side Query (Avoid)
List Critical Issues (client-side query)
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" query { "friority": ".*Critical.*" }
```

List Multiply Severities (client-side query)
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" query { "friority": "^(Critical|High)$" }
```

**Note**: Server-side filters (`--filter`) are more efficient for large datasets. Use query parameter for complex custom filtering.

## Additional Information

### Listing Issues
```tool
# Basic list
fcli_ssc_issue_list --appversion "MyApp:MyRelease"

# With Details (recommended)
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --embed "details"

# Include suppressed issues
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --include "suppressed"
```

| `--embed` options | `--include` options |
|-------------------|---------------------|
| `details` - Vulnerability details | `visible` - (default) |
| `comments` - User comments | `hidden` - Hidden issues |
| `auditHistory` - Audit trail | `removed` - Removed issues |
| Combine: `"details,auditHistory"` | `suppressed` - Suppressed |

### Apply filters
```tool
# Filtered by severity (discover filters first!)
fcli_ssc_issue_list_filters --appversion "MyApp:MyRelease"  # Step 1: Discover
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --filter "Folder:Critical"  # Step 2: Apply filter
```
