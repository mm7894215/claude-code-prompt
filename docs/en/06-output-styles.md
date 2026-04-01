# 06 - Output Styles

> **Source**: `constants/outputStyles.ts`
>
> The output style system modifies how Claude Code communicates by injecting style-specific prompts into the System Prompt.

---

## Style Loading Priority

```mermaid
flowchart LR
    B["Built-in"] --> P["Plugin"] --> U["User<br/>~/.claude/outputStyles/"] --> Proj["Project<br/>.claude/outputStyles/"] --> M["Managed Policy"]
    M --> Active["Active outputStyle"]
    Active --> Inject["Injected into System Prompt"]

    style M fill:#e74c3c,color:#fff
```

---

## 1. Default Style

No custom output style. Behavior controlled by Tone & Style + Output Efficiency sections.

---

## 2. Explanatory Style

```
Provide educational insights about the codebase using bordered insight blocks:

"★ Insight ─────────────────────────────────
[2-3 key educational points about the code, pattern, or concept]
─────────────────────────────────────────────"

Guidelines:
- Keep insights concise (2-3 points max)
- Focus on "why" not just "what"
- Highlight patterns, anti-patterns, or architectural choices
```

---

## 3. Learning Style

```
Help users learn through hands-on practice. Ask for small code contributions
(2-10 lines) when generating larger blocks (20+ lines):

"● **Learn by Doing**
**Context:** [what's built and why this decision matters]
**Your Task:** [specific function, file, TODO(human)]
**Guidance:** [trade-offs and constraints to consider]"

Rules:
- Target meaningful decisions, not boilerplate
- One request at a time
- If human declines, complete the code yourself
```
