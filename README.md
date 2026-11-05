# Bruno Skills
My agent skills are stored in this repository. Each skill is a separate folder with its own code and configuration files.

## Skills

| Skill | Description | Platforms |
|-------|-------------|-----------|
| `springboot-upgrade` | Upgrade a Maven Spring Boot application incrementally using the self-contained workflow and pinned OpenRewrite recipes. | All |

## Quick Start

### Option 1: Install with npx (Recommended)

**Install individual skills:**
```bash
# Spring Boot upgrade
npx skills install github:bantunes82/skills/springboot-upgrade
```
**Managing skills:**
```bash
# List installed skills
npx skills list

# For more details about the skill commands
npx skills --help 
```

#### Option 2: Manual Installation

**1. Clone this repo:**
```bash
# for OpenCode, GitHub Copilot, Codex and Gemini CLI
git clone https://github.com/bantunes82/skills.git ~/.agents/skills/springboot-upgrade
```

```bash
# for Claude Code
git clone https://github.com/bantunes82/skills.git ~/.claude/skills/springboot-upgrade
```


#### Usage

```
> Upgrade this SpringBoot application
> /springboot-upgrade
```

Skills auto-discover from `~/.agents/skills/` or `~/.claude/skills/` (for Claude Code) and your project's `skills/` directory.

## Skill Details

### springboot-upgrade
Upgrade a Maven Spring Boot application incrementally using the self-contained workflow and pinned OpenRewrite recipes.

Features:
- Incremental upgrade of Spring Boot applications from version 1.x to 2.x, from 2.x to 3.x, and from 3.x to 4.x.
- Report all the changes applied by the OpenRewrite recipes and the ones applied by the agent itself to fix the issues that were not fixed by the recipes.
- Run any validation/test/how to run checklist described in the `README.md`, `AGENTS.md`, and `CLAUDE.md` after each upgrade step to ensure the application is still working as expected.
- Run some smoke tests to ensure the application is still working as expected after each upgrade step.
- Run the agent skill in a self-contained workflow that does not require any external dependencies or tools, except for Maven and Java.
