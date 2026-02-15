# Finding Applications and Releases
**Prerequisites:** Authentication verified (see SKILL.md)

## Use Case
You need to find the correct application and release when the exact name is uncertain or when exploring available options.

## Workflow Steps

### Step 1 - Try Direct Release Lookup
If you know the exact release identifier, try getting it directly:

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"
```

**Expected**: Release details with ID

Example response:
```json
{
  "releaseId": 10022,
  "releaseName": "MyRelease",
  "applicationName": "MyApp",
  "applicationId": 5011,
  "sdlcStatusType": "Production",
  "rating": "Critical"
}
```

**If this succeeds**, you have the correct release - proceed with your task.

**If this fails**, continue to Step 2 to discover the correct name.

---

### Step 2 - List All Applications
If the release name is unknown or incorrect, start by listing applications:

```tool
fcli_fod_app_list
```

**Expected**: List of all accessible applications

Example response:
```json
{
  "records": [
    {
      "applicationId": 5011,
      "applicationName": "MyApp",
      "businessCriticalityType": "High"
    },
    {
      "applicationId": 5012,
      "applicationName": "MyOtherApp",
      "businessCriticalityType": "Medium"
    }
  ]
}
```

---

### Step 3 - Search Applications by Partial Name
If there are many applications, use client-side filtering to narrow down:

**Search by partial application name (case-insensitive regex):**
```tool
fcli_fod_app_list query { "applicationName": ".*MyApp.*" }
```

**Expected**: Applications matching the pattern

---

### Step 4 - List Releases for Specific Application
Once you identify the application, list all its releases:

```tool
fcli_fod_release_list query { "applicationName": "MyApp" }
```

**Or list all releases and filter:**
```tool
fcli_fod_release_list
```

**Expected**: List of releases with their details

Example response:
```json
{
  "records": [
    {
      "releaseId": 10022,
      "releaseName": "MyRelease",
      "applicationName": "MyApp",
      "sdlcStatusType": "Production"
    },
    {
      "releaseId": 10023,
      "releaseName": "MyRelease-v2",
      "applicationName": "MyApp",
      "sdlcStatusType": "Development"
    }
  ]
}
```

---

### Step 5 - Search Releases by Partial Name
Search for releases matching a pattern:

**Search by partial release name:**
```tool
fcli_fod_release_list query { "releaseName": ".*MyRelease.*" }
```

**Search by application and release:**
```tool
fcli_fod_release_list query { "applicationName": ".*MyApp.*", "releaseName": ".*v2.*" }
```

**Expected**: Releases matching the criteria

---

### Step 6 - Present Results to User
When a query returns multiple matches:

1. **Present results to the user** as a numbered list with key details:
   - `1. MyApp:MyRelease (Production, releaseId: 10022)`
   - `2. MyApp:MyRelease-v2 (Development, releaseId: 10023)`
   - `3. MyOtherApp:MyRelease (QA, releaseId: 10024)`

2. **Ask the user to pick** the correct release

3. **Never assume** which one the user intended — always confirm

---

### Step 7 - Retrieve Full Release Details
Once the user confirms the correct release, retrieve full details:

```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease-v2"
```

**Or use the release ID directly:**
```tool
fcli_fod_release_get --qualifiedReleaseNameOrId "10023"
```

**Expected**: Complete release information including:
- Release ID and name
- Application details
- SDLC status (Development, QA, Production)
- Assessment types available
- Current scan status

---

## Complete Example Flow

```tool
# 1. Verify authentication
fcli_fod_session_list refresh-cache=true

# 2. Try direct lookup first
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease"

# 3. If not found, list all applications
fcli_fod_app_list

# 4. Search for specific application
fcli_fod_app_list query { "applicationName": ".*MyApp.*" }

# 5. List releases for that application
fcli_fod_release_list query { "applicationName": "MyApp" }

