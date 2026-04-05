# Claude Code — Architecture Flowcharts

> High-fidelity Mermaid diagrams covering every major system, data flow, and protocol in the Claude Code Rust reimplementation.

---

## Table of Contents

1. [System Architecture — Crate Dependency Graph](#1-system-architecture--crate-dependency-graph)
2. [CLI Boot Sequence](#2-cli-boot-sequence)
3. [Agentic Query Loop](#3-agentic-query-loop)
4. [Tool Dispatch & Permission System](#4-tool-dispatch--permission-system)
5. [API Client — Streaming & Retry Logic](#5-api-client--streaming--retry-logic)
6. [Auto-Compaction Flow](#6-auto-compaction-flow)
7. [MCP Client Protocol](#7-mcp-client-protocol)
8. [Bridge Remote Control Protocol](#8-bridge-remote-control-protocol)
9. [TUI Rendering Pipeline](#9-tui-rendering-pipeline)
10. [OAuth Authentication Flow](#10-oauth-authentication-flow)
11. [Buddy Companion Generation](#11-buddy-companion-generation)
12. [Slash Command Dispatch](#12-slash-command-dispatch)
13. [Memory & Context Assembly](#13-memory--context-assembly)
14. [Session Persistence](#14-session-persistence)

---

## 1. System Architecture — Crate Dependency Graph

```mermaid
graph TB
    CLI["<b>cc-cli</b><br/>Binary Entry Point<br/><i>arg parsing · REPL loop · OAuth</i>"]
    CORE["<b>cc-core</b><br/>Foundation<br/><i>types · config · errors · permissions</i>"]
    API["<b>cc-api</b><br/>Anthropic Client<br/><i>SSE streaming · retries · auth</i>"]
    TOOLS["<b>cc-tools</b><br/>32 Tools<br/><i>bash · files · search · web · tasks</i>"]
    QUERY["<b>cc-query</b><br/>Agentic Loop<br/><i>turn execution · compaction · cron</i>"]
    TUI["<b>cc-tui</b><br/>Terminal UI<br/><i>ratatui · input · rendering</i>"]
    COMMANDS["<b>cc-commands</b><br/>Slash Commands<br/><i>33 built-in commands</i>"]
    MCP["<b>cc-mcp</b><br/>MCP Client<br/><i>JSON-RPC 2.0 · stdio transport</i>"]
    BRIDGE["<b>cc-bridge</b><br/>Remote Control<br/><i>JWT auth · long-polling</i>"]
    BUDDY["<b>cc-buddy</b><br/>Companion Pet<br/><i>deterministic gacha · sprites</i>"]

    CLI --> CORE
    CLI --> API
    CLI --> TOOLS
    CLI --> QUERY
    CLI --> TUI
    CLI --> COMMANDS
    CLI --> MCP
    CLI --> BRIDGE
    CLI --> BUDDY

    API --> CORE
    TOOLS --> CORE
    QUERY --> CORE
    QUERY --> API
    QUERY --> TOOLS
    TUI --> CORE
    COMMANDS --> CORE
    MCP --> CORE
    BRIDGE --> CORE
    BUDDY --> CORE

    style CLI fill:#2563eb,stroke:#1e40af,color:#fff
    style CORE fill:#7c3aed,stroke:#5b21b6,color:#fff
    style API fill:#dc2626,stroke:#991b1b,color:#fff
    style TOOLS fill:#ea580c,stroke:#c2410c,color:#fff
    style QUERY fill:#059669,stroke:#047857,color:#fff
    style TUI fill:#0891b2,stroke:#0e7490,color:#fff
    style COMMANDS fill:#4f46e5,stroke:#3730a3,color:#fff
    style MCP fill:#c026d3,stroke:#a21caf,color:#fff
    style BRIDGE fill:#e11d48,stroke:#be123c,color:#fff
    style BUDDY fill:#f59e0b,stroke:#d97706,color:#000
```

---

## 2. CLI Boot Sequence

```mermaid
flowchart TD
    START(["$ claude ..."])
    PARSE["Parse CLI args<br/><i>clap derive</i>"]
    FAST{"Fast-path<br/>check"}
    VERSION["Print version<br/>& exit"]
    AUTH_CMD["OAuth flow<br/><i>login · logout · status</i>"]
    NAMED["Named subcommand<br/><i>agents · ide · etc.</i>"]

    SETUP_LOG["Setup tracing<br/><i>verbose flag</i>"]
    CWD["Resolve working<br/>directory"]
    SETTINGS["Load settings<br/><i>~/.claude/settings.json</i>"]
    MERGE["Merge config<br/><i>CLI args > settings > defaults</i>"]
    CONTEXT["Build context<br/><i>git status · CLAUDE.md</i>"]

    AUTH_RES{"Auth<br/>resolution"}
    CFG_KEY["Use config<br/>API key"]
    ENV_KEY["Use env<br/>ANTHROPIC_API_KEY"]
    OAUTH_TOK["Load saved<br/>OAuth tokens"]
    REFRESH["Silent token<br/>refresh"]
    INTERACTIVE_LOGIN["Interactive<br/>OAuth login"]

    BUILD_CLIENT["Build AnthropicClient<br/><i>Bearer or x-api-key</i>"]
    PERM_HANDLER{"Mode?"}
    AUTO_PERM["AutoPermissionHandler<br/><i>headless defaults</i>"]
    INTER_PERM["InteractivePermissionHandler<br/><i>user-interactive</i>"]

    MCP_CONN["Connect MCP servers"]
    BUILD_TOOLS["Build tool list<br/><i>native + MCP wrappers</i>"]
    CRON["Spawn cron<br/>scheduler"]

    MODE{"Execution<br/>mode?"}
    HEADLESS["run_headless()<br/><i>single query → output → exit</i>"]
    INTERACTIVE["run_interactive()<br/><i>terminal UI · REPL loop</i>"]

    START --> PARSE
    PARSE --> FAST
    FAST -->|"--version"| VERSION
    FAST -->|"auth ..."| AUTH_CMD
    FAST -->|"agents/ide/..."| NAMED
    FAST -->|"default"| SETUP_LOG

    SETUP_LOG --> CWD --> SETTINGS --> MERGE --> CONTEXT

    CONTEXT --> AUTH_RES
    AUTH_RES -->|"config key"| CFG_KEY --> BUILD_CLIENT
    AUTH_RES -->|"env var"| ENV_KEY --> BUILD_CLIENT
    AUTH_RES -->|"saved tokens"| OAUTH_TOK
    OAUTH_TOK -->|"expired"| REFRESH --> BUILD_CLIENT
    OAUTH_TOK -->|"valid"| BUILD_CLIENT
    AUTH_RES -->|"none found"| INTERACTIVE_LOGIN --> BUILD_CLIENT

    BUILD_CLIENT --> PERM_HANDLER
    PERM_HANDLER -->|"headless / --print"| AUTO_PERM
    PERM_HANDLER -->|"interactive"| INTER_PERM

    AUTO_PERM --> MCP_CONN
    INTER_PERM --> MCP_CONN
    MCP_CONN --> BUILD_TOOLS --> CRON

    CRON --> MODE
    MODE -->|"--print"| HEADLESS
    MODE -->|"default"| INTERACTIVE

    style START fill:#2563eb,stroke:#1e40af,color:#fff
    style HEADLESS fill:#059669,stroke:#047857,color:#fff
    style INTERACTIVE fill:#0891b2,stroke:#0e7490,color:#fff
    style VERSION fill:#6b7280,stroke:#4b5563,color:#fff
    style AUTH_CMD fill:#6b7280,stroke:#4b5563,color:#fff
```

---

## 3. Agentic Query Loop

```mermaid
flowchart TD
    ENTRY(["run_query_loop()"])
    INIT["Initialize turn counter<br/><i>turn = 0</i>"]
    CHECK_TURNS{"turn < max_turns?"}
    BUILD["Build API request<br/><i>messages + tools + system prompt</i>"]
    STREAM["Stream from<br/>Anthropic API"]
    ACCUM["StreamAccumulator<br/><i>collect text · tool_use · thinking</i>"]
    COST["Track tokens & cost<br/><i>input · output · cache</i>"]

    STOP{"stop_reason?"}

    END_TURN["Return<br/>EndTurn"]
    MAX_TOK["Return<br/>MaxTokens"]
    CANCEL["Return<br/>Cancelled"]

    TOOL_USE["Extract ToolUse blocks"]
    PRE_HOOK["Fire PreToolUse hooks<br/><i>can block execution</i>"]
    BLOCKED{"Hook<br/>blocked?"}
    BLOCK_MSG["Append blocked<br/>tool result"]

    EXEC_TOOLS["Execute each tool<br/><i>parallel where safe</i>"]
    POST_HOOK["Fire PostToolUse hooks"]
    TOOL_RESULT["Append ToolResult<br/>message to history"]

    COMPACT_CHK{"Context near<br/>90% full?"}
    COMPACT["Auto-compact<br/>conversation"]

    INCREMENT["turn += 1"]
    STOP_HOOK["Fire Stop hooks"]
    MAX_REACHED["Return EndTurn<br/><i>max turns exhausted</i>"]

    EMIT_STREAM["Emit QueryEvents<br/><i>Stream · ToolStart · ToolEnd</i>"]

    ENTRY --> INIT --> CHECK_TURNS
    CHECK_TURNS -->|"yes"| BUILD
    CHECK_TURNS -->|"no"| MAX_REACHED

    BUILD --> STREAM
    STREAM --> ACCUM
    ACCUM --> EMIT_STREAM
    EMIT_STREAM --> COST

    COST --> STOP
    STOP -->|"end_turn"| STOP_HOOK --> END_TURN
    STOP -->|"max_tokens"| MAX_TOK
    STOP -->|"cancelled"| CANCEL
    STOP -->|"tool_use"| TOOL_USE

    TOOL_USE --> PRE_HOOK
    PRE_HOOK --> BLOCKED
    BLOCKED -->|"yes"| BLOCK_MSG --> COMPACT_CHK
    BLOCKED -->|"no"| EXEC_TOOLS

    EXEC_TOOLS --> POST_HOOK
    POST_HOOK --> TOOL_RESULT --> COMPACT_CHK

    COMPACT_CHK -->|"yes"| COMPACT --> INCREMENT
    COMPACT_CHK -->|"no"| INCREMENT

    INCREMENT --> CHECK_TURNS

    style ENTRY fill:#059669,stroke:#047857,color:#fff
    style END_TURN fill:#2563eb,stroke:#1e40af,color:#fff
    style MAX_TOK fill:#f59e0b,stroke:#d97706,color:#000
    style CANCEL fill:#6b7280,stroke:#4b5563,color:#fff
    style MAX_REACHED fill:#dc2626,stroke:#991b1b,color:#fff
    style COMPACT fill:#7c3aed,stroke:#5b21b6,color:#fff
```

---

## 4. Tool Dispatch & Permission System

```mermaid
flowchart TD
    TOOL_USE(["ToolUse block received<br/><i>{ id, name, input }</i>"])
    FIND{"Find tool<br/>by name"}
    NOT_FOUND["Return error:<br/>'Unknown tool'"]

    CHECK_ENABLED{"Tool<br/>enabled?"}
    DISABLED["Return error:<br/>'Tool disabled'"]

    PERM_LEVEL{"Permission<br/>level?"}

    NONE["PermissionLevel::None<br/><i>informational</i>"]
    READ["PermissionLevel::ReadOnly<br/><i>file reads · search</i>"]
    WRITE["PermissionLevel::Write<br/><i>file edit · create</i>"]
    EXEC["PermissionLevel::Execute<br/><i>bash · tasks · cron</i>"]

    MODE{"Permission<br/>mode?"}

    BYPASS["BypassPermissions<br/><i>allow all</i>"]
    PLAN["Plan mode<br/><i>reads only</i>"]
    DEFAULT["Default mode"]
    ACCEPT["AcceptEdits<br/><i>allow file ops</i>"]

    PLAN_CHECK{"Read-only<br/>tool?"}
    DENY_PLAN["Deny:<br/>'Plan mode - read only'"]

    ASK_USER["Show permission dialog<br/><i>tool name · description · risk</i>"]
    USER_DECISION{"User<br/>decision?"}
    ALLOW["Allow"]
    ALLOW_PERM["Allow permanently<br/><i>remember for session</i>"]
    DENY_USER["Deny"]

    RUN_HOOKS["Run PreToolUse hooks<br/><i>external commands</i>"]
    HOOK_RESULT{"Hook<br/>outcome?"}
    HOOK_BLOCK["Blocked by hook"]

    VALIDATE["Validate input<br/>against JSON Schema"]
    INVALID["Return error:<br/>'Invalid input'"]

    EXECUTE["tool.execute(input, ctx)"]
    RESULT(["ToolResult<br/><i>{ content, is_error, metadata }</i>"])

    TOOL_USE --> FIND
    FIND -->|"not found"| NOT_FOUND
    FIND -->|"found"| CHECK_ENABLED
    CHECK_ENABLED -->|"no"| DISABLED
    CHECK_ENABLED -->|"yes"| PERM_LEVEL

    PERM_LEVEL -->|"None"| NONE --> VALIDATE
    PERM_LEVEL -->|"ReadOnly"| READ --> MODE
    PERM_LEVEL -->|"Write"| WRITE --> MODE
    PERM_LEVEL -->|"Execute"| EXEC --> MODE

    MODE -->|"Bypass"| BYPASS --> VALIDATE
    MODE -->|"Plan"| PLAN --> PLAN_CHECK
    MODE -->|"AcceptEdits"| ACCEPT --> VALIDATE
    MODE -->|"Default"| DEFAULT --> ASK_USER

    PLAN_CHECK -->|"yes"| VALIDATE
    PLAN_CHECK -->|"no"| DENY_PLAN

    ASK_USER --> USER_DECISION
    USER_DECISION -->|"allow"| ALLOW --> VALIDATE
    USER_DECISION -->|"always"| ALLOW_PERM --> VALIDATE
    USER_DECISION -->|"deny"| DENY_USER

    VALIDATE -->|"invalid"| INVALID
    VALIDATE -->|"valid"| RUN_HOOKS

    RUN_HOOKS --> HOOK_RESULT
    HOOK_RESULT -->|"blocked"| HOOK_BLOCK
    HOOK_RESULT -->|"allowed"| EXECUTE --> RESULT

    style TOOL_USE fill:#ea580c,stroke:#c2410c,color:#fff
    style RESULT fill:#059669,stroke:#047857,color:#fff
    style NOT_FOUND fill:#dc2626,stroke:#991b1b,color:#fff
    style DISABLED fill:#dc2626,stroke:#991b1b,color:#fff
    style DENY_PLAN fill:#dc2626,stroke:#991b1b,color:#fff
    style DENY_USER fill:#dc2626,stroke:#991b1b,color:#fff
    style HOOK_BLOCK fill:#dc2626,stroke:#991b1b,color:#fff
    style INVALID fill:#dc2626,stroke:#991b1b,color:#fff
```

---

## 5. API Client — Streaming & Retry Logic

```mermaid
flowchart TD
    REQ(["create_message_stream()<br/><i>model · messages · tools · system</i>"])
    SET_STREAM["Set stream: true"]
    BUILD_HEADERS["Build headers<br/><i>auth · version · beta · content-type</i>"]

    AUTH{"Auth<br/>mode?"}
    BEARER["Authorization: Bearer {key}"]
    API_KEY["x-api-key: {key}"]

    SEND["POST /v1/messages<br/><i>reqwest with timeout 600s</i>"]

    STATUS{"HTTP<br/>status?"}
    S200["200 OK<br/><i>start SSE stream</i>"]
    S429["429 Rate Limited"]
    S529["529 Overloaded"]
    S401["401/403 Auth Error"]
    S_OTHER["Other Error"]

    RETRY_CHECK{"Retries<br/>remaining?"}
    RETRY_AFTER{"Retry-After<br/>header?"}
    USE_HEADER["Wait Retry-After<br/>seconds"]
    BACKOFF["Exponential backoff<br/><i>delay × 2, max 60s</i>"]
    RETRY_DEC["retries -= 1"]

    AUTH_ERR["ClaudeError::Auth"]
    API_ERR["ClaudeError::ApiStatus"]

    SPAWN["Spawn background<br/>reader task"]
    READ_SSE["Read SSE frame<br/><i>event: ... + data: ...</i>"]
    PARSE["Parse StreamEvent"]

    EVT{"Event<br/>type?"}
    MSG_START["MessageStart<br/><i>{ id, model, usage }</i>"]
    BLOCK_START["ContentBlockStart<br/><i>{ index, type }</i>"]
    DELTA["ContentBlockDelta<br/><i>text · json · thinking · sig</i>"]
    BLOCK_STOP["ContentBlockStop"]
    MSG_DELTA["MessageDelta<br/><i>{ stop_reason, usage }</i>"]
    MSG_STOP["MessageStop"]
    PING["Ping<br/><i>keep-alive</i>"]
    ERR_EVT["Error<br/><i>{ type, message }</i>"]

    CHANNEL["Send to mpsc channel"]
    ACCUMULATE["StreamAccumulator<br/><i>reconstruct full Message</i>"]
    DONE(["Return (Message,<br/>UsageInfo, stop_reason)"])

    REQ --> SET_STREAM --> BUILD_HEADERS
    BUILD_HEADERS --> AUTH
    AUTH -->|"bearer"| BEARER --> SEND
    AUTH -->|"api_key"| API_KEY --> SEND

    SEND --> STATUS
    STATUS -->|"200"| S200 --> SPAWN
    STATUS -->|"429"| S429 --> RETRY_CHECK
    STATUS -->|"529"| S529 --> RETRY_CHECK
    STATUS -->|"401/403"| S401 --> AUTH_ERR
    STATUS -->|"other"| S_OTHER --> API_ERR

    RETRY_CHECK -->|"yes"| RETRY_AFTER
    RETRY_CHECK -->|"no"| API_ERR
    RETRY_AFTER -->|"present"| USE_HEADER --> RETRY_DEC --> SEND
    RETRY_AFTER -->|"absent"| BACKOFF --> RETRY_DEC

    SPAWN --> READ_SSE --> PARSE --> EVT
    EVT -->|"message_start"| MSG_START --> CHANNEL
    EVT -->|"content_block_start"| BLOCK_START --> CHANNEL
    EVT -->|"content_block_delta"| DELTA --> CHANNEL
    EVT -->|"content_block_stop"| BLOCK_STOP --> CHANNEL
    EVT -->|"message_delta"| MSG_DELTA --> CHANNEL
    EVT -->|"message_stop"| MSG_STOP --> CHANNEL
    EVT -->|"ping"| PING --> READ_SSE
    EVT -->|"error"| ERR_EVT --> CHANNEL

    CHANNEL --> ACCUMULATE
    MSG_STOP --> DONE

    style REQ fill:#dc2626,stroke:#991b1b,color:#fff
    style DONE fill:#059669,stroke:#047857,color:#fff
    style AUTH_ERR fill:#dc2626,stroke:#991b1b,color:#fff
    style API_ERR fill:#dc2626,stroke:#991b1b,color:#fff
    style S200 fill:#059669,stroke:#047857,color:#fff
```

---

## 6. Auto-Compaction Flow

```mermaid
flowchart TD
    TRIGGER(["After each turn<br/>in query loop"])
    CHECK["should_auto_compact()"]
    ESTIMATE["Estimate token usage<br/><i>messages + tools + system</i>"]
    WINDOW["Get context window<br/><i>model-specific max tokens</i>"]
    RATIO{"usage / window<br/>> 90%?"}
    SKIP["Skip compaction<br/><i>context has room</i>"]

    DISABLED{"auto_compact<br/>enabled?"}
    SKIP2["Skip compaction<br/><i>disabled by config</i>"]

    START_COMPACT["Start compact_conversation()"]
    SUMMARY_PROMPT["Build summary prompt<br/><i>'Summarize the conversation so far...'</i>"]
    API_CALL["Call Claude API<br/><i>compact model · messages</i>"]
    RECEIVE["Receive summary"]

    REPLACE["Replace old messages<br/>with compact summary"]
    SYSTEM["Prepend system note:<br/><i>'[Context compacted]'</i>"]
    EMIT["Emit Status event:<br/><i>'Conversation compacted'</i>"]
    CONTINUE(["Continue<br/>query loop"])

    TRIGGER --> CHECK --> ESTIMATE --> WINDOW --> RATIO
    RATIO -->|"no"| SKIP
    RATIO -->|"yes"| DISABLED
    DISABLED -->|"no"| SKIP2
    DISABLED -->|"yes"| START_COMPACT

    START_COMPACT --> SUMMARY_PROMPT --> API_CALL --> RECEIVE
    RECEIVE --> REPLACE --> SYSTEM --> EMIT --> CONTINUE

    style TRIGGER fill:#7c3aed,stroke:#5b21b6,color:#fff
    style CONTINUE fill:#059669,stroke:#047857,color:#fff
    style SKIP fill:#6b7280,stroke:#4b5563,color:#fff
    style SKIP2 fill:#6b7280,stroke:#4b5563,color:#fff
```

---

## 7. MCP Client Protocol

```mermaid
sequenceDiagram
    participant CLI as Claude Code CLI
    participant MGR as McpManager
    participant CLIENT as McpClient
    participant TRANSPORT as StdioTransport
    participant SERVER as MCP Server<br/>(subprocess)

    Note over CLI,SERVER: Connection Phase
    CLI->>MGR: connect_all(configs)
    loop For each server config
        MGR->>CLIENT: connect_stdio(config)
        CLIENT->>TRANSPORT: spawn subprocess<br/>(command + args)
        TRANSPORT->>SERVER: Start process<br/>stdin/stdout pipes

        Note over CLIENT,SERVER: Initialization Handshake
        CLIENT->>TRANSPORT: send(initialize)
        TRANSPORT->>SERVER: {"jsonrpc":"2.0","method":"initialize",...}
        SERVER-->>TRANSPORT: {"jsonrpc":"2.0","result":{capabilities,...}}
        TRANSPORT-->>CLIENT: InitializeResult

        CLIENT->>TRANSPORT: send(notifications/initialized)
        TRANSPORT->>SERVER: {"jsonrpc":"2.0","method":"notifications/initialized"}
    end

    Note over CLI,SERVER: Tool Discovery
    CLI->>MGR: all_tool_definitions()
    MGR->>CLIENT: list_tools()
    CLIENT->>TRANSPORT: send(tools/list)
    TRANSPORT->>SERVER: {"jsonrpc":"2.0","method":"tools/list"}
    SERVER-->>TRANSPORT: {"result":{"tools":[...]}}
    TRANSPORT-->>CLIENT: Vec<McpTool>
    CLIENT-->>MGR: tools with schemas
    MGR-->>CLI: prefixed tool definitions<br/>(server_name::tool_name)

    Note over CLI,SERVER: Tool Execution
    CLI->>MGR: call_tool("server::tool", args)
    MGR->>CLIENT: call_tool("tool", args)
    CLIENT->>TRANSPORT: send(tools/call)
    TRANSPORT->>SERVER: {"jsonrpc":"2.0","method":"tools/call",...}
    SERVER-->>TRANSPORT: {"result":{"content":[...]}}
    TRANSPORT-->>CLIENT: CallToolResult
    CLIENT-->>MGR: result
    MGR-->>CLI: ToolResult

    Note over CLI,SERVER: Resource Access
    CLI->>MGR: read_resource(uri)
    MGR->>CLIENT: read_resource(uri)
    CLIENT->>TRANSPORT: send(resources/read)
    TRANSPORT->>SERVER: {"jsonrpc":"2.0","method":"resources/read",...}
    SERVER-->>TRANSPORT: {"result":{"contents":[...]}}
    TRANSPORT-->>CLIENT: ResourceContents
    CLIENT-->>MGR: resource data
    MGR-->>CLI: resource content
```

---

## 8. Bridge Remote Control Protocol

```mermaid
sequenceDiagram
    participant WEB as claude.ai<br/>Web UI
    participant API as Bridge API<br/>Server
    participant BRIDGE as BridgeSession<br/>(local CLI)
    participant QUERY as Query Loop

    Note over WEB,QUERY: Session Registration
    BRIDGE->>BRIDGE: device_fingerprint()<br/>SHA-256(host:user:home)
    BRIDGE->>API: POST /api/claude_code/sessions<br/>{session_id, device_id, client_version}
    API-->>BRIDGE: 200 OK {session_id}

    Note over WEB,QUERY: Poll Loop (background task)
    loop Every polling_interval_ms (1s)
        BRIDGE->>API: POST /events<br/>[queued BridgeEvents]
        API-->>BRIDGE: 200 OK

        BRIDGE->>API: GET /poll<br/>(long-polling, 35s timeout)

        alt User sends message
            WEB->>API: Send user message
            API-->>BRIDGE: BridgeMessage::UserMessage<br/>{content, session_id, attachments}
            BRIDGE->>QUERY: Inject as user turn
            QUERY-->>BRIDGE: Stream response events
            BRIDGE->>API: POST /events [TextDelta...]
            API-->>WEB: Display streaming response
        else Permission needed
            QUERY-->>BRIDGE: PermissionRequest<br/>{tool_name, description}
            BRIDGE->>API: POST /events [PermissionRequest]
            API-->>WEB: Show permission dialog
            WEB->>API: User decision
            API-->>BRIDGE: BridgeMessage::PermissionResponse<br/>{decision}
            BRIDGE->>QUERY: Apply decision
        else Cancel request
            WEB->>API: Cancel
            API-->>BRIDGE: BridgeMessage::Cancel
            BRIDGE->>QUERY: Cancel current turn
        else Keep-alive
            API-->>BRIDGE: BridgeMessage::Ping
            BRIDGE->>API: BridgeEvent::Pong
        else Timeout / Error
            API-->>BRIDGE: timeout (35s)
            BRIDGE->>BRIDGE: Exponential backoff<br/>then retry
        end
    end

    Note over WEB,QUERY: Session Teardown
    BRIDGE->>API: DELETE /api/claude_code/sessions/{id}
    API-->>BRIDGE: 200 OK
```

---

## 9. TUI Rendering Pipeline

```mermaid
flowchart TD
    LOOP(["Main event loop<br/><i>16ms tick</i>"])
    POLL["Poll crossterm events<br/><i>keyboard · mouse · resize</i>"]
    HAS_EVENT{"Event<br/>received?"}

    KEY_EVENT["Handle key event"]
    KEY_TYPE{"Key?"}
    CTRL_C["Ctrl+C<br/><i>cancel stream or quit</i>"]
    CTRL_D["Ctrl+D<br/><i>quit on empty input</i>"]
    CTRL_R["Ctrl+R<br/><i>history search mode</i>"]
    ENTER["Enter<br/><i>submit input</i>"]
    UP_DOWN["Up/Down<br/><i>history navigation</i>"]
    TEXT["Character input<br/><i>UTF-8 cursor handling</i>"]
    PG_UP_DOWN["PageUp/PageDown<br/><i>scroll messages</i>"]

    DRAIN["Drain query events<br/><i>from mpsc channel</i>"]
    Q_EVENT{"Event<br/>type?"}
    Q_STREAM["Stream delta<br/><i>append to streaming_text</i>"]
    Q_TOOL_START["ToolStart<br/><i>add spinner</i>"]
    Q_TOOL_END["ToolEnd<br/><i>update result · ✓/✗</i>"]
    Q_COMPLETE["TurnComplete<br/><i>finalize message</i>"]
    Q_STATUS["Status<br/><i>update status bar</i>"]
    Q_ERROR["Error<br/><i>show error message</i>"]

    RENDER["render_app(frame)"]
    LAYOUT["Split into 3 zones<br/><i>Constraint layout</i>"]

    MSGS_PANE["Messages Pane<br/><i>Min(3) height</i>"]
    RENDER_MSGS["Render messages"]
    ROLE_HDR["Role headers<br/><i>Human · Assistant</i>"]
    MD_RENDER["Markdown rendering<br/><i>headings · code · bold · inline</i>"]
    TOOL_BLOCKS["Tool use blocks<br/><i>spinner · name · output</i>"]
    STREAM_RENDER["Streaming text<br/><i>with cursor spinner</i>"]

    INPUT_PANE["Input Pane<br/><i>Length(5)</i>"]
    PROMPT["> prompt text<br/><i>cursor · slash hint</i>"]

    STATUS_BAR["Status Bar<br/><i>Length(1)</i>"]
    LEFT_STATUS["Left: status message"]
    RIGHT_STATUS["Right: model │ cost │ tokens"]

    OVERLAYS{"Active<br/>overlay?"}
    PERM_DIALOG["Permission dialog<br/><i>centered · y/a/n</i>"]
    HELP_OVERLAY["Help overlay<br/><i>F1 toggle</i>"]
    HISTORY_SEARCH["History search<br/><i>Ctrl+R overlay</i>"]

    FLUSH["Flush to terminal<br/><i>crossterm write</i>"]

    LOOP --> POLL --> HAS_EVENT
    HAS_EVENT -->|"yes"| KEY_EVENT
    HAS_EVENT -->|"no"| DRAIN

    KEY_EVENT --> KEY_TYPE
    KEY_TYPE -->|"Ctrl+C"| CTRL_C --> DRAIN
    KEY_TYPE -->|"Ctrl+D"| CTRL_D --> DRAIN
    KEY_TYPE -->|"Ctrl+R"| CTRL_R --> DRAIN
    KEY_TYPE -->|"Enter"| ENTER --> DRAIN
    KEY_TYPE -->|"↑/↓"| UP_DOWN --> DRAIN
    KEY_TYPE -->|"char"| TEXT --> DRAIN
    KEY_TYPE -->|"PgUp/PgDn"| PG_UP_DOWN --> DRAIN

    DRAIN --> Q_EVENT
    Q_EVENT -->|"Stream"| Q_STREAM --> RENDER
    Q_EVENT -->|"ToolStart"| Q_TOOL_START --> RENDER
    Q_EVENT -->|"ToolEnd"| Q_TOOL_END --> RENDER
    Q_EVENT -->|"TurnComplete"| Q_COMPLETE --> RENDER
    Q_EVENT -->|"Status"| Q_STATUS --> RENDER
    Q_EVENT -->|"Error"| Q_ERROR --> RENDER
    Q_EVENT -->|"none"| RENDER

    RENDER --> LAYOUT
    LAYOUT --> MSGS_PANE
    LAYOUT --> INPUT_PANE
    LAYOUT --> STATUS_BAR

    MSGS_PANE --> RENDER_MSGS
    RENDER_MSGS --> ROLE_HDR
    RENDER_MSGS --> MD_RENDER
    RENDER_MSGS --> TOOL_BLOCKS
    RENDER_MSGS --> STREAM_RENDER

    INPUT_PANE --> PROMPT
    STATUS_BAR --> LEFT_STATUS
    STATUS_BAR --> RIGHT_STATUS

    RENDER --> OVERLAYS
    OVERLAYS -->|"permission"| PERM_DIALOG --> FLUSH
    OVERLAYS -->|"help"| HELP_OVERLAY --> FLUSH
    OVERLAYS -->|"search"| HISTORY_SEARCH --> FLUSH
    OVERLAYS -->|"none"| FLUSH

    FLUSH --> LOOP

    style LOOP fill:#0891b2,stroke:#0e7490,color:#fff
    style RENDER fill:#2563eb,stroke:#1e40af,color:#fff
    style FLUSH fill:#059669,stroke:#047857,color:#fff
```

---

## 10. OAuth Authentication Flow

```mermaid
sequenceDiagram
    participant USER as User
    participant CLI as Claude Code CLI
    participant BROWSER as Web Browser
    participant AUTH as platform.claude.com<br/>(OAuth Server)

    Note over USER,AUTH: PKCE Authorization Code Flow

    USER->>CLI: /login or needs auth
    CLI->>CLI: generate_code_verifier()<br/>32 random bytes → base64url (43 chars)
    CLI->>CLI: generate_code_challenge(verifier)<br/>SHA-256 → base64url
    CLI->>CLI: generate_state()<br/>random CSRF token

    CLI->>CLI: build_auth_url()<br/>client_id + redirect + scopes + PKCE
    Note right of CLI: Scopes:<br/>org:create_api_key<br/>user:inference<br/>user:profile<br/>user:sessions:claude_code<br/>user:mcp_servers<br/>user:file_upload

    CLI->>BROWSER: Open authorization URL
    BROWSER->>AUTH: GET /oauth/authorize?...
    AUTH->>USER: Show login/consent page
    USER->>AUTH: Approve access
    AUTH->>BROWSER: Redirect to localhost callback<br/>?code=AUTH_CODE&state=STATE

    CLI->>CLI: Receive callback<br/>Verify state matches

    CLI->>AUTH: POST /v1/oauth/token<br/>{grant_type: authorization_code,<br/>code, redirect_uri, code_verifier}
    AUTH-->>CLI: {access_token, refresh_token,<br/>expires_in, scope}

    CLI->>CLI: Save to ~/.claude/oauth_tokens.json<br/>{access_token, refresh_token,<br/>expires_at_ms, scopes, email}

    Note over USER,AUTH: Silent Token Refresh (on startup)

    CLI->>CLI: Load saved tokens
    CLI->>CLI: Check is_expired()

    alt Token expired
        CLI->>AUTH: POST /v1/oauth/token<br/>{grant_type: refresh_token,<br/>refresh_token}
        AUTH-->>CLI: {new access_token,<br/>new refresh_token, expires_in}
        CLI->>CLI: Save updated tokens
    else Token valid
        CLI->>CLI: Use existing access_token
    end

    CLI->>CLI: Build client with<br/>Authorization: Bearer {token}
```

---

## 11. Buddy Companion Generation

```mermaid
flowchart TD
    INPUT(["user_id<br/><i>string</i>"])
    HASH["SHA-256(user_id + 'friend-2026-401')"]
    SEED["Extract first 4 bytes<br/>as u32 little-endian"]
    PRNG["Initialize Mulberry32<br/>PRNG(seed)"]

    ROLL_RARITY["Roll rarity<br/><i>next_f64()</i>"]
    RARITY{"Random<br/>value?"}
    COMMON["Common<br/><i>60% · floor=5</i>"]
    UNCOMMON["Uncommon<br/><i>25% · floor=15</i>"]
    RARE["Rare<br/><i>10% · floor=25</i>"]
    EPIC["Epic<br/><i>4% · floor=35</i>"]
    LEGENDARY["Legendary<br/><i>1% · floor=50</i>"]

    ROLL_SPECIES["Roll species<br/><i>18 options</i>"]
    SPECIES["Species assigned<br/><i>Duck · Goose · Blob · Cat ·<br/>Dragon · Octopus · Owl · Penguin ·<br/>Turtle · Snail · Ghost · Axolotl ·<br/>Capybara · Cactus · Robot ·<br/>Rabbit · Mushroom · Chonk</i>"]

    ROLL_EYE["Roll eye style<br/><i>6 options</i>"]
    EYES["Eye: · ✦ × ◉ @ °"]

    ROLL_HAT["Roll hat<br/><i>8 options</i>"]
    HAT_CHECK{"Rarity =<br/>Common?"}
    NO_HAT["Hat: None"]
    HAT["Hat: Crown · Tophat ·<br/>Propeller · Halo ·<br/>Wizard · Beanie · TinyDuck"]

    ROLL_SHINY["Roll shiny<br/><i>1% chance</i>"]
    SHINY{"Shiny?"}
    YES_SHINY["✨ Shiny variant"]
    NO_SHINY["Normal variant"]

    ROLL_STATS["Roll 5 stats"]
    STATS["Stats (floor-based):<br/><i>DEBUGGING · PATIENCE ·<br/>CHAOS · WISDOM · SNARK</i><br/><br/>Peak: floor + 50-79<br/>Dump: floor − 10-15<br/>Others: floor + 0-40"]

    BONES(["CompanionBones<br/><i>immutable · deterministic</i>"])

    SOUL_CHECK{"Soul exists in<br/>companion.json?"}
    LOAD_SOUL["Load CompanionSoul<br/><i>name · personality · hatched_at</i>"]
    GEN_SOUL["Generate soul via Claude<br/><i>AI-written name + personality</i>"]
    SAVE_SOUL["Save to<br/>~/.claude/companion.json"]

    COMPANION(["Complete Companion<br/><i>Bones + Soul</i>"])

    INPUT --> HASH --> SEED --> PRNG

    PRNG --> ROLL_RARITY
    ROLL_RARITY --> RARITY
    RARITY -->|"0.00 - 0.60"| COMMON
    RARITY -->|"0.60 - 0.85"| UNCOMMON
    RARITY -->|"0.85 - 0.95"| RARE
    RARITY -->|"0.95 - 0.99"| EPIC
    RARITY -->|"0.99 - 1.00"| LEGENDARY

    COMMON --> ROLL_SPECIES
    UNCOMMON --> ROLL_SPECIES
    RARE --> ROLL_SPECIES
    EPIC --> ROLL_SPECIES
    LEGENDARY --> ROLL_SPECIES

    ROLL_SPECIES --> SPECIES --> ROLL_EYE --> EYES --> ROLL_HAT
    ROLL_HAT --> HAT_CHECK
    HAT_CHECK -->|"yes"| NO_HAT --> ROLL_SHINY
    HAT_CHECK -->|"no"| HAT --> ROLL_SHINY

    ROLL_SHINY --> SHINY
    SHINY -->|"yes (1%)"| YES_SHINY --> ROLL_STATS
    SHINY -->|"no (99%)"| NO_SHINY --> ROLL_STATS

    ROLL_STATS --> STATS --> BONES

    BONES --> SOUL_CHECK
    SOUL_CHECK -->|"yes"| LOAD_SOUL --> COMPANION
    SOUL_CHECK -->|"no"| GEN_SOUL --> SAVE_SOUL --> COMPANION

    style INPUT fill:#f59e0b,stroke:#d97706,color:#000
    style COMPANION fill:#059669,stroke:#047857,color:#fff
    style BONES fill:#2563eb,stroke:#1e40af,color:#fff
    style YES_SHINY fill:#f59e0b,stroke:#d97706,color:#000
    style LEGENDARY fill:#7c3aed,stroke:#5b21b6,color:#fff
    style EPIC fill:#c026d3,stroke:#a21caf,color:#fff
    style RARE fill:#2563eb,stroke:#1e40af,color:#fff
    style UNCOMMON fill:#059669,stroke:#047857,color:#fff
    style COMMON fill:#6b7280,stroke:#4b5563,color:#fff
```

---

## 12. Slash Command Dispatch

```mermaid
flowchart TD
    INPUT(["User input<br/><i>string</i>"])
    SLASH{"Starts with<br/>'/'?"}
    PROMPT["Send as user<br/>message to query loop"]

    PARSE["Parse command name<br/>& arguments"]
    LOOKUP["Look up command<br/><i>name + aliases</i>"]
    FOUND{"Command<br/>found?"}
    FUZZY["Fuzzy match<br/>suggestion"]
    NOT_FOUND["Error:<br/>'Unknown command'<br/><i>Did you mean /xxx?</i>"]

    HIDDEN{"Command<br/>hidden?"}
    EXEC["command.execute(args, ctx)"]

    RESULT{"CommandResult?"}
    MSG["Message(text)<br/><i>display to user</i>"]
    USER_MSG["UserMessage(text)<br/><i>inject as user turn</i>"]
    CONFIG_CHG["ConfigChange(cfg)<br/><i>update runtime config</i>"]
    CLEAR_CONV["ClearConversation<br/><i>reset all messages</i>"]
    SET_MSGS["SetMessages(msgs)<br/><i>rewind to state</i>"]
    OAUTH["StartOAuthFlow<br/><i>trigger login</i>"]
    EXIT_CMD["Exit<br/><i>quit REPL</i>"]
    SILENT["Silent<br/><i>no output</i>"]
    ERROR["Error(msg)<br/><i>show error</i>"]

    INPUT --> SLASH
    SLASH -->|"no"| PROMPT
    SLASH -->|"yes"| PARSE --> LOOKUP

    LOOKUP --> FOUND
    FOUND -->|"no"| FUZZY --> NOT_FOUND
    FOUND -->|"yes"| HIDDEN
    HIDDEN -->|"visible or hidden"| EXEC

    EXEC --> RESULT
    RESULT -->|"Message"| MSG
    RESULT -->|"UserMessage"| USER_MSG
    RESULT -->|"ConfigChange"| CONFIG_CHG
    RESULT -->|"ClearConversation"| CLEAR_CONV
    RESULT -->|"SetMessages"| SET_MSGS
    RESULT -->|"StartOAuthFlow"| OAUTH
    RESULT -->|"Exit"| EXIT_CMD
    RESULT -->|"Silent"| SILENT
    RESULT -->|"Error"| ERROR

    style INPUT fill:#4f46e5,stroke:#3730a3,color:#fff
    style PROMPT fill:#059669,stroke:#047857,color:#fff
    style EXIT_CMD fill:#dc2626,stroke:#991b1b,color:#fff
    style NOT_FOUND fill:#dc2626,stroke:#991b1b,color:#fff
    style ERROR fill:#dc2626,stroke:#991b1b,color:#fff
```

---

## 13. Memory & Context Assembly

```mermaid
flowchart TD
    START(["Build system prompt"])
    CTX_BUILDER["ContextBuilder::new()"]

    SYS_CTX["System Context"]
    PLATFORM["Platform info<br/><i>OS · arch · shell</i>"]
    WORK_DIR["Working directory"]
    GIT_STATUS["git status --short"]
    GIT_LOG["git log --oneline -10"]

    USER_CTX["User Context"]
    DATE["Today's date<br/><i>ISO 8601</i>"]
    GLOBAL_MD["Global CLAUDE.md<br/><i>~/.claude/CLAUDE.md</i>"]
    PROJECT_MD["Project CLAUDE.md<br/><i>walk up from cwd</i>"]
    LOCAL_MD["Local CLAUDE.md<br/><i>.claude/CLAUDE.md</i>"]

    BASE_PROMPT["Base system prompt<br/><i>identity · capabilities · rules</i>"]
    CUSTOM{"Custom system<br/>prompt?"}
    OVERRIDE["Replace with<br/>custom prompt"]
    APPEND{"Append system<br/>prompt?"}
    APPEND_TEXT["Append additional<br/>instructions"]

    TOOL_DEFS["Tool definitions<br/><i>32+ tool schemas</i>"]
    MCP_TOOLS["MCP tool definitions<br/><i>from connected servers</i>"]
    MERGE_TOOLS["Merge tool list<br/><i>native + MCP</i>"]

    HISTORY["Conversation history<br/><i>previous messages</i>"]
    COMPACT_CHK{"Previously<br/>compacted?"}
    COMPACT_NOTE["Prepend:<br/>'[Context compacted]'"]

    FINAL(["Final API Request<br/><i>system + messages + tools</i>"])

    START --> CTX_BUILDER

    CTX_BUILDER --> SYS_CTX
    SYS_CTX --> PLATFORM
    SYS_CTX --> WORK_DIR
    SYS_CTX --> GIT_STATUS
    SYS_CTX --> GIT_LOG

    CTX_BUILDER --> USER_CTX
    USER_CTX --> DATE
    USER_CTX --> GLOBAL_MD
    USER_CTX --> PROJECT_MD
    USER_CTX --> LOCAL_MD

    SYS_CTX --> BASE_PROMPT
    USER_CTX --> BASE_PROMPT

    BASE_PROMPT --> CUSTOM
    CUSTOM -->|"yes"| OVERRIDE --> TOOL_DEFS
    CUSTOM -->|"no"| APPEND
    APPEND -->|"yes"| APPEND_TEXT --> TOOL_DEFS
    APPEND -->|"no"| TOOL_DEFS

    TOOL_DEFS --> MCP_TOOLS --> MERGE_TOOLS

    MERGE_TOOLS --> HISTORY
    HISTORY --> COMPACT_CHK
    COMPACT_CHK -->|"yes"| COMPACT_NOTE --> FINAL
    COMPACT_CHK -->|"no"| FINAL

    style START fill:#7c3aed,stroke:#5b21b6,color:#fff
    style FINAL fill:#059669,stroke:#047857,color:#fff
    style GLOBAL_MD fill:#f59e0b,stroke:#d97706,color:#000
    style PROJECT_MD fill:#f59e0b,stroke:#d97706,color:#000
    style LOCAL_MD fill:#f59e0b,stroke:#d97706,color:#000
```

---

## 14. Session Persistence

```mermaid
flowchart TD
    START(["Session event"])

    CREATE["Create new session<br/><i>ConversationSession::new()</i>"]
    UUID["Generate UUID v4<br/><i>session ID</i>"]
    INIT["Initialize:<br/><i>messages: [] · model ·<br/>created_at · working_dir</i>"]

    TURN_END["Turn completes"]
    ADD_MSG["Append messages<br/><i>user + assistant + tool results</i>"]
    UPDATE_TS["Update updated_at<br/>timestamp"]

    SAVE["save_session()"]
    PATH["~/.claude/conversations/<br/>{session_id}.json"]
    SERIALIZE["serde_json::to_string_pretty()"]
    WRITE["Write to file<br/><i>atomic via tempfile</i>"]

    RESUME_CMD["/resume command"]
    LIST["list_sessions()"]
    READ_DIR["Read ~/.claude/conversations/"]
    SORT["Sort by updated_at<br/><i>most recent first</i>"]
    DISPLAY["Display session list<br/><i>title · date · message count</i>"]

    SELECT["User selects session"]
    LOAD["load_session(id)"]
    DESERIALIZE["Parse JSON file"]
    RESTORE["Restore messages<br/>& state to App"]

    REWIND["/rewind command"]
    REMOVE["Remove last N<br/>message pairs"]
    SET_MSGS["SetMessages(truncated)"]

    EXPORT["/export command"]
    FORMAT["Format conversation<br/><i>markdown · JSON</i>"]
    OUTPUT["Write to file<br/>or stdout"]

    DELETE["Session cleanup"]
    DELETE_FILE["delete_session(id)<br/><i>remove JSON file</i>"]

    START -->|"new session"| CREATE --> UUID --> INIT
    START -->|"turn end"| TURN_END --> ADD_MSG --> UPDATE_TS --> SAVE
    SAVE --> PATH --> SERIALIZE --> WRITE

    START -->|"/resume"| RESUME_CMD --> LIST --> READ_DIR --> SORT --> DISPLAY
    DISPLAY --> SELECT --> LOAD --> DESERIALIZE --> RESTORE

    START -->|"/rewind"| REWIND --> REMOVE --> SET_MSGS --> SAVE

    START -->|"/export"| EXPORT --> FORMAT --> OUTPUT

    START -->|"delete"| DELETE --> DELETE_FILE

    style START fill:#2563eb,stroke:#1e40af,color:#fff
    style WRITE fill:#059669,stroke:#047857,color:#fff
    style RESTORE fill:#059669,stroke:#047857,color:#fff
    style OUTPUT fill:#059669,stroke:#047857,color:#fff
    style DELETE_FILE fill:#dc2626,stroke:#991b1b,color:#fff
```

---

## Quick Reference — All Tool Names

```mermaid
graph LR
    subgraph Shell
        BASH["BashTool"]
        PS["PowerShellTool"]
    end

    subgraph Files
        READ["FileReadTool"]
        EDIT["FileEditTool"]
        WRITE["FileWriteTool"]
        NB["NotebookEditTool"]
    end

    subgraph Search
        GLOB["GlobTool"]
        GREP["GrepTool"]
        TS["ToolSearchTool"]
    end

    subgraph Web
        FETCH["WebFetchTool"]
        SEARCH["WebSearchTool"]
    end

    subgraph Tasks
        TC["TaskCreateTool"]
        TG["TaskGetTool"]
        TU["TaskUpdateTool"]
        TL["TaskListTool"]
        TSTOP["TaskStopTool"]
        TOUT["TaskOutputTool"]
        TODO["TodoWriteTool"]
    end

    subgraph Agents
        SM["SendMessageTool"]
        ASK["AskUserQuestionTool"]
        BRIEF["BriefTool"]
    end

    subgraph Planning
        EPM["EnterPlanModeTool"]
        XPM["ExitPlanModeTool"]
    end

    subgraph MCP
        LMR["ListMcpResourcesTool"]
        RMR["ReadMcpResourceTool"]
    end

    subgraph Scheduling
        CC["CronCreateTool"]
        CD["CronDeleteTool"]
        CL["CronListTool"]
    end

    subgraph Worktree
        EW["EnterWorktreeTool"]
        XW["ExitWorktreeTool"]
    end

    subgraph Other
        SLEEP["SleepTool"]
        CONFIG["ConfigTool"]
        SKILL["SkillTool"]
    end

    style Shell fill:#dc2626,stroke:#991b1b,color:#fff
    style Files fill:#ea580c,stroke:#c2410c,color:#fff
    style Search fill:#f59e0b,stroke:#d97706,color:#000
    style Web fill:#059669,stroke:#047857,color:#fff
    style Tasks fill:#0891b2,stroke:#0e7490,color:#fff
    style Agents fill:#2563eb,stroke:#1e40af,color:#fff
    style Planning fill:#4f46e5,stroke:#3730a3,color:#fff
    style MCP fill:#7c3aed,stroke:#5b21b6,color:#fff
    style Scheduling fill:#c026d3,stroke:#a21caf,color:#fff
    style Worktree fill:#e11d48,stroke:#be123c,color:#fff
    style Other fill:#6b7280,stroke:#4b5563,color:#fff
```
