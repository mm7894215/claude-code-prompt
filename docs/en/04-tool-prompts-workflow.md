# 04 - Workflow Tool Prompts

> These tools support Claude Code's advanced workflows: planning, skill invocation, configuration, scheduling, worktrees, and teams.

---

## 1. EnterPlanModeTool

**Source**: `tools/EnterPlanModeTool/prompt.ts` | **Name**: `EnterPlanMode`

```
Use proactively when starting a non-trivial implementation task.

When to Use:
1. New Feature Implementation
2. Multiple Valid Approaches
3. Code Modifications affecting existing behavior
4. Architectural Decisions
5. Multi-File Changes (more than 2-3 files)
6. Unclear Requirements
7. User Preferences Matter

When NOT to Use:
- Single-line fixes, trivial tweaks
- Very specific, detailed user instructions
- Pure research/exploration (use Agent tool)
```

---

## 2. ExitPlanModeTool

**Name**: `ExitPlanMode`

```
Signals you're done planning and ready for user review.
- Plan must already be written to the plan file
- Do NOT use AskUserQuestion to ask "Is this plan okay?" — that's what this tool does
- Only use for tasks requiring code implementation, not research
```

---

## 3. AskUserQuestionTool

**Name**: `AskUserQuestion`

```
Ask the user questions during execution:
- Gather preferences, clarify instructions, get decisions, offer choices
- Users can always select "Other" for custom text
- Use multiSelect: true for multiple answers
- Recommended option: put first with "(Recommended)" label
- Supports preview field for ASCII mockups, code snippets, diagrams
```

---

## 4. SkillTool

**Source**: `tools/SkillTool/prompt.ts` | **Name**: `Skill`

```
Execute a skill via slash commands (e.g., /commit, /review-pr).
- BLOCKING REQUIREMENT: invoke Skill tool BEFORE generating any other response
- NEVER mention a skill without calling this tool
- Available skills listed in system-reminder messages
```

---

## 5. ConfigTool

**Name**: `Config`

```
Get or set Claude Code configuration settings.
- Get: omit "value" parameter
- Set: include "value" parameter
- Global settings stored in ~/.claude.json
- Project settings stored in settings.json
```

---

## 6. ScheduleCronTool

**Names**: `CronCreate` / `CronDelete` / `CronList`

```
Schedule prompts for future execution using 5-field cron (local timezone).
- One-shot: recurring: false, auto-deletes after firing
- Recurring: recurring: true (default)
- Avoid :00 and :30 marks for fleet load distribution
- durable: true persists to .claude/scheduled_tasks.json
- Recurring tasks auto-expire after 30 days
```

---

## 7. Worktree Tools

### EnterWorktree
```
ONLY use when user explicitly says "worktree".
Creates git worktree inside .claude/worktrees/ with new branch from HEAD.
```

### ExitWorktree
```
Exit worktree session, return to original directory.
action: "keep" (leave intact) or "remove" (delete worktree + branch)
Only operates on worktrees created by EnterWorktree in this session.
```

---

## 8. ToolSearchTool

**Name**: `ToolSearch`

```
Fetches full schemas for deferred tools so they can be called.
- Deferred tools show only names until fetched
- Query: "select:Read,Edit" (exact), "notebook jupyter" (search), "+slack send" (name filter)
- MCP tools always deferred; core tools never deferred
```

---

## 9. SleepTool

**Name**: `Sleep`

```
Wait for a specified duration. User can interrupt.
Prefer over Bash(sleep ...) — doesn't hold a shell process.
Can call concurrently with other tools.
```

---

## 10. TeamCreateTool

**Name**: `TeamCreate`

```
Create a team for multi-agent collaboration.
Workflow: TeamCreate -> TaskCreate -> Spawn teammates -> Assign tasks -> Work -> Shutdown
Messages from teammates delivered automatically.
Teammates go idle between turns — this is normal.
```
