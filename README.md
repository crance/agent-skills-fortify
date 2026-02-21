# Agent Skills for Fortify
Collection of AI Agent skills for Opentext Application Security (Fortify)

## Prerequisites
### FCLI
1. Download the native binary archive or fcli.jar from [releases](https://github.com/fortify/fcli/releases)

**Linux / MacOS / Java**
``` bash
# Linux
curl -sSL https://github.com/fortify/fcli/releases/download/latest/fcli-linux.tgz -o fcli-linux.tgz

# MacOS
curl -sSL https://github.com/fortify/fcli/releases/download/latest/fcli-mac.tgz -o fcli-mac.tgz

# Java
curl -sSL https://github.com/fortify/fcli/releases/download/latest/java.jar -o fcli.jar
```

**Windows**
```powershell
iwr https://github.com/fortify/fcli/releases/download/latest/fcli-windows.zip -OutFile fcli-windows.zip
```

2. Extract the downloaded zip, if applicable
3. MCP Configuration

**VSCode Example (`mcp.json`)**
```json
// use "java -jar fcli.jar" equivalent if using fcli.jar
{
	"servers": {
    "fcli-fod": {
			"type": "stdio",
			"command": "fcli",
			"args": [
				"util",
				"mcp-server",
				"start",
				"-m",
				"fod"
			]
		},
		"fcli-ssc": {
			"type": "stdio",
			"command": "fcli",
			"args": [
				"util",
				"mcp-server",
				"start",
				"-m",
				"ssc"
			]
		}
  }
}
```

## Install a Skill
### Option 1: Manual installation
1. Clone the repository
2. `cd agent-skills-fortify`
3. Add skill to project or personal skills

**Linux**
``` bash
# Project Skill
# -------------
# Example: GitHub Copilot Pro
ln -s $(pwd)/skills/fortify-fod /path/to/project/.github/skills/

# Example: Claude
ln -s $(pwd)/skills/fortify-fod /path/to/project/.claude/skills/

# Personal Skill
# -------------
# Example: GitHub Copilot Pro
ln -s $(pwd)/skills/fortify-fod ~/.copilot/skills/

# Example: Claude
ln -s $(pwd)/skills/fortify-fod ~/.claude/skills/
```

**Windows**
``` powershell
# Project Skill
# -------------
# Example: GitHub Copilot Pro
New-Item -ItemType SymbolicLink -Path "~\.github\skills\fortify-fod" -Target "$pwd\skills\fortify-fod"

# Example: Claude
New-Item -ItemType SymbolicLink -Path "~\.claude\skills\fortify-fod" -Target "$pwd\skills\fortify-fod"

# Personal Skill
# -------------
# Example: GitHub Copilot Pro
New-Item -ItemType SymbolicLink -Path "~\.copilot\skills\fortify-fod" -Target "$pwd\skills\fortify-fod"

# Example: Claude
New-Item -ItemType SymbolicLink -Path "~\.claude\skills\fortify-fod" -Target "$pwd\skills\fortify-fod"
```

### Option 2: Install via npx
```bash
# On-premise
npx skills add https://github.com/crance/agent-skills-fortify --skill fortify-ssc
npx skills add https://github.com/crance/agent-skills-fortify --skill fortify-scsast
npx skills add https://github.com/crance/agent-skills-fortify --skill fortify-scdast

# SaaS
npx skills add https://github.com/crance/agent-skills-fortify --skill fortify-fod
```

## Authentication
**Authenticate to FoD**
```bash
fcli fod session login --url "<URL>" --client-id "<id>" --client-secret '<secret>'
```

**Authenticate to SSC**
```bash
fcli ssc session login --url "<URL>" -u "<user>" -p '<pass>
```
## Verify Setup
After session has been authenticated, try asking the agent one of the following to ensure the skill and MCP are working:
- "List my SSC applications"
- "List my FoD applications"

## Disclaimer

This project is provided as-is for demonstration and educational purposes only. It is **not an official offering** from OpenText or Fortify and is not officially supported in any capacity.

**No Warranty or Liability:** The maintainers assume no liability for any damages, data loss, or issues arising from the use of this project. This project is provided without any warranty of any kind, express or implied. Users are solely responsible for their use and implementation.

**Compatibility:** Compatibility with specific Fortify/OpenText product versions are not guaranteed. API changes and breaking changes may occur without notice.