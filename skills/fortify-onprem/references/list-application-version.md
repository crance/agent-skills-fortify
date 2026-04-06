# Listing Applications and Versions
**Prerequisites:** Authentication verified (see SKILL.md)

**List Applications**
```tool
fcli_ssc_app_list
```

**List Versions for a specific Application**
```tool
fcli_ssc_appversion_list
```

**Note**: To find application versions, list applications first, then use the app name with known version names.

## Use Case
You need to get the application version id for subsequent processing.

## Workflow Steps
### Step 1 - Verify if Application Version exists
```tool
fcli_ssc_appversion_get appVersionNameOrId "MyApp:MyRelease"
```
**Expected**: Application Version details with its ID

---

### Step 2 - If Application Version Not Found
If Step 2 fails, use a `client-side query` to help the user find the correct name.

**Search by application name (partial match):**
```tool
fcli_ssc_app_list query { "name": ".*MyApp.*" }
```

**Search by application version (partial match):**
```tool
fcli_ssc_appversion_list query { "application.name": ".*MyApp.*" }
```

---

### Step 3 - Handle Multiple Results
When a query returns multiple matches:

1. **Present results to the user** as a numbered list:
   - `1. MyApp:1.0`
   - `2. MyApp:2.0`
   - `3. MyApp-Legacy:1.0`
2. **Ask the user to pick** the correct application version
3. **Never assume** which one the user intended — always confirm

---

### Step 4 - Retrieve Version Details
Once the user confirms the correct version, retrieve full details:
```tool
fcli_ssc_appversion_get appVersionNameOrId "MyApp:2.0"
```

Store the `id` from the response for use in subsequent operations (e.g., `issue_list`, `issue_count`).

---

## Common Patterns

### List all versions for a known application
```tool
fcli_ssc_appversion_list query { "application.name": "^MyApp$" }
```

### Find applications when user gives a vague name
```tool
fcli_ssc_app_list query { "name": ".*payment.*" }
```
**Tip**: Use broad regex (`.*keyword.*`) when the user is unsure of the exact name.

### Get version with embedded attributes
```tool
fcli_ssc_appversion_get appVersionNameOrId "MyApp:2.0" --embed "attrs"
```