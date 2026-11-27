# Bruno Skills
My agent skills are stored in this repository. Each skill is a separate folder with its own code and configuration files.

## Skills

| Skill | Description | Platforms |
|-------|-------------|-----------|
| `springboot-upgrade` | Upgrade a Maven Spring Boot application incrementally using the self-contained workflow and pinned OpenRewrite recipes. | All |
| `quarkus-upgrade` | Upgrade a Maven Quarkus application incrementally using the self-contained workflow, pinned OpenRewrite recipes, and the Quarkus update command. | All |

## Quick Start

### Option 1: Install with npx (Recommended)

**Install individual skills:**
```bash
# Spring Boot upgrade
npx skills install github:bantunes82/skills/springboot-upgrade

# Quarkus upgrade
npx skills install github:bantunes82/skills/quarkus-upgrade
```
**Managing skills:**
```bash
# List installed skills
npx skills list

# For more details about the skill commands
npx skills --help 
```

### Option 2: Manual Installation

**1. Clone this repo:**
```bash
# for OpenCode, GitHub Copilot, Codex and Gemini CLI
git clone https://github.com/bantunes82/skills.git ~/.agents/skills/springboot-upgrade
```

```bash
# for Claude Code
git clone https://github.com/bantunes82/skills.git ~/.claude/skills/springboot-upgrade
```

### Usage

```
> Upgrade this SpringBoot application
> /springboot-upgrade
> Upgrade this Quarkus application
> /quarkus-upgrade
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

### quarkus-upgrade
Upgrade a Maven Quarkus application incrementally using the self-contained workflow, pinned OpenRewrite recipes, and the Quarkus update command.

Features:
- Incremental upgrade of Quarkus applications from 1.x or older 2.x to the latest 2.x (OpenRewrite `Quarkus1to2Migration`), from 2.x to the latest 3.x (`quarkus-maven-plugin:update` pinned to the 3.40 stream), and from 3.x to the latest 4.x (`quarkus-maven-plugin:update` pinned to the 4.0 stream).
- Raise the project to Java 17 (OpenRewrite `UpgradeToJava17` + `JavaVersion17`) before moving to Quarkus 3.x when it targets an older Java version; stop and ask when a Quarkus 1.x project still targets Java 8.
- Before moving to Quarkus 4.x on a project below Java 21, ask whether to move to Java 21 (the new minimum) or Java 25 (recommended), then raise it with the OpenRewrite recipes bundled in `quarkus-upgrade/assets/`.
- Use the official [Quarkus migration guides](https://github.com/quarkusio/quarkus/wiki/Migration-Guides) to fix any issue the recipes or the Quarkus update leave behind, citing the guide section for each manual fix.
- Report all the changes applied by the OpenRewrite recipes / Quarkus update and the ones applied by the agent itself to fix the issues that were not fixed by them.
- Run any validation/test/how to run checklist described in the `README.md`, `AGENTS.md`, and `CLAUDE.md` after each upgrade step to ensure the application is still working as expected.
- Run smoke tests (health, OpenAPI/Swagger UI, REST endpoints, security, web UI, Dev UI) after each upgrade step to ensure the application is still working as expected.
- Run the agent skill in a self-contained workflow that does not require any external dependencies or tools, except for Maven and Java.
