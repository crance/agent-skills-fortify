# Project Instructions

This project is used for creating Fortify skill files. Each skill file contains instructions for how the agent should use the MCP tools to interact with a specific Fortify product (SSC, FoD, SC-SAST, SC-DAST). 

The MCP tools are accessed via FCLI, which is the command-line interface for Fortify products. The skill files should provide clear guidance on when and how to use each MCP tool, including parameter formats, authentication steps, and example workflows.


# References
- [Agent Skills Specification](https://agentskills.io/specification)
- [FCLI General Documentation](https://fortify.github.io/fcli/latest/)
- [FCLI MCP Server Documentation](https://fortify.github.io/fcli/latest/manpage/fcli-util-mcp-server-start.html)
- [FCLI GitHub Repository](https://github.com/fortify/fcli)

---

## Skill Architecture

### Design Principles
1. **Self-contained skills** - Each skill must work independently with no dependencies on other skills
2. **One skill per MCP module** - Mirrors FCLI's MCP architecture (`fcli-ssc`, `fcli-fod`, `fcli-sc-sast`, `fcli-sc-dast`)
3. **Accept necessary duplication** - Small duplication is acceptable for reliability
4. **Progressive disclosure** - Put detailed workflows, scripts, etc in `references/` folder, referenced from SKILL.md

### Folder Structure
```
skills/
├── fortify-fod/
│   ├── SKILL.md
│   └── references/
├── fortify-ssc/
│   ├── SKILL.md              # Main skill file (loaded when skill activated)
│   └── references/           # Detailed workflows (loaded on-demand)
│       ├── README.md
│       └── *.md
├── fortify-scsast/
│   └── SKILL.md
└── fortify-scdast/
    └── SKILL.md
```

### Loading Behavior
| Component | When Loaded | Context Impact |
|-----------|-------------|----------------|
| `SKILL.md` | Skill activation | Immediate - keep concise |
| `references/*.md` | On-demand | LLM fetches when relevant |

### Optimizing Examples for Progressive Disclosure

The LLM decides to fetch examples based on **keyword matching** between user requests and reference text. To maximize progressive disclosure effectiveness:

**1. In SKILL.md - Use keyword-rich example references:**
```markdown
## Example Workflows
| Workflow | Use When User Says... |
|----------|----------------------|
| [upload-workflow.md](./references/upload-workflow.md) | "upload FPR", "import scan", "submit artifact", "upload results" |
| [filter-issues.md](./references/filter-issues.md) | "filter by severity", "show critical", "find high priority" |
```

**2. In each example file - Start with clear use case:**
```markdown
# Upload Scan Workflow

## Use Case
Upload FPR or SCA scan results to SSC for analysis.

## Keywords (more keywords = better matching = smarter progressive loading)
upload, FPR, artifact, SCA, import, scan results
```

---

## SKILL.md Template Structure

Each skill file should follow this consistent structure with product-specific content:

```markdown
---
name: fortify-{product}
description: {Product} guide for MCP tools. {Brief description of product and capabilities}.
metadata:
  version: "0.0.1"
---

# Fortify {Product} Skill
Fortify {Product} ({Product Acronym}) integration via Model Context Protocol (MCP).

## Available MCP Tools
Only key MCP tools for {product} are listed here.
| Tool | Description | When to Use |
|-----------|-------------|-------------|
| `fcli_{product}_...` | ... | ... |

## Parameter Formats (Required)
| Parameter | Format | Example |
|-----------|--------|---------|
| ... | ... | ... |

## Authentication
Product-specific session check and login instructions.
- Session check tool
- What to do if expired
- Login command syntax (for user to run locally)

## Filtering
- Filtering preference, server-side filter is preferred
- Other filtering approach, such as client-side

## Pagination (if applicable)
How to handle large result sets, jobToken, pagination-offset.

## Error Recovery
| Error | Recovery |
|-------|----------|
| "Session expired" | Ask user to run `fcli {product} session login` locally |
| ... | ... |

## Decision Tree: Choosing the Right Approach
| User Intent | Action |
|-------|----------|
| "list/show vulnerabilities" | `issue_list` with `--filter` + `--embed details` |
| ... | ... |

## Best Practices
**DO:**
- ✅ ...
- ✅ ...

**Do NOT:**
- ❌ ...
- ❌ ...


## References
#### Example Workflows
| Workflow | Use When User Says... |
|----------|----------------------|
| [Provide Recommendations](references/provide-recommendations.md) | "show recommendations", "provide remediation advice", "how to fix" |
| ... |-=...|
### External Resources
- [FCLI Documentation](https://fortify.github.io/fcli/)
```

---

## Product-Specific Considerations

### Authentication Differences
| Product | Session Check | Login Command |
|---------|---------------|---------------|
| SSC | `fcli_ssc_session_list` | `fcli ssc session login --url <URL> -u <user> -p <pass>` |
| SC-SAST | `fcli_ssc_session_list` | `fcli ssc session login --ssc-url <URL> --ssc-user <user> -p <pass> --client-auth-token <token>` |
| SC-DAST | `fcli_ssc_session_list` | `fcli ssc session login --ssc-url <URL> --ssc-user <user> -p <pass>` |
| FoD | `fcli_fod_session_list` | `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Writing Guidelines

1. **Show parameter formats with examples** - Users need to know exact syntax
2. **Include anti-patterns** - What NOT to do is as important as what to do
3. **Provide error recovery** - Common errors and how to handle them
4. **Reference examples, don't inline** - Keep SKILL.md concise, put details in examples/