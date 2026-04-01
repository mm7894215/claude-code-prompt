# 00 - Claude Code 产品功能架构全景图

> 本文档使用 Mermaid 图表展示 Claude Code 的完整产品架构、模块关系和数据流。

---

## 1. 系统总体架构

```mermaid
graph TB
    subgraph User["用户入口"]
        CLI["CLI 终端"]
        Desktop["桌面应用"]
        WebApp["Web App (claude.ai/code)"]
        IDE["IDE 插件 (VS Code / JetBrains)"]
    end

    subgraph Core["核心引擎"]
        QE["QueryEngine<br/>查询引擎"]
        SP["System Prompt<br/>系统提示词组装"]
        TP["Tool Pool<br/>工具池"]
        PM["Permission Manager<br/>权限管理"]
    end

    subgraph LLM["模型层"]
        API["Claude API<br/>(Opus/Sonnet/Haiku)"]
        Cache["Prompt Cache<br/>提示词缓存"]
    end

    subgraph Tools["工具系统"]
        FileOps["文件操作<br/>Read/Write/Edit"]
        Search["搜索工具<br/>Glob/Grep"]
        Shell["Shell 执行<br/>Bash/PowerShell"]
        Web["网络工具<br/>WebFetch/WebSearch"]
        Workflow["工作流工具<br/>PlanMode/Skill/Config"]
        MCP["MCP 工具<br/>外部服务集成"]
    end

    subgraph AgentSys["Agent 系统"]
        Agent["Agent Tool<br/>子代理启动器"]
        Fork["Fork 模式<br/>继承上下文"]
        Subagent["Subagent 模式<br/>独立上下文"]
        Coord["Coordinator<br/>协调器模式"]
        Team["Team 系统<br/>多 Agent 协作"]
    end

    subgraph Context["上下文管理"]
        Memory["Memory/CLAUDE.md<br/>持久化记忆"]
        Scratchpad["Scratchpad<br/>临时文件目录"]
        History["Conversation History<br/>对话历史"]
        Compact["Auto Compact<br/>自动压缩"]
    end

    User --> QE
    QE --> SP
    QE --> TP
    QE --> PM
    SP --> API
    TP --> API
    API --> Cache
    TP --> Tools
    TP --> AgentSys
    Agent --> Fork
    Agent --> Subagent
    Agent --> Coord
    Coord --> Team
    QE --> Context
```

---

## 2. System Prompt 组装流程

```mermaid
flowchart TD
    Start["getSystemPrompt()"] --> Check{"CLAUDE_CODE_SIMPLE?"}
    Check -->|Yes| Simple["返回最小化 prompt<br/>(仅 CWD + Date)"]
    Check -->|No| LoadConfig["加载配置<br/>skillToolCommands<br/>outputStyleConfig<br/>envInfo"]

    LoadConfig --> ProCheck{"Proactive 模式?"}
    ProCheck -->|Yes| ProPath["自主工作模式路径<br/>autonomous agent prompt"]
    ProCheck -->|No| NormalPath["标准路径"]

    NormalPath --> Static["静态 Section (可全局缓存)"]
    Static --> S1["1. Identity / Intro<br/>身份定义"]
    Static --> S2["2. System<br/>系统运行规则"]
    Static --> S3["3. Doing Tasks<br/>任务执行指导"]
    Static --> S4["4. Actions<br/>谨慎执行操作"]
    Static --> S5["5. Using Tools<br/>工具使用指导"]
    Static --> S6["6. Tone and Style<br/>语气风格"]
    Static --> S7["7. Output Efficiency<br/>输出效率"]

    S7 --> Boundary["=== DYNAMIC BOUNDARY ===<br/>缓存分界线"]

    Boundary --> Dynamic["动态 Section (每轮/每次计算)"]
    Dynamic --> D1["8. Session Guidance<br/>会话特定指导"]
    Dynamic --> D2["9. Memory<br/>持久化记忆"]
    Dynamic --> D3["10. Environment<br/>环境信息"]
    Dynamic --> D4["11. Language<br/>语言偏好"]
    Dynamic --> D5["12. Output Style<br/>输出风格"]
    Dynamic --> D6["13. MCP Instructions<br/>MCP 服务指令"]
    Dynamic --> D7["14. Scratchpad<br/>临时目录"]
    Dynamic --> D8["15. FRC / Summarize<br/>结果清理与总结"]

    Dynamic --> Final["最终 System Prompt 数组"]

    style Boundary fill:#ff6b6b,color:#fff
    style Static fill:#4ecdc4,color:#fff
    style Dynamic fill:#45b7d1,color:#fff
```

---

## 3. 工具系统架构

