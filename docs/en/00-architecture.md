# 00 - Claude Code Product Architecture Overview

> This document uses Mermaid diagrams to illustrate the complete product architecture, module relationships, and data flows of Claude Code.

---

## 1. Overall System Architecture

```mermaid
graph TB
    subgraph User["User Entry Points"]
        CLI["CLI Terminal"]
        Desktop["Desktop App"]
        WebApp["Web App (claude.ai/code)"]
        IDE["IDE Extensions (VS Code / JetBrains)"]
    end

    subgraph Core["Core Engine"]
        QE["QueryEngine"]
        SP["System Prompt<br/>Assembly"]
        TP["Tool Pool"]
        PM["Permission Manager"]
    end

    subgraph LLM["Model Layer"]
        API["Claude API<br/>(Opus/Sonnet/Haiku)"]
        Cache["Prompt Cache"]
    end

    subgraph Tools["Tool System"]
        FileOps["File Operations<br/>Read/Write/Edit"]
        Search["Search Tools<br/>Glob/Grep"]
        Shell["Shell Execution<br/>Bash/PowerShell"]
        Web["Web Tools<br/>WebFetch/WebSearch"]
        Workflow["Workflow Tools<br/>PlanMode/Skill/Config"]
        MCP["MCP Tools<br/>External Service Integration"]
    end

    subgraph AgentSys["Agent System"]
        Agent["Agent Tool<br/>Agent Launcher"]
        Fork["Fork Mode<br/>Inherits Context"]
        Subagent["Subagent Mode<br/>Independent Context"]
        Coord["Coordinator<br/>Multi-Worker Orchestration"]
        Team["Team System<br/>Multi-Agent Collaboration"]
    end

    subgraph Context["Context Management"]
        Memory["Memory/CLAUDE.md<br/>Persistent Memory"]
        Scratchpad["Scratchpad<br/>Temp File Directory"]
        History["Conversation History"]
        Compact["Auto Compact<br/>Context Compression"]
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

## 2. System Prompt Assembly Flow

```mermaid
flowchart TD
    Start["getSystemPrompt()"] --> Check{"CLAUDE_CODE_SIMPLE?"}
    Check -->|Yes| Simple["Return minimal prompt<br/>(CWD + Date only)"]
    Check -->|No| LoadConfig["Load config in parallel<br/>skillToolCommands<br/>outputStyleConfig<br/>envInfo"]

    LoadConfig --> ProCheck{"Proactive mode?"}
    ProCheck -->|Yes| ProPath["Autonomous work path<br/>autonomous agent prompt"]
    ProCheck -->|No| NormalPath["Standard path"]

    NormalPath --> Static["Static Sections (globally cacheable)"]
    Static --> S1["1. Identity / Intro"]
    Static --> S2["2. System Rules"]
    Static --> S3["3. Doing Tasks"]
    Static --> S4["4. Actions"]
    Static --> S5["5. Using Tools"]
    Static --> S6["6. Tone and Style"]
    Static --> S7["7. Output Efficiency"]

    S7 --> Boundary["=== DYNAMIC BOUNDARY ===<br/>Cache boundary marker"]

    Boundary --> Dynamic["Dynamic Sections (per-session)"]
    Dynamic --> D1["8. Session Guidance"]
    Dynamic --> D2["9. Memory"]
    Dynamic --> D3["10. Environment"]
    Dynamic --> D4["11. Language"]
    Dynamic --> D5["12. Output Style"]
    Dynamic --> D6["13. MCP Instructions"]
    Dynamic --> D7["14. Scratchpad"]
    Dynamic --> D8["15. FRC / Summarize"]

    Dynamic --> Final["Final System Prompt Array"]

    style Boundary fill:#ff6b6b,color:#fff
    style Static fill:#4ecdc4,color:#fff
    style Dynamic fill:#45b7d1,color:#fff
