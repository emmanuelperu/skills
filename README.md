# Agent Skills

[![License](https://img.shields.io/github/license/emmanuelperu/skills)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-compatible-blue)](https://agentskills.io)
[![GitHub stars](https://img.shields.io/github/stars/emmanuelperu/skills)](https://github.com/emmanuelperu/skills/stargazers)
[![Skills](https://img.shields.io/badge/skills-3-green)](skills/)

A collection of agent skills that teach AI coding agents how to work with developer tools in your workflow. Skills are grouped by domain and prefixed accordingly (`microcks-*`, ...), so you can install only the ones you need.

## Skills

### Microcks

[![Microcks](https://img.shields.io/badge/Microcks-CNCF%20Sandbox-9cf)](https://microcks.io)

Skills for [Microcks](https://microcks.io), the CNCF open-source tool for API mocking and testing.

| Skill | Description |
|-------|-------------|
| [microcks-local-server](skills/microcks-local-server/) | Start a local Microcks server with Docker Compose for API mocking and testing |
| [microcks-openapi-mocking](skills/microcks-openapi-mocking/) | Write OpenAPI examples that work with Microcks dispatchers for API mocking |
| [microcks-import-artifacts](skills/microcks-import-artifacts/) | Import OpenAPI specs into Microcks and auto-configure dispatchers |

```bash
npx skills add emmanuelperu/skills -s microcks-local-server,microcks-openapi-mocking,microcks-import-artifacts
```

## Quick Install

```bash
# List the available skills
npx skills add emmanuelperu/skills -l

# Install all skills
npx skills add emmanuelperu/skills

# Install only some skills
npx skills add emmanuelperu/skills -s microcks-local-server

# Install for a specific agent
npx skills add emmanuelperu/skills -a claude-code
npx skills add emmanuelperu/skills -a cursor
npx skills add emmanuelperu/skills -a github-copilot
```

## Compatible Agents

These skills work with any agent that supports the [Agent Skills](https://agentskills.io) specification:

- Claude Code
- Cursor
- GitHub Copilot
- Windsurf
- And more

## Complementary Skills

| Skill | Repository | Description |
|-------|------------|-------------|
| openapi-spec-generation | [wshobson/agents](https://github.com/wshobson/agents) | Patterns for creating and validating OpenAPI specifications |

```bash
npx skills add https://github.com/wshobson/agents --skill openapi-spec-generation
```

## Links

- [Agent Skills](https://agentskills.io): open specification for AI agent skills
- [Microcks](https://microcks.io): API mocking and testing platform

## Acknowledgments

Some parts of these skills were written with the help of [Claude](https://claude.ai).

## License

Apache 2.0
