# Claude Session Startup Checklist

## ⚠️ MANDATORY - Complete before ANY user task

### Session Start:
□ 1. Check if `project-context.md` exists
□ 2. If exists → READ it before any action
□ 3. If not exists → Ask user to initialize

### Before Starting Task:
□ 4. Identify task Level (0/1/2)
□ 5. DECLARE Level in response
□ 6. Level 1/2 → Declare execution plan first
□ 7. Level 2 → Use ralph-loop

### Before Editing Files:
□ 8. READ the file first (Rule 5.1)
□ 9. DECLARE list of files to modify (Rule 5.6)

### After Editing Files:
□ 10. REPORT what was changed (Rule 5.6)
□ 11. VERIFY code runs without errors (Rule 5.3)

---

## Pre-Commit Checklist:

□ 12. Does project-context.md need update? (Rule 4.2)
□ 13. Should code-simplifier be suggested? (Rule 3.3)
□ 14. All files declared before editing?
□ 15. Code verified to run?

---

## Tool Trigger Reminders:

| Trigger | Tool |
|---------|------|
| Third-party library | Context7 BEFORE coding |
| API error | Context7 IMMEDIATELY |
| "review", "审查" | pr-review-toolkit |
| Level 2 completed + >100 lines | Suggest code-simplifier |
| Level 2 task | ralph-loop |

---

**STOP and verify completion before proceeding.**