```mermaid
graph LR
    subgraph CoreTools["核心文件工具"]
        Read["Read<br/>文件读取<br/>支持图片/PDF/Notebook"]
        Write["Write<br/>文件创建/覆写"]
        Edit["Edit<br/>精确字符串替换"]
    end

    subgraph ShellTools["Shell 工具"]
        Bash["Bash<br/>命令执行<br/>+沙箱+后台"]
        PS["PowerShell<br/>Windows 专用"]
    end

    subgraph SearchTools["搜索工具"]
        Glob["Glob<br/>文件名模式匹配"]
        Grep["Grep<br/>文件内容搜索<br/>(ripgrep)"]
    end

    subgraph WebTools["网络工具"]
        WF["WebFetch<br/>URL 内容获取"]
        WS["WebSearch<br/>网络搜索"]
    end

    subgraph WorkflowTools["工作流工具"]
        Plan["EnterPlanMode<br/>进入规划模式"]
        ExitPlan["ExitPlanMode<br/>退出规划模式"]
        Skill["Skill Tool<br/>技能调用"]
        Config["Config<br/>配置管理"]
        Ask["AskUserQuestion<br/>用户交互"]
    end

    subgraph ScheduleTools["调度工具"]
        Cron["CronCreate/Delete/List<br/>定时任务"]
        Sleep["Sleep<br/>休眠等待"]
    end

    subgraph WorktreeTools["Worktree 工具"]
        EW["EnterWorktree<br/>进入隔离工作区"]
        XW["ExitWorktree<br/>退出工作区"]
    end

    subgraph AgentTools["Agent 工具"]
        AT["Agent<br/>子代理/Fork"]
        TS["ToolSearch<br/>延迟工具加载"]
        TC["TeamCreate<br/>团队创建"]
        SM["SendMessage<br/>消息传递"]
        TaskTools["Task*<br/>任务管理"]
    end

    subgraph MCPTools["MCP 外部工具"]
        MCPTool["MCP Tool<br/>动态加载"]
    end
```

---

## 4. Agent 系统工作模式

```mermaid
flowchart TD
    UserReq["用户请求"] --> MainAgent["主 Agent (Claude Code)"]

    MainAgent --> Decision{"任务复杂度评估"}

    Decision -->|"简单任务"| Direct["直接执行<br/>使用 Core Tools"]
    Decision -->|"需要规划"| PlanMode["EnterPlanMode<br/>→ 探索 → 设计方案 → 用户审批"]
    Decision -->|"需要并行/隔离"| AgentTool["Agent Tool"]
    Decision -->|"多 Worker 协调"| CoordMode["Coordinator 模式"]

    AgentTool --> ForkOrSub{"选择模式"}

    ForkOrSub -->|"omit subagent_type"| Fork["Fork 模式"]
    Fork --> ForkDesc["继承父级全部上下文<br/>共享 prompt cache<br/>适合: 研究、多步实现"]

    ForkOrSub -->|"specify subagent_type"| Sub["Subagent 模式"]
    Sub --> SubDesc["独立空白上下文<br/>需要完整 prompt<br/>适合: 独立验证、专项任务"]

    CoordMode --> Phase1["Phase 1: Research<br/>多个 Worker 并行调研"]
    Phase1 --> Phase2["Phase 2: Synthesis<br/>Coordinator 综合分析"]
    Phase2 --> Phase3["Phase 3: Implementation<br/>Worker 按 spec 实现"]
    Phase3 --> Phase4["Phase 4: Verification<br/>独立 Worker 验证"]

    subgraph TeamMode["Team 模式"]
        TeamCreate["TeamCreate<br/>创建团队"]
        TaskCreate2["TaskCreate<br/>创建任务"]
        SpawnMates["Agent(team_name, name)<br/>生成队友"]
        Assign["TaskUpdate(owner)<br/>分配任务"]
        Work["队友工作<br/>完成 → 自动通知"]
        Shutdown["SendMessage<br/>type: shutdown_request"]

        TeamCreate --> TaskCreate2 --> SpawnMates --> Assign --> Work --> Shutdown
    end

    style Fork fill:#4ecdc4,color:#fff
    style Sub fill:#45b7d1,color:#fff
    style CoordMode fill:#f7dc6f,color:#333
```

---

## 5. 权限与安全体系

```mermaid
flowchart LR
    subgraph PermModes["权限模式"]
        Auto["自动批准模式"]
        Plan2["Plan 模式<br/>读取自动/写入需批准"]
        Normal["Normal 模式<br/>按类型审批"]
    end

    subgraph Safety["安全层"]
        Cyber["Cyber Risk Instruction<br/>安全操作边界"]
        Sandbox["Bash Sandbox<br/>文件系统+网络限制"]
        Hooks["User Hooks<br/>事件钩子"]
        Actions["Action Guard<br/>危险操作确认"]
    end

    subgraph Guardrails["护栏规则"]
        NoURL["禁止猜测 URL"]
        NoDestructive["危险操作需确认<br/>force push / reset --hard"]
        NoSecrets["不提交敏感文件<br/>.env / credentials"]
        InjectionCheck["Prompt 注入检测<br/>标记可疑工具输出"]
    end

    PermModes --> Safety
    Safety --> Guardrails
```

---

## 6. Prompt 缓存机制

