# Claude Workflow v2.0

> This workflow maximizes Claude's effectiveness, not restricts capabilities.

---

## ⚠️ CRITICAL: Language Rules

**MUST respond to user in Chinese (中文回复)**
- All responses, explanations, and communications MUST be in Chinese
- Code comments follow project convention (usually English)
- This rule applies regardless of the language used in this instruction file

---

## ⚠️ CRITICAL: Session Startup Check (MANDATORY - DO NOT SKIP)

At the start of EVERY new session, I MUST:
1. Check if `project-context.md` exists in current directory
2. If NOT exists → Ask: "Detected new project (no project-context.md). Initialize project config?"
3. If EXISTS → MUST read and load project context BEFORE any action

**STOP: Confirm this check is completed before proceeding with any task.**

This is a MANDATORY rule. NEVER skip.

---

## Part 1: Tool Environment

### Global Plugins

| Plugin | Purpose | When to Use |
|--------|---------|-------------|
| ralph-wiggum | Loop execution for complex tasks | DEFAULT for Level 2 tasks |
| security-guidance | Security checks | Always running |
| pr-review-toolkit | Multi-angle PR review | When preparing PR |
| code-simplifier | Code cleanup and simplification | After long sessions / major changes |

### Global Hooks

- **PreToolUse (Edit|Write)**: Block direct edits on main/master branch

### MCP Services

- **Context7**: Fetch latest documentation for third-party libraries

---

## Part 2: Task Workflow

### 2.1 Task Levels (Claude determines, user can override)

| Level | Characteristics | Workflow |
|-------|-----------------|----------|
| 0 | Single file, <10 lines, no logic change (rename, typo fix) | Execute directly |
| 1 | Multi-file, logic changes, clear scope (implement function, fix bug) | Read → Declare plan → Execute → Verify |
| 2 | Cross-module, architecture change, needs iteration (full feature, refactor) | Full planning → User confirm → ralph-loop |

**MUST declare task level in response before starting work.**

### 2.2 ralph-loop Rules

**Activation conditions:**
- Level 2 tasks: MUST use by DEFAULT
- Level 1 tasks: Use when user requests

**PAUSE conditions (stop and ask user when):**
- Need to install new dependencies
- Need to modify config files (package.json, tsconfig.json, etc.)
- Design decisions required
- At 50% of max iterations (10th): Report progress

**Safety valve:**
- Default max: 20 iterations
- After max: PAUSE and ask user to continue

**Completion conditions:**
- Has tests → All related tests pass
- No tests → User-specified condition, or Claude proposes + user confirms

### 2.3 Failure Handling

1. Record failed approach and reason
2. Maintain "excluded approaches" list
3. NEVER retry a disproven approach
4. After 3 failures → STOP and re-analyze root cause

---

## Part 3: Tool Trigger Rules

### 3.1 Context7 (MCP Service)

**MUST query BEFORE writing code when:**
- Task involves third-party libraries
- User mentions "latest", "new version", "upgrade"
- Unfamiliar library usage

**MUST query IMMEDIATELY when encountering:**
- "Module not found" / "Cannot find module"
- "xxx is not a function"
- "xxx is deprecated"
- "No exported member"
- TypeError / AttributeError

**NO need to query:**
- Pure business logic code
- Language built-in features
- Same library already queried in this session

**Query failure handling:**
- Found docs → Use latest info
- Not found → Continue with existing knowledge, inform user "Docs not found, using existing knowledge"
- NEVER get stuck because docs not found

**Core principle:** Don't wait for user request, don't wait for multiple failures. Query proactively and early.

### 3.2 pr-review-toolkit (MANDATORY)

**MUST invoke `/pr-review-toolkit:review-pr` when user intent is code review:**
- Explicit review request: "review", "check quality", "审查", "review 一下"
- PR operations: "preparing PR", "before merge", "要合并了"

**Exception (can answer directly):**
- Single point questions: "Is this variable name correct?"
- Status inquiry: "PR status?"

### 3.3 code-simplifier (PROACTIVE SUGGESTION)

**MUST suggest (not force) when:**
- Level 2 task completed + new code > 100 lines
- ralph-loop finished + modified files > 3
- User says "simplify", "cleanup", "清理" → Invoke directly

### 3.4 ralph-wiggum (DEFAULT FOR LEVEL 2)

**MUST use ralph-loop for:**
- All Level 2 tasks (cross-module, architecture changes)
- Level 1 tasks when user requests

**PAUSE conditions during ralph-loop:**
- Need to install new dependencies
- Need to modify config files
- Design decisions required
- At 50% of max iterations (report progress)

---

## Part 4: Project Standards

### 4.1 New Project Initialization

When project has no `project-context.md`, proactively ask:

```
Detected new project (no project-context.md).
Initialize project config? I will:
1. Create project-context.md
2. Detect tech stack and suggest LSP installation
3. Create project-level .claude/settings.json

Yes/No?
```

**Tech stack detection rules:**
- tsconfig.json → Suggest typescript-lsp
- pyproject.toml / requirements.txt → Suggest pyright-lsp
- go.mod → Suggest gopls-lsp
- Cargo.toml → Suggest rust-analyzer-lsp

### 4.2 Knowledge Base Update Timing

**MUST suggest update (user confirms before executing) when:**
- New correct method verified
- New incorrect method disproven
- Project config changed

### 4.3 Port Convention

Recommend 9700+ range to avoid conflicts.
Common conflict ports (avoid): 3000, 5173, 5174, 8080, 8000, 3306, 5432

---

## Part 5: Core Rules

### 5.1 Read Before Edit
**MUST** use Read tool to read file BEFORE modifying.
**NEVER** modify based on memory or assumptions.

### 5.2 Check Before Create
**MUST** use Glob/Grep to check if file/function exists BEFORE creating.
**NEVER** duplicate or overwrite existing code.

### 5.3 Verify After Change
**MUST** verify results after every code modification:
- Run relevant commands to confirm no errors
- NEVER assume modification succeeded

### 5.4 Clean Temp Files
At task end, clean all temp files:
- test-*.js, debug-*.js, temp-*.js
- Files in temp/, tmp/ directories
- .bak, .backup files

### 5.5 Only Change What's Requested
- Only modify explicitly requested code
- **NEVER** "optimize" other code incidentally
- **NEVER** add unrequested features

### 5.6 Modification Transparency
- **BEFORE editing:** Declare list of files to be modified
- **AFTER editing:** Report actual changes made
- Recommend atomic commits (one logical unit per commit)

### 5.7 System Environment Detection
At startup, identify current system environment:
- Windows: Use Windows-compatible commands
- Mac/Linux: Use Unix commands

### 5.8 Windows Command Compatibility
When system is Windows, **MUST** use Windows-compatible commands:
- Use `dir` instead of `ls`
- Use `type` instead of `cat`
- Use `copy` instead of `cp`
- Use `del` instead of `rm`
- Don't use `cd /d` in bash environment

---

## Part 6: Pre-Commit Checklist

**⚠️ CRITICAL: MUST complete before EVERY git commit**

### Checklist:
□ 1. Section 4.2 - Does project-context.md need update?
□ 2. Section 3.3 - Should code-simplifier be suggested? (Level 2 + >100 lines or >3 files)
□ 3. Section 5.6 - Have I declared all modified files?
□ 4. Section 5.3 - Have I verified code runs without errors?

**STOP: Complete this checklist before executing git commit.**

---

*Version: 2.0 (English)*
*Last updated: 2026-01-12*
