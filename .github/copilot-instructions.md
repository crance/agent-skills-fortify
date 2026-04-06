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
2. **One skill per deployment model** - `fortify-onprem` covers all on-prem products (SSC + SC-SAST + SC-DAST); `fortify-fod` covers SaaS
3. **Accept necessary duplication** - Small duplication is acceptable for reliability
4. **Progressive disclosure** - Put detailed workflows, scripts, etc in `references/` folder, referenced from SKILL.md

### Folder Structure
```
skills/
├── fortify-fod/
│   ├── SKILL.md              # Main skill file (loaded when skill activated)
│   └── references/           # Detailed workflows (loaded on-demand)
│       └── *.md
└── fortify-onprem/
    ├── SKILL.md              # Combined On-Prem skill (SSC + SC-SAST + SC-DAST)
    └── references/           # Detailed workflows (loaded on-demand)
        └── *.md
```

### Optimizing Examples for Progressive Disclosure

The LLM decides to fetch examples based on **keyword matching** between user requests and reference text. In `SKILL.md`, use a keyword-rich references table with a "Use When User Says..." column. In each reference file, open with a `## Use Case` section and a `## Keywords` line listing relevant terms.

---

## SKILL.md Template Structure

See existing skill files as canonical examples of the required structure:
- `skills/fortify-fod/SKILL.md` — FoD (SaaS) skill
- `skills/fortify-onprem/SKILL.md` — combined On-Prem skill (SSC + SC-SAST + SC-DAST)

Required sections in every skill: YAML frontmatter (`name`, `description`, `metadata.version`), Available MCP Tools, Parameter Formats, Authentication, Filtering, Error Recovery, Decision Tree, Best Practices, References (workflows table + external links).

---

## Product-Specific Considerations

### Authentication Differences
| Product | Session Check | Login Command |
|---------|---------------|---------------|
| SSC | `fcli_ssc_session_list` | `fcli ssc session login --url <URL> -u <user> -p <pass>` |
| SC-SAST | `fcli_ssc_session_list` | `fcli ssc session login --url <URL> -u <user> -p <pass> --sc-sast-url <SC-SAST-URL> --client-auth-token <token>`  |
| SC-DAST | `fcli_ssc_session_list` | `fcli ssc session login --url <URL> -u <user> -p <pass>` |
| FoD | `fcli_fod_session_list` | `fcli fod session login --url <URL> --client-id <id> --client-secret <secret>` |

---

## Writing Guidelines

1. **Show parameter formats with examples** - Users need to know exact syntax
2. **Include anti-patterns** - What NOT to do is as important as what to do
3. **Provide error recovery** - Common errors and how to handle them
4. **Reference examples, don't inline** - Keep SKILL.md concise, put details in examples/