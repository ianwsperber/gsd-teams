# Contributing to gsd-teams

Thanks for your interest in contributing to gsd-teams. This guide covers the basics.

## Getting Started

1. Fork and clone the repository
2. Create a feature branch: `git checkout -b my-feature`
3. Make your changes
4. Test locally by installing from your local path
5. Open a pull request

## Making Changes

### Commands

All commands live in `plugins/teams/commands/` as Markdown files. Commands are user-invoked (slash commands), not auto-detected skills.

To add a new command:
- Create a `.md` file in the commands directory
- The filename becomes the command suffix (e.g., `share.md` → `/gsd-teams:share`)
- Include YAML frontmatter with `allowed_tools` and `description`

### Agents

Agent files live in `plugins/teams/agents/` and follow GSD agent structure:
- YAML frontmatter (allowed_tools, description)
- Role and philosophy sections
- Context and execution_flow with XML-style step tags
- Structured returns and success criteria

## Pull Requests

- Write a clear description of what your PR does and why
- Test locally by installing the plugin from your local path:
  ```bash
  /plugin install /path/to/your/gsd-teams@gsd-teams
  ```
- Keep PRs focused -- one concern per PR
- Ensure no regressions to existing commands

## Code Style

- Use YAML frontmatter in all command and agent Markdown files
- Use XML-style tags (`<step>`, `<context>`, etc.) for structural sections
- Use fenced code blocks with language identifiers for shell examples
- Keep lines under 120 characters where practical
- Follow the patterns established in existing files

## Reporting Issues

Use GitHub Issues. Please include:
- What you expected vs what happened
- Your Claude Code version (`claude --version`)
- Your GSD version (check your `.claude/commands/` directory)
- Steps to reproduce
