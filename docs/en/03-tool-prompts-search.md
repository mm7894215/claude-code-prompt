# 03 - Search Tool Prompts (Glob / Grep / WebSearch / WebFetch)

> These 4 tools cover local file searching and web content retrieval.

---

## 1. GlobTool

**Source**: `tools/GlobTool/prompt.ts` | **Name**: `Glob`

```
- Fast file pattern matching tool for any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- For open-ended search requiring multiple rounds, use Agent tool instead
```

---

## 2. GrepTool

**Source**: `tools/GrepTool/prompt.ts` | **Name**: `Grep`

```
A powerful search tool built on ripgrep.
- ALWAYS use Grep for search. NEVER invoke grep or rg via Bash.
- Supports full regex syntax
- Filter with glob or type parameters
- Output modes: "content", "files_with_matches" (default), "count"
- Use multiline: true for cross-line patterns
- Use Agent tool for open-ended searches requiring multiple rounds
```

---

## 3. WebSearchTool

**Source**: `tools/WebSearchTool/prompt.ts` | **Name**: `WebSearch`

```
- Search the web for up-to-date information
- CRITICAL: MUST include a "Sources:" section with markdown hyperlinks after answering
- Domain filtering is supported
- Only available in the US
- IMPORTANT: Use the correct year (current month: {currentMonthYear}) in queries
```

---

## 4. WebFetchTool

**Source**: `tools/WebFetchTool/prompt.ts` | **Name**: `WebFetch`

```
- Fetches URL content, converts HTML to markdown, processes with AI model
- If MCP-provided web fetch tool is available, prefer that instead
- HTTP URLs auto-upgraded to HTTPS
- Handles redirects (informs user, requires new request)
- For GitHub URLs, prefer gh CLI via Bash
- Includes 15-minute self-cleaning cache
```
