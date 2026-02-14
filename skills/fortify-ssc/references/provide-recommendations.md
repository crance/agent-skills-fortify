# Provide vulnerability recommendations

## Use Case
You are asked to provide recommendations from the scan findings.

## Workflow Steps
### Step 1 - Verify Authentication
```tool
fcli_ssc_session_list
```
**Expected**: `Expired` column shows `No`

---

### Step 2 - Retrieve the issue details
If no issue id is provided, retrieve details, including audit history of Critical and High Issues and combine into a list.
```tool
fcli_ssc_issue_list --appversion "MyApp:MyRelease" --embed "details" --filter "Folder:Critical"

fcli_ssc_issue_list --appversion "MyApp:MyRelease" --embed "details" --filter "Folder:High"
```

**Expected**: Recommendations included, with optional Aviator's code fix suggestion.

Example with Aviator's code fix suggestion:
```json
{
  "exitCode" : 0,
  "stderr" : "",
  "records" : [ {
    "projectVersionId" : 10022,
    "recommendation" : "It is good practice to run your containers as a non-root user when possible...",
    "audited" : true,
    "hidden" : false,
    "removed" : false,
    "suppressed" : false,
    "primaryTag" : {
      "tagId" : 10,
      "tagGuid" : "87f2364f-dcd4-49e6-861d-f8d3f351686b",
      "tagName" : "Analysis",
      "tagValue" : "Suspicious"
    },
    "comments" : [ {
      "issueId" : 38994,
      "issueName" : null,
      "seqNumber" : 0,
      "auditTime" : "2025-10-10T08:27:49.643+00:00",
      "comment" : "Here's a suggested modification:\n\nIn the Dockerfile, add the following lines after the RUN apt-get update instruction:...",
      "userName" : "Fortify Aviator",
      "projectVersionId" : null,
      "projectVersionName" : null,
      "projectName" : null,
      "issueInstanceId" : null,
      "issueEngineType" : null
    } ],
    "auditHistory" : [ {
      "issueId" : 38994,
      "sequenceNumber" : 0,
      "userName" : "Fortify Aviator",
      "auditDateTime" : "2025-10-10T08:27:49.643+00:00",
      "attributeName" : "Analysis",
      "oldValue" : null,
      "newValue" : "Suspicious",
      "conflict" : false,
      "valueType" : "LIST"
    } ]
  }]
}

```

---

### Step 3 - Provide Recommendations
Open issues are issues that have not been hidden (`hidden: false`), removed (`removed: false`),  suppressed (`suppressed: false`), or `Analysis` tag is not `Not an Issue` 

If there is a code fix suggestion provided by Aviator, we should use it. Otherwise, use the information from `recommendation` attribue.

Show the recommendations for Open Issues to the user.

Ask the user if he needs your help to apply Aviator's suggested code fix, if applicable

---