# Claude Code Workflow v2.0

A set of rules and tool configurations to optimize Claude Code CLI workflow efficiency.

## Features

- **Task Level System**: Automatically determine task complexity (Level 0/1/2) and select appropriate workflow
- **Loop Execution**: Level 2 complex tasks use ralph-loop for automatic iteration until completion
- **Auto Documentation**: Proactively fetch third-party library docs via Context7 MCP
- **PR Review Tools**: Multi-angle code review (security, type design, test coverage, etc.)
- **Code Simplifier**: Auto cleanup and simplify code after long sessions
- **Project Context**: Persist project config and verified/disproven approaches

## Key Improvements in v2.0

- **English Rules**: All rules written in English for better LLM recognition
- **Chinese Response**: Responses to user always in Chinese (用户交互使用中文)
- **Tool Trigger Rules**: Explicit rules for Context7, pr-review-toolkit, code-simplifier, ralph-wiggum
- **Pre-Commit Checklist**: Mandatory checklist before every git commit
- **Startup Checklist**: Session startup checklist for consistency

## Installation

### 1. Add Plugin Marketplace

```bash
claude plugin marketplace add anthropics/claude-code
claude plugin marketplace add anthropics/claude-plugins-official
```

### 2. Install Plugins

```bash
claude plugin install ralph-wiggum@claude-code-plugins
claude plugin install security-guidance@claude-code-plugins
claude plugin install pr-review-toolkit@claude-plugins-official
claude plugin install code-simplifier@claude-plugins-official
```

### 3. Configure MCP Service

```bash
claude mcp add -s user context7 npx @upstash/context7-mcp@latest
```

### 4. Verify MCP Configuration

```bash
claude mcp list
```

### 5. Copy Configuration Files

Copy files from this repository to your Claude configuration directory:

```
# Windows
C:\Users\<username>\.claude\CLAUDE.md
C:\Users\<username>\.claude\startup-checklist.md
C:\Users\<username>\.claude\commands\init.md
C:\Users\<username>\.claude\commands\riper.md

# Mac/Linux
~/.claude/CLAUDE.md
~/.claude/startup-checklist.md
~/.claude/commands/init.md
~/.claude/commands/riper.md
```

## File Structure

```
.
├── CLAUDE.md                    # Core workflow rules (English)
├── README.md                    # This file
├── startup-checklist.md         # Session startup checklist
└── .claude/
    └── commands/
        ├── init.md              # /init command - Initialize session
        └── riper.md             # /riper command - Activate engineering protocol
```

## Available Commands

| Command | Purpose |
|---------|---------|
| /init | Initialize session, detect project |
| /riper | Activate engineering protocol |
| /ralph-wiggum:ralph-loop <task> | Start loop execution |
| /ralph-wiggum:cancel-ralph | Cancel loop |
| /pr-review-toolkit:review-pr | PR review |
| /code-simplifier | Code simplification |

## Usage

### Start Session

In any project directory:

```
/init
```

This will:
1. Detect project tech stack
2. Check for project-context.md
3. Report system environment and ready state

### Task Levels

| Level | Characteristics | Workflow |
|-------|-----------------|----------|
| 0 | Single file, <10 lines, no logic change | Execute directly |
| 1 | Multi-file, logic changes, clear scope | Declare plan → Execute → Verify |
| 2 | Cross-module, architecture change | Full planning → User confirm → ralph-loop |

### Activate Engineering Protocol

When deep analysis and multi-role thinking is needed:

```
/riper
```

## Core Rules

1. **Read Before Edit** - MUST read file before modifying
2. **Check Before Create** - Check if exists before creating
3. **Verify After Change** - Verify results after modification
4. **No Repeated Failures** - Never retry disproven approaches
5. **Only Change Requested** - Never "improve" other code incidentally

## Tool Trigger Rules

| Trigger | Tool | Action |
|---------|------|--------|
| Third-party library | Context7 | Query BEFORE coding |
| API error | Context7 | Query IMMEDIATELY |
| "review", "审查" | pr-review-toolkit | Invoke |
| Level 2 + >100 lines | code-simplifier | Suggest |
| Level 2 task | ralph-wiggum | Use ralph-loop |

## Pre-Commit Checklist

Before every git commit:
- [ ] Does project-context.md need update?
- [ ] Should code-simplifier be suggested?
- [ ] Have all modified files been declared?
- [ ] Has code been verified to run?

## Changelog

### v2.0 (2026-01-12)
- Rewrote all rules in English for better recognition
- Added tool trigger rules section
- Added pre-commit checklist
- Added startup checklist file
- Maintained Chinese response requirement

### v1.0
- Initial Chinese version

## License

MIT
