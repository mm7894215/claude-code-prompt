# 09 - Prompt 组装流程与缓存机制

> **源文件**: `constants/prompts.ts`, `constants/systemPromptSections.ts`
>
> 本文档详细解析 System Prompt 的完整组装流程、缓存边界设计和动态 Section 注册机制。

---

## 1. 组装总流程

```mermaid
flowchart TD
    Entry["getSystemPrompt(options)"] --> SimpleCheck{"CLAUDE_CODE_SIMPLE?"}

    SimpleCheck -->|"Yes"| MinimalPrompt["返回最小 prompt<br/>仅 CWD + Date + Git"]
    SimpleCheck -->|"No"| LoadInputs["并行加载输入"]

    LoadInputs --> I1["loadSkillToolCommands()"]
    LoadInputs --> I2["getOutputStyleConfig()"]
    LoadInputs --> I3["computeSimpleEnvInfo()"]
    LoadInputs --> I4["getMemorySection()"]

    I1 --> BuildCheck{"isProactive?"}
    I2 --> BuildCheck
    I3 --> BuildCheck
    I4 --> BuildCheck

    BuildCheck -->|"Yes"| ProactivePath["自主工作模式<br/>getProactiveSection()"]
    BuildCheck -->|"No"| NormalPath["标准路径"]

    NormalPath --> Sections["组装 Section 数组"]

    Sections --> Static["=== 静态 Sections ===<br/>(可跨组织缓存)"]
    Static --> S1["getSimpleIntroSection()"]
    Static --> S2["getSimpleSystemSection()"]
    Static --> S3["getSimpleDoingTasksSection()"]
    Static --> S4["getActionsSection()"]
    Static --> S5["getUsingYourToolsSection()"]
    Static --> S6["getSimpleToneAndStyleSection()"]
    Static --> S7["getOutputEfficiencySection()"]

    S7 --> Boundary["DYNAMIC_BOUNDARY 标记"]

    Boundary --> Dynamic["=== 动态 Sections ===<br/>(用户/会话特定)"]
    Dynamic --> D1["getSessionSpecificGuidanceSection()"]
    Dynamic --> D2["loadMemoryPrompt()"]
    Dynamic --> D3["getAntModelOverrideSection()"]
    Dynamic --> D4["computeSimpleEnvInfo()"]
    Dynamic --> D5["getLanguageSection()"]
    Dynamic --> D6["getOutputStyleSection()"]
    Dynamic --> D7["getMcpInstructionsSection()"]
    Dynamic --> D8["getScratchpadInstructions()"]
    Dynamic --> D9["getFunctionResultClearingSection()"]
    Dynamic --> D10["SUMMARIZE_TOOL_RESULTS_SECTION"]

    Dynamic --> CoordCheck{"Coordinator Mode?"}
    CoordCheck -->|"Yes"| CoordPrompt["替换为 Coordinator Prompt<br/>完全不同的身份和工具集"]
    CoordCheck -->|"No"| Final["最终 SystemPromptPart[] 数组"]

    style Static fill:#2ecc71,color:#fff
    style Dynamic fill:#3498db,color:#fff
    style Boundary fill:#e74c3c,color:#fff
```

---

## 2. 缓存边界 (DYNAMIC BOUNDARY)

### 2.1 为什么需要缓存边界

Claude API 支持 Prompt Caching，可以缓存 System Prompt 避免重复计算 token。但只有完全相同的前缀才能命中缓存。

System Prompt 被分为两部分:

| 区域 | 缓存范围 | 内容特征 |
|------|---------|---------|
| **标记前** (静态) | `scope: 'global'` — 跨组织/会话可复用 | 身份、规则、工具使用、风格等通用内容 |
| **标记后** (动态) | `scope: 'session'` — 仅当前会话 | 环境信息、Memory、语言偏好、MCP 指令等 |

### 2.2 标记格式

```
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
```

### 2.3 缓存效果

```mermaid
sequenceDiagram
    participant API as Claude API
    participant Cache as Prompt Cache

    Note over API: Turn 1: 全量计算
    API->>Cache: 缓存静态部分 (scope: global)
    API->>Cache: 缓存动态部分 (scope: session)

    Note over API: Turn 2: 命中缓存
    API->>Cache: 静态部分命中 global cache
    API->>Cache: 动态部分命中 session cache

    Note over API: 新会话 Turn 1
    API->>Cache: 静态部分命中 global cache (复用!)
    API->>Cache: 动态部分需重新计算
```

---

## 3. Section 注册机制

**源文件**: `constants/systemPromptSections.ts`

### 3.1 两种 Section 类型

```mermaid
flowchart LR
    subgraph Cached["systemPromptSection()"]
        C1["计算一次后缓存"]
        C2["整个会话内复用"]
        C3["仅 /clear 时重置"]
        C4["不影响 prompt cache"]
    end

    subgraph Uncached["DANGEROUS_uncachedSection()"]
        U1["每轮重新计算"]
        U2["会破坏 prompt cache"]
        U3["用于真正需要实时更新的数据"]
        U4["命名含 DANGEROUS 前缀警示"]
    end

    style Cached fill:#2ecc71,color:#fff
    style Uncached fill:#e74c3c,color:#fff
```

### 3.2 核心 API

