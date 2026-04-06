---
description: "Fix a bug from the skills and references."
name: "fix bug from bug report"
argument-hint: "Describe the bug to fix or provide the bug ID from the bug report."
tools: [
  vscode, 
  todo, 
  edit, 
  search,
  read,
  agent,
  web,
  'fcli-ssc/*', 
  'fcli-sast/*', 
  'fcli-dast/*', 
  'fcli-fod/*']
---

## MCP Tool Call
Before making any changes, perform a MCP tool call to verify the bug and gather necessary information for the fix. Use the appropriate tool based on the bug description or ID provided.

# Fix Bug
Fix the bug described in the bug report.
If no bug id was provided, or it is unclear which bug to fix, ask the user before proceeding.

STRICTLY test with both positive and negative test cases with the MCP tool to ensure the bug is fully resolved and does not cause any regressions.

Make the necessary changes if the actual tool call tallys with the documented bug.
Otherwise, ask the user for clarification on the bug details before proceeding with the fix.

After fixing the bug, summarize what was done and marked the bug as "fixed" in the bug report. 