```

---

## 3. Tool System Architecture

```mermaid
graph LR
    subgraph CoreTools["Core File Tools"]
        Read["Read<br/>File reading<br/>Image/PDF/Notebook support"]
        Write["Write<br/>File create/overwrite"]
        Edit["Edit<br/>Exact string replacement"]
    end

    subgraph ShellTools["Shell Tools"]
        Bash["Bash<br/>Command execution<br/>+Sandbox+Background"]
        PS["PowerShell<br/>Windows only"]
    end

    subgraph SearchTools["Search Tools"]
        Glob["Glob<br/>Filename pattern matching"]
        Grep["Grep<br/>File content search<br/>(ripgrep)"]
    end

    subgraph WebTools["Web Tools"]
        WF["WebFetch<br/>URL content retrieval"]
        WS["WebSearch<br/>Web search"]
    end

    subgraph WorkflowTools["Workflow Tools"]
        Plan["EnterPlanMode<br/>Enter planning mode"]
        ExitPlan["ExitPlanMode<br/>Exit planning mode"]
        Skill["Skill Tool<br/>Skill invocation"]
        Config["Config<br/>Configuration management"]
        Ask["AskUserQuestion<br/>User interaction"]
    end

    subgraph ScheduleTools["Scheduling Tools"]
        Cron["CronCreate/Delete/List<br/>Scheduled tasks"]
        Sleep["Sleep<br/>Wait/idle"]
    end

    subgraph WorktreeTools["Worktree Tools"]
        EW["EnterWorktree<br/>Isolated workspace"]
        XW["ExitWorktree<br/>Exit workspace"]
    end

    subgraph AgentTools["Agent Tools"]
        AT["Agent<br/>Subagent/Fork"]
        TS["ToolSearch<br/>Deferred tool loading"]
        TC["TeamCreate<br/>Team creation"]
        SM["SendMessage<br/>Message passing"]
        TaskTools["Task*<br/>Task management"]
    end

    subgraph MCPTools["MCP External Tools"]
        MCPTool["MCP Tool<br/>Dynamic loading"]
    end
```

---

## 4. Agent System Modes

```mermaid
flowchart TD
    UserReq["User Request"] --> MainAgent["Main Agent (Claude Code)"]

    MainAgent --> Decision{"Task Complexity Assessment"}

    Decision -->|"Simple task"| Direct["Direct Execution<br/>Using Core Tools"]
    Decision -->|"Needs planning"| PlanMode["EnterPlanMode<br/>-> Explore -> Design -> User Approval"]
    Decision -->|"Needs parallelism"| AgentTool["Agent Tool"]
    Decision -->|"Multi-worker orchestration"| CoordMode["Coordinator Mode"]

    AgentTool --> ForkOrSub{"Choose Mode"}

    ForkOrSub -->|"omit subagent_type"| Fork["Fork Mode"]
    Fork --> ForkDesc["Inherits parent's full context<br/>Shares prompt cache<br/>Best for: research, implementation"]

    ForkOrSub -->|"specify subagent_type"| Sub["Subagent Mode"]
    Sub --> SubDesc["Fresh independent context<br/>Needs complete prompt<br/>Best for: verification, specialized tasks"]

    CoordMode --> Phase1["Phase 1: Research<br/>Multiple Workers in parallel"]
    Phase1 --> Phase2["Phase 2: Synthesis<br/>Coordinator analyzes findings"]
    Phase2 --> Phase3["Phase 3: Implementation<br/>Workers execute per spec"]
    Phase3 --> Phase4["Phase 4: Verification<br/>Independent Worker verifies"]

    subgraph TeamMode["Team Mode"]
        TeamCreate["TeamCreate"]
        TaskCreate2["TaskCreate"]
        SpawnMates["Agent(team_name, name)<br/>Spawn teammates"]
        Assign["TaskUpdate(owner)<br/>Assign tasks"]
        Work["Teammates work<br/>Complete -> auto-notify"]
        Shutdown["SendMessage<br/>type: shutdown_request"]

        TeamCreate --> TaskCreate2 --> SpawnMates --> Assign --> Work --> Shutdown
    end

    style Fork fill:#4ecdc4,color:#fff
    style Sub fill:#45b7d1,color:#fff
    style CoordMode fill:#f7dc6f,color:#333
```

---

## 5. Permission & Security System

```mermaid
flowchart LR
    subgraph PermModes["Permission Modes"]
        Auto["Auto-approve Mode"]
        Plan2["Plan Mode<br/>Read: auto / Write: approval"]
        Normal["Normal Mode<br/>Approve by type"]
    end

    subgraph Safety["Safety Layer"]
        Cyber["Cyber Risk Instruction<br/>Security boundaries"]
        Sandbox["Bash Sandbox<br/>Filesystem + Network restrictions"]
        Hooks["User Hooks<br/>Event-driven shell commands"]
        Actions["Action Guard<br/>Dangerous action confirmation"]
    end

    subgraph Guardrails["Guardrails"]
        NoURL["No URL guessing"]
        NoDestructive["Destructive ops require confirmation<br/>force push / reset --hard"]
        NoSecrets["No committing secrets<br/>.env / credentials"]
        InjectionCheck["Prompt injection detection<br/>Flag suspicious tool output"]
    end

    PermModes --> Safety
    Safety --> Guardrails
```

---

## 6. Prompt Caching Mechanism

```mermaid
flowchart TD
    SP["System Prompt Array"] --> Split{"Cache Boundary Marker<br/>DYNAMIC_BOUNDARY"}

    Split -->|"Before marker"| Global["Global Cache Scope<br/>Reusable across orgs/sessions"]
    Split -->|"After marker"| Session["Session Cache<br/>User/session specific"]

    Global --> GContent["Identity + System Rules + Task Guidance<br/>+ Actions + Tool Usage + Style<br/>+ Output Efficiency"]

    Session --> SContent["Session Guidance + Memory + Env Info<br/>+ Language + Output Style + MCP Instructions<br/>+ Scratchpad + FRC"]

    subgraph SectionRegistry["Section Registry"]
        Cached["systemPromptSection()<br/>Computed once, cached until /clear"]
        Uncached["DANGEROUS_uncachedSection()<br/>Recomputed every turn, breaks cache"]
    end

    Session --> SectionRegistry

    style Global fill:#2ecc71,color:#fff
    style Session fill:#e74c3c,color:#fff
    style Uncached fill:#e74c3c,color:#fff
```

---

## 7. Tool Deferred Loading Mechanism

```mermaid
sequenceDiagram
    participant User as User
    participant CC as Claude Code
    participant TS as ToolSearch
    participant MCP as MCP Tools

    Note over CC: At startup: only load core tool schemas<br/>(Bash, Read, Edit, Write, Glob, Grep, Agent...)
    Note over CC: MCP tools and shouldDefer tools<br/>show names only, no schemas loaded

    User->>CC: "Send a message via Slack"
    CC->>CC: Identifies need for "slack" related tool
    CC->>TS: ToolSearch("+slack send")
    TS-->>CC: Returns full schema<br/><functions>..slack_send..</functions>
    CC->>MCP: Invoke slack_send tool
    MCP-->>CC: Execution result
    CC->>User: Message sent
```

---

## 8. Conversation Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Init: User starts Claude Code

    Init --> Normal: Standard conversation mode
    Init --> Proactive: Autonomous work mode (KAIROS)
    Init --> Coordinator: Coordinator mode

    Normal --> PlanMode: EnterPlanMode
    PlanMode --> Normal: ExitPlanMode (user approves)

    Normal --> Worktree: EnterWorktree
    Worktree --> Normal: ExitWorktree

    Normal --> AgentSpawn: Agent Tool
    AgentSpawn --> Normal: Notification returns result

    Normal --> AutoCompact: Context approaching limit
    AutoCompact --> Normal: Compress history

    Proactive --> TickLoop: Receive tick heartbeat
    TickLoop --> Proactive: Sleep / continue work
    TickLoop --> UserInteract: User message arrives
    UserInteract --> Proactive: Respond then continue

    Coordinator --> SpawnWorkers: Dispatch Workers
    SpawnWorkers --> WaitNotify: Wait for notifications
    WaitNotify --> Synthesize: Synthesize results
    Synthesize --> SpawnWorkers: Continue dispatching
    Synthesize --> ReportUser: Report to user

    Normal --> [*]: Session ends
    Proactive --> [*]: Session ends
    Coordinator --> [*]: Session ends
```

---

## 9. Output Style System

```mermaid
graph TD
    subgraph Styles["Output Style Config Sources"]
        Builtin["Built-in Styles<br/>Default / Explanatory / Learning"]
        Plugin["Plugin Styles<br/>forceForPlugin"]
        User["User Custom<br/>~/.claude/outputStyles/"]
        Project["Project Level<br/>.claude/outputStyles/"]
        Managed["Managed Policy<br/>policySettings"]
    end

    subgraph Priority["Priority (low -> high)"]
        P1["Built-in"] --> P2["Plugin"]
        P2 --> P3["User"]
        P3 --> P4["Project"]
        P4 --> P5["Managed Policy"]
    end

    Styles --> Merge["Merge all styles"]
    Merge --> Active["Currently active outputStyle"]
    Active --> Inject["Inject into System Prompt<br/># Output Style: {name}<br/>{prompt}"]
```

---

## 10. MCP Integration Architecture

```mermaid
graph LR
    subgraph MCPServers["MCP Server Connections"]
        S1["Feishu MCP"]
        S2["Context7 MCP"]
        S3["Custom MCP"]
    end

    subgraph MCPFlow["MCP Workflow"]
        Connect["Establish connection"]
        Instructions["Get Instructions"]
        ToolList["Get Tool list"]
        Defer["ToolSearch deferred loading"]
    end

    subgraph Injection["Injection Points"]
        SysPrompt["System Prompt<br/># MCP Server Instructions"]
        ToolPool["Tool Pool<br/>MCP tool registration"]
        Delta["mcp_instructions_delta<br/>Incremental update (optional)"]
    end

    MCPServers --> Connect
    Connect --> Instructions
    Connect --> ToolList
    Instructions --> SysPrompt
    Instructions --> Delta
    ToolList --> Defer
    Defer --> ToolPool
```