```typescript
// 缓存型: 计算一次后在会话内持续重用
systemPromptSection(key: string, compute: () => string | null): string | null

// 非缓存型: 每轮都重新计算 (谨慎使用)
DANGEROUS_uncachedSystemPromptSection(key: string, compute: () => string | null): string | null

// 清除所有缓存 (通常在 /clear 时调用)
clearCachedSystemPromptSections(): void
```

### 3.3 已知的动态 Section

| Section Key | 类型 | 说明 |
|-------------|------|------|
| `mcpInstructions` | 可选缓存/增量 | MCP 服务指令 |
| `memory` | 缓存 | CLAUDE.md 持久化记忆 |
| `envInfo` | 缓存 | 环境信息 |
| `language` | 缓存 | 语言偏好 |
| `outputStyle` | 缓存 | 输出风格 |
| `scratchpad` | 缓存 | 临时目录 |
| `sessionGuidance` | 缓存 | 会话特定指导 |
| `frc` | 缓存 | 函数结果清理 |

---

## 4. SystemPromptPart 结构

```typescript
interface SystemPromptPart {
  content: string          // prompt 文本内容
  scope: 'global' | 'session'  // 缓存范围
}
```

组装完成后返回的是 `SystemPromptPart[]` 数组:

```
[
  { content: "Identity + System + ...", scope: "global" },   // 静态部分
  { content: "__DYNAMIC_BOUNDARY__",    scope: "global" },   // 边界标记
  { content: "Session guidance + ...",  scope: "session" },  // 动态部分
]
```

---

## 5. CLAUDE_CODE_SIMPLE 模式

当设置环境变量 `CLAUDE_CODE_SIMPLE=1` 时，返回极简 prompt:

```
# System

Environment:
- Working directory: {cwd}
- Today's date: {date}
- {isGit ? "This is a git repo" : "Not a git repo"}
```

> 这个模式用于极端精简场景，跳过所有复杂的 Section 组装。

---

## 6. Proactive 模式路径

当检测到 `isProactiveActivity` 时，走完全不同的 prompt 路径:

```mermaid
flowchart TD
    Proactive["getProactiveSection()"] --> PA["自主工作身份定义"]
    PA --> PB["Pacing 规则 (Sleep 控制节奏)"]
    PB --> PC["First wake-up (问候用户)"]
    PC --> PD["Bias toward action (倾向行动)"]
    PD --> PE["Concise output (简洁输出)"]
    PE --> PF["Terminal focus (焦点状态)"]
    PF --> Final2["+ 环境信息 + Memory + 其他动态 Section"]
```

---

## 7. Coordinator 模式路径

当 `isCoordinatorMode()` 返回 `true` 时，**完全替换** 标准的 System Prompt:

```mermaid
flowchart TD
    CoordCheck{"isCoordinatorMode()?"}
    CoordCheck -->|"Yes"| Replace["替换为 Coordinator Prompt"]
    Replace --> CR1["角色: 协调器, 不直接写代码"]
    Replace --> CR2["工具: Agent + SendMessage + TaskStop"]
    Replace --> CR3["工作流: Research → Synthesis → Implementation → Verification"]
    Replace --> CR4["Worker Prompt 编写指导"]

    CoordCheck -->|"No"| Normal["标准 System Prompt"]
```

Coordinator 模式同样会附加动态 Section (环境信息、MCP 指令等)，但核心身份和工具集完全不同。

---

## 8. 完整组装顺序表

| 序号 | Section | 函数/来源 | 缓存 Scope | 条件 |
|------|---------|-----------|-----------|------|
| 1 | Identity / Intro | `getSimpleIntroSection()` | global | 始终 |
| 2 | System | `getSimpleSystemSection()` | global | 始终 |
| 3 | Doing Tasks | `getSimpleDoingTasksSection()` | global | 始终 |
| 4 | Actions | `getActionsSection()` | global | 始终 |
| 5 | Using Tools | `getUsingYourToolsSection()` | global | 始终 |
| 6 | Tone and Style | `getSimpleToneAndStyleSection()` | global | 始终 |
| 7 | Output Efficiency | `getOutputEfficiencySection()` | global | 始终 |
| -- | **DYNAMIC BOUNDARY** | 标记 | -- | -- |
| 8 | Session Guidance | `getSessionSpecificGuidanceSection()` | session | 始终 |
| 9 | Memory | `loadMemoryPrompt()` | session | 有 CLAUDE.md 时 |
| 10 | Ant Model Override | `getAntModelOverrideSection()` | session | USER_TYPE=ant |
| 11 | Environment | `computeSimpleEnvInfo()` | session | 始终 |
| 12 | Language | `getLanguageSection()` | session | 有语言偏好时 |
| 13 | Output Style | `getOutputStyleSection()` | session | 有输出风格时 |
| 14 | MCP Instructions | `getMcpInstructionsSection()` | session | 有 MCP 服务时 |
| 15 | Scratchpad | `getScratchpadInstructions()` | session | 功能开启时 |
| 16 | Function Result Clearing | `getFunctionResultClearingSection()` | session | 始终 |
| 17 | Summarize Tool Results | `SUMMARIZE_TOOL_RESULTS_SECTION` | session | 功能开启时 |