```mermaid
flowchart TD
    SP["System Prompt 数组"] --> Split{"缓存边界标记<br/>DYNAMIC_BOUNDARY"}

    Split -->|"标记前"| Global["Global Cache Scope<br/>跨组织/会话复用"]
    Split -->|"标记后"| Session["Session Cache<br/>用户/会话特定"]

    Global --> GContent["身份定义 + 系统规则 + 任务指导<br/>+ 操作谨慎 + 工具使用 + 风格<br/>+ 输出效率"]

    Session --> SContent["会话指导 + Memory + 环境信息<br/>+ 语言 + 输出风格 + MCP 指令<br/>+ Scratchpad + FRC"]

    subgraph SectionRegistry["Section 注册器"]
        Cached["systemPromptSection()<br/>计算一次, 缓存到 /clear"]
        Uncached["DANGEROUS_uncachedSection()<br/>每轮重算, 会破坏缓存"]
    end

    Session --> SectionRegistry

    style Global fill:#2ecc71,color:#fff
    style Session fill:#e74c3c,color:#fff
    style Uncached fill:#e74c3c,color:#fff
```

---

## 7. 工具延迟加载机制

```mermaid
sequenceDiagram
    participant User as 用户
    participant CC as Claude Code
    participant TS as ToolSearch
    participant MCP as MCP Tools

    Note over CC: 启动时只加载核心工具 schema<br/>(Bash, Read, Edit, Write, Glob, Grep, Agent...)
    Note over CC: MCP 工具和 shouldDefer 工具<br/>只显示名称, 不加载 schema

    User->>CC: "帮我用 Slack 发消息"
    CC->>CC: 识别到需要 "slack" 相关工具
    CC->>TS: ToolSearch("+slack send")
    TS-->>CC: 返回完整 schema<br/><functions>..slack_send..</functions>
    CC->>MCP: 调用 slack_send 工具
    MCP-->>CC: 执行结果
    CC->>User: 消息已发送
```

---

## 8. 对话生命周期

```mermaid
stateDiagram-v2
    [*] --> Init: 用户启动 Claude Code

    Init --> Normal: 标准对话模式
    Init --> Proactive: 自主工作模式 (KAIROS)
    Init --> Coordinator: 协调器模式

    Normal --> PlanMode: EnterPlanMode
    PlanMode --> Normal: ExitPlanMode (用户批准)

    Normal --> Worktree: EnterWorktree
    Worktree --> Normal: ExitWorktree

    Normal --> AgentSpawn: Agent Tool
    AgentSpawn --> Normal: 通知返回结果

    Normal --> AutoCompact: 上下文接近限制
    AutoCompact --> Normal: 压缩历史消息

    Proactive --> TickLoop: 接收 tick 心跳
    TickLoop --> Proactive: Sleep / 继续工作
    TickLoop --> UserInteract: 用户消息到达
    UserInteract --> Proactive: 响应后继续

    Coordinator --> SpawnWorkers: 派发 Worker
    SpawnWorkers --> WaitNotify: 等待通知
    WaitNotify --> Synthesize: 综合分析结果
    Synthesize --> SpawnWorkers: 继续派发
    Synthesize --> ReportUser: 汇报用户

    Normal --> [*]: 会话结束
    Proactive --> [*]: 会话结束
    Coordinator --> [*]: 会话结束
```

---

## 9. 输出风格系统

```mermaid
graph TD
    subgraph Styles["输出风格配置源"]
        Builtin["内置样式<br/>Default / Explanatory / Learning"]
        Plugin["插件样式<br/>forceForPlugin"]
        User["用户自定义<br/>~/.claude/ outputStyles/"]
        Project["项目级样式<br/>.claude/ outputStyles/"]
        Managed["管理策略样式<br/>policySettings"]
    end

    subgraph Priority["优先级 (低→高)"]
        P1["内置"] --> P2["插件"]
        P2 --> P3["用户级"]
        P3 --> P4["项目级"]
        P4 --> P5["管理策略"]
    end

    Styles --> Merge["合并所有样式"]
    Merge --> Active["当前激活的 outputStyle"]
    Active --> Inject["注入 System Prompt<br/># Output Style: {name}<br/>{prompt}"]
```

---

## 10. MCP 集成架构

```mermaid
graph LR
    subgraph MCPServers["MCP Server 连接"]
        S1["Feishu MCP"]
        S2["Context7 MCP"]
        S3["Custom MCP"]
    end

    subgraph MCPFlow["MCP 工作流"]
        Connect["建立连接"]
        Instructions["获取 Instructions"]
        ToolList["获取 Tool 列表"]
        Defer["ToolSearch 延迟加载"]
    end

    subgraph Injection["注入点"]
        SysPrompt["System Prompt<br/># MCP Server Instructions"]
        ToolPool["Tool Pool<br/>MCP 工具注册"]
        Delta["mcp_instructions_delta<br/>增量更新 (可选)"]
    end

    MCPServers --> Connect
    Connect --> Instructions
    Connect --> ToolList
    Instructions --> SysPrompt
    Instructions --> Delta
    ToolList --> Defer
    Defer --> ToolPool
```