# 6. Get specific release details
fcli_fod_release_get --qualifiedReleaseNameOrId "MyApp:MyRelease-v2"
```

---

## Understanding Release Identifiers

FoD releases can be referenced in two ways:

### Option 1: Colon-Separated Format
```
"<ApplicationName>:<ReleaseName>"
```

Examples:
- `"MyApp:MyRelease"`
- `"WebPortal:v2.0"`
- `"API-Gateway:Production"`

**Note**: Case-sensitive and must match exactly

### Option 2: Release ID
```
"<ReleaseId>"
```

Examples:
- `"10022"`
- `"10023"`

**Note**: Numeric ID, stays consistent even if release name changes

---

## Common Search Patterns

### Pattern 1: Find Production Releases
```tool
fcli_fod_release_list query { "sdlcStatusType": "Production" }
```

### Pattern 2: Find Releases for Multiple Apps
```tool
# List all releases
fcli_fod_release_list

# Then filter client-side by multiple app names
fcli_fod_release_list query { "applicationName": ".*(App1|App2|App3).*" }
```

### Pattern 3: Find Development/Test Releases
```tool
fcli_fod_release_list query { "sdlcStatusType": "Development" }
```

### Pattern 4: Find Recently Created Releases
```tool
# List all releases (sorted by creation date if available)
fcli_fod_release_list
```

---

## Getting Application Details

If you need more information about an application:

```tool
fcli_fod_app_get app-name-or-id "MyApp"
```

**Or by ID:**
```tool
fcli_fod_app_get app-name-or-id "5011"
```

**Expected**: Detailed application information including:
- Business criticality
- Application type
- Owner information
- Attribute details

---

## List Assessment Types for Release

Check what scan types are available for a release:

```tool
fcli_fod_release_list_assessment_types --release "MyApp:MyRelease"
```

**Expected**: Available assessment types (SAST, DAST, Mobile, OSS, etc.)

Example response:
```json
{
  "records": [
    {
      "assessmentTypeId": 1,
      "name": "Static Assessment",
      "scanType": "Static"
    },
    {
      "assessmentTypeId": 2,
      "name": "Dynamic Assessment",
      "scanType": "Dynamic"
    },
    {
      "assessmentTypeId": 3,
      "name": "Open Source Analysis",
      "scanType": "OpenSource"
    }
  ]
}
```

---

## List Scans for Application or Release

### List all scans for an application:
```tool
fcli_fod_app_list_scans app-name-or-id "MyApp"
```

### List all scans for a specific release:
```tool
fcli_fod_release_list_scans --release "MyApp:MyRelease"
```

**Expected**: Historical scan information including:
- Scan types (SAST, DAST, OSS)
- Scan dates
- Scan status
- Vulnerability counts

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Application not found" | List all apps with `fcli_fod_app_list` to see available names |
| "Release not found" | Check exact spelling and case; use `fcli_fod_release_list` to discover |
| "Multiple matches" | Present options to user; never guess which one they meant |
| "No releases for app" | Verify application name; check if releases exist with `fcli_fod_release_list` |
| "Permission denied" | User may not have access to specific applications/releases |
| "Session expired" | Re-authenticate: `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Best Practices

✅ **DO:**
- Start with direct lookup if you have a specific name
- Use partial matching (regex) when exploring
- Present multiple matches as numbered list for user selection
- Include key details (SDLC status, ID) when presenting options
- Use release ID for consistency once identified

❌ **DO NOT:**
- Guess which release the user meant when multiple matches exist
- Assume release names match application names
- Use case-insensitive matching for final lookups (release names are case-sensitive)
- Forget to verify the release exists before starting operations

---

## Related Workflows

- **After finding release**: Run scans (see [SAST](run-sast-scan.md), [SCA](run-sca-scan.md), [DAST](run-dast-scan.md))
- **View vulnerabilities**: See [List and Filter Vulnerabilities](list-filter-vulnerabilities.md)
- **Summary of findings**: See [Vulnerability Summary](vulnerability-summary.md)
