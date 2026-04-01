# 08 - Companion & Auxiliary Prompts

> Companion (Buddy) system, Hooks, Language settings, XML Tags, Memory, and other auxiliary prompts.

---

## 1. Companion / Buddy System

**Source**: `buddy/prompt.ts`

```
A small {species} named {name} sits beside the user's input box and occasionally
comments in a speech bubble. You're not {name} — it's a separate watcher.

When the user addresses {name} directly, respond in ONE line or less.
Don't explain that you're not {name}. Don't narrate what {name} might say.
```

---

## 2. Hooks System

```
Users may configure 'hooks' — shell commands triggered by events (tool calls,
prompt submission). Treat hook feedback as coming from the user. If blocked
by a hook, try to adjust or ask the user to check their hooks configuration.
```

---

## 3. Language Setting

```
# Language
Always respond in {languagePreference}. Technical terms and code identifiers
should remain in their original form.
```

---

## 4. XML Tag System

**Source**: `constants/xml.ts`

| Category | Tags | Purpose |
|----------|------|---------|
| Command/Skill | `command-name`, `command-message`, `command-args` | Skill metadata |
| Terminal I/O | `bash-input`, `bash-stdout`, `bash-stderr`, `local-command-*` | Terminal output markers |
| Lifecycle | `tick`, `system-reminder` | Heartbeat, system info |
| Tasks | `task-notification`, `task-id`, `status`, `summary`, `reason` | Background task results |
| Worktree | `worktree`, `worktreePath`, `worktreeBranch` | Worktree state |
| Collaboration | `teammate-message`, `channel-message`, `cross-session-message` | Inter-agent/channel comms |
| Fork | `fork-boilerplate` | Fork child initialization |
| Review | `ultraplan`, `remote-review`, `remote-review-progress` | Remote planning/review |

---

## 5. Memory / CLAUDE.md

```mermaid
flowchart TD
    Global["~/.claude/CLAUDE.md<br/>Global memory"] --> Load["loadMemoryPrompt()"]
    Project["./CLAUDE.md<br/>Project memory"] --> Load
    Local[".claude/CLAUDE.local.md<br/>Local memory (not committed)"] --> Load
    Parent["../CLAUDE.md<br/>Parent directory memory"] --> Load
    Load --> Inject["Inject into System Prompt"]
```

Typically contains: build/test/deploy commands, code style rules, architecture decisions, common workflows.

---

## 6. TodoWrite / Task Management

```
Break down and manage your work with the TodoWrite tool. Mark each task
as completed as soon as you're done — don't batch up multiple tasks.
```

---

## 7. Interactive Shorthand

```
User can use `! <command>` as a prompt to run a shell command with Bash.
Example: `! git log -n 5` runs git log without confirmation.
```
