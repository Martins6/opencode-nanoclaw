# NanoClaw Architecture Explained

## Overview

NanoClaw is a personal Claude assistant that connects to WhatsApp (and other channels), routing messages to isolated AI agent containers. Each group gets its own container with isolated filesystem and memory.

## High-Level Architecture

```mermaid
graph TB
    subgraph External["External Systems"]
        WA[WhatsApp API]
    end

    subgraph NanoClaw["NanoClaw Core"]
        IDX[index.ts<br/>Orchestrator]
        DB[(SQLite<br/>messages.db)]
        GQ[GroupQueue<br/>Concurrency Control]
        IPC[IPC Watcher<br/>Inter-Process Communication]
        TS[Task Scheduler<br/>Cron Jobs]
    end

    subgraph Channels["Channel Layer"]
        WAC[WhatsAppChannel<br/>Baileys SDK]
    end

    subgraph Containers["Agent Containers"]
        C1[Container<br/>Group A]
        C2[Container<br/>Group B]
        C3[Container<br/>Main Group]
    end

    WA <--> WAC
    WAC --> IDX
    IDX --> DB
    IDX --> GQ
    IDX --> IPC
    IDX --> TS
    GQ --> C1
    GQ --> C2
    GQ --> C3
    IPC -.->|IPC Files| C1
    IPC -.->|IPC Files| C2
    IPC -.->|IPC Files| C3
```

## Message Flow

### 1. Incoming Message Flow

```mermaid
sequenceDiagram
    participant WA as WhatsApp API
    participant WAC as WhatsAppChannel
    participant IDX as index.ts
    participant DB as SQLite DB
    participant GQ as GroupQueue
    participant CTR as Container Runner
    participant AG as Agent Container

    WA->>WAC: New message received
    WAC->>WAC: Parse Baileys message
    WAC->>IDX: onMessage(chatJid, msg)

    alt Registered Group
        IDX->>DB: storeMessage(msg)
        IDX->>IDX: Check trigger pattern

        alt Has Trigger
            IDX->>GQ: enqueueMessageCheck(jid)
            GQ->>CTR: runContainerAgent()
            CTR->>AG: Spawn container
            AG->>AG: Process with Claude
            AG-->>CTR: Stream output
            CTR-->>IDX: onOutput(result)
            IDX->>WAC: sendMessage(jid, text)
            WAC->>WA: Send response
        end
    end
```

### 2. Container Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Idle : Message arrives
    Idle --> Active : GroupQueue spawns container
    Active --> Processing : Agent receives prompt
    Processing --> Streaming : Agent generates output
    Streaming --> Idle : Output sent, waiting for more
    Idle --> Processing : New message piped via IPC
    Idle --> Closed : Idle timeout / Task complete
    Active --> Error : Container crash
    Error --> Retry : Exponential backoff
    Retry --> Active : Retry attempt
    Closed --> [*] : Container removed (--rm)
```

## Core Components

### 1. Orchestrator (src/index.ts)

```mermaid
flowchart TD
    subgraph Orchestrator
        A[main] --> B[ensureContainerSystemRunning]
        B --> C[initDatabase]
        C --> D[loadState]
        D --> E[Connect Channels]
        E --> F[startSchedulerLoop]
        E --> G[startIpcWatcher]
        E --> H[GroupQueue.setProcessMessagesFn]
        E --> I[recoverPendingMessages]
        I --> J[startMessageLoop]

        J --> K{New Messages?}
        K -->|Yes| L[getNewMessages]
        L --> M[Advance cursor]
        M --> N[Format messages]
        N --> O{Has active container?}
        O -->|Yes| P[Pipe to container]
        O -->|No| Q[Enqueue for new container]
        Q --> R[GroupQueue.enqueueMessageCheck]
        K -->|No| J
    end
```

### 2. WhatsApp Channel (src/channels/whatsapp.ts)

```mermaid
flowchart LR
    subgraph BaileysEvents["Baileys Events"]
        A[connection.update]
        B[messages.upsert]
        C[creds.update]
    end

    subgraph ConnectionStates["Connection States"]
        D{QR Code?}
        E{Close?}
        F{Open?}
    end

    subgraph MessageProcessing["Message Processing"]
        G[Translate LID JID]
        H[Check registered]
        I[Extract content]
        J[Detect bot msg]
        K[Call onMessage]
    end

    A --> D
    D -->|Yes| L[Exit - needs auth]
    E -->|Logout| M[Exit]
    E -->|Other| N[Reconnect]
    F --> O[Set available]
    F --> P[Flush queue]
    F --> Q[Sync groups]

    B --> G --> H
    H -->|Registered| I --> J --> K
    H -->|Not registered| R[Store metadata only]
```

### 3. Group Queue (src/group-queue.ts)

```mermaid
flowchart TD
    subgraph QueueManagement["Group Queue Management"]
        A[enqueueMessageCheck] --> B{Active?}
        B -->|Yes| C[Set pending flag]
        B -->|No| D{At concurrency limit?}
        D -->|Yes| E[Add to waiting list]
        D -->|No| F[runForGroup]

        F --> G[Mark active]
        G --> H[processMessagesFn]
        H --> I{Success?}
        I -->|Yes| J[Clear retry count]
        I -->|No| K[scheduleRetry]
        K --> L[Exponential backoff]

        H --> M[Mark inactive]
        M --> N[drainGroup]

        N --> O{Pending tasks?}
        O -->|Yes| P[Run task]
        O -->|No| Q{Pending messages?}
        Q -->|Yes| R[runForGroup drain]
        Q -->|No| S[drainWaiting]

        S --> T{Waiting groups?}
        T -->|Yes| U{Has capacity?}
        U -->|Yes| V[Dequeue & run]
    end
```

### 4. Container Runner (src/container-runner.ts)

```mermaid
flowchart TD
    subgraph ContainerRunner["Container Runner"]
        A[runContainerAgent] --> B[buildVolumeMounts]
        B --> C[buildContainerArgs]

        subgraph VolumeMounts["Volume Mounts"]
            D[/workspace/project<br/>Main: readonly/]
            E[/workspace/group<br/>Writable/]
            F[/home/node/.claude<br/>Sessions/]
            G[/workspace/ipc<br/>IPC namespace/]
            H[/app/src<br/>Agent runner source/]
        end

        C --> I[spawn container]
        I --> J[Write secrets to stdin]
        J --> K[Parse stdout markers]

        K --> L[OUTPUT_START_MARKER]
        L --> M[Parse JSON]
        M --> N[Call onOutput]
        N --> O[OUTPUT_END_MARKER]

        K --> P{Timeout?}
        P -->|Yes| Q[Stop container]
        P -->|No| R[Container closes]

        R --> S{Exit code?}
        S -->|0| T[Parse final output]
        S -->|Error| U[Log error]

        T --> V[Write container log]
    end
```

### 5. Database Layer (src/db.ts)

```mermaid
erDiagram
    CHATS {
        string jid PK
        string name
        string last_message_time
        string channel
        int is_group
    }

    MESSAGES {
        string id PK
        string chat_jid FK
        string sender
        string sender_name
        text content
        string timestamp
        int is_from_me
        int is_bot_message
    }

    SCHEDULED_TASKS {
        string id PK
        string group_folder
        string chat_jid
        text prompt
        string schedule_type
        string schedule_value
        string context_mode
        string next_run
        string last_run
        string last_result
        string status
        string created_at
    }

    TASK_RUN_LOGS {
        int id PK
        string task_id FK
        string run_at
        int duration_ms
        string status
        text result
        text error
    }

    ROUTER_STATE {
        string key PK
        string value
    }

    SESSIONS {
        string group_folder PK
        string session_id
    }

    REGISTERED_GROUPS {
        string jid PK
        string name
        string folder
        string trigger_pattern
        string added_at
        text container_config
        int requires_trigger
    }

    CHATS ||--o{ MESSAGES : contains
    SCHEDULED_TASKS ||--o{ TASK_RUN_LOGS : logs
```

### 6. IPC System (src/ipc.ts)

```mermaid
flowchart TD
    subgraph IPCWatcher["IPC Watcher"]
        A[startIpcWatcher] --> B[Scan IPC directories]
        B --> C[For each group folder]

        C --> D[Process messages/]
        D --> E{Authorized?}
        E -->|Main or own group| F[sendMessage]
        E -->|Other| G[Block - log warning]

        C --> H[Process tasks/]
        H --> I{Task type?}

        I -->|schedule_task| J[Create task]
        I -->|pause_task| K[Pause task]
        I -->|resume_task| L[Resume task]
        I -->|cancel_task| M[Delete task]
        I -->|refresh_groups| N[Sync metadata]
        I -->|register_group| O[Register group]

        J --> P{Authorized?}
        P -->|Main or own| Q[createTask]
        P -->|Other| R[Block]

        N --> S{Is main?}
        S -->|Yes| T[syncGroupMetadata]
        S -->|No| U[Block]

        O --> V{Is main?}
        V -->|Yes| W[registerGroup]
        V -->|No| X[Block]
    end
```

### 7. Task Scheduler (src/task-scheduler.ts)

```mermaid
flowchart TD
    subgraph Scheduler["Task Scheduler"]
        A[startSchedulerLoop] --> B[Loop]
        B --> C[getDueTasks]
        C --> D{Due tasks?}
        D -->|Yes| E[For each task]
        E --> F[Re-check status]
        F --> G{Still active?}
        G -->|Yes| H[queue.enqueueTask]
        H --> I[runTask]

        I --> J[writeTasksSnapshot]
        J --> K[runContainerAgent]
        K --> L[Stream output]
        L --> M[sendMessage]
        L --> N[scheduleClose]

        I --> O[logTaskRun]
        O --> P[Calculate next_run]
        P --> Q[updateTaskAfterRun]

        D -->|No| R[Sleep interval]
        R --> B
    end
```

## Security & Isolation

### Container Isolation Model

```mermaid
graph TB
    subgraph Host["Host Machine"]
        NC[NanoClaw Core]
        DB[(SQLite DB)]

        subgraph DataDir["data/"]
            IPC[ipc/]
            SESSIONS[sessions/]
        end
    end

    subgraph GroupA["Group A Container"]
        GA_WORKSPACE[/workspace/group<br/>Writable/]
        GA_CLAUDE[/home/node/.claude<br/>Isolated/]
        GA_IPC[/workspace/ipc<br/>Namespace A/]
        GA_AGENT[Agent Runner]
    end

    subgraph GroupB["Group B Container"]
        GB_WORKSPACE[/workspace/group<br/>Writable/]
        GB_CLAUDE[/home/node/.claude<br/>Isolated/]
        GB_IPC[/workspace/ipc<br/>Namespace B/]
        GB_AGENT[Agent Runner]
    end

    subgraph MainGroup["Main Group Container"]
        M_WORKSPACE[/workspace/group<br/>Writable/]
        M_PROJECT[/workspace/project<br/>Read-only/]
        M_CLAUDE[/home/node/.claude<br/>Isolated/]
        M_IPC[/workspace/ipc<br/>Main namespace/]
        M_AGENT[Agent Runner]
    end

    NC --> DB
    NC -->|Spawn| GroupA
    NC -->|Spawn| GroupB
    NC -->|Spawn| MainGroup

    IPC -.->|messages/| GA_IPC
    IPC -.->|messages/| GB_IPC
    IPC -.->|messages/| M_IPC

    SESSIONS -.->|Group A| GA_CLAUDE
    SESSIONS -.->|Group B| GB_CLAUDE
    SESSIONS -.->|Main| M_CLAUDE
```

### Authorization Matrix

| Action                    | Main Group        | Non-Main Group     |
| ------------------------- | ----------------- | ------------------ |
| Register new groups       | ✅ Yes            | ❌ No              |
| Schedule tasks            | ✅ All groups     | ⚠️ Own group only  |
| Pause/Resume/Cancel tasks | ✅ All            | ⚠️ Own only        |
| Send messages             | ✅ All registered | ⚠️ Own group only  |
| Refresh group metadata    | ✅ Yes            | ❌ No              |
| See all available groups  | ✅ Yes            | ❌ No (empty list) |

## State Management

### Persistence Flow

```mermaid
flowchart LR
    subgraph State["In-Memory State"]
        A[lastTimestamp]
        B[lastAgentTimestamp]
        C[sessions]
        D[registeredGroups]
    end

    subgraph Database["SQLite Database"]
        E[router_state table]
        F[sessions table]
        G[registered_groups table]
        H[messages table]
    end

    subgraph Filesystem["Filesystem"]
        I[groups/{folder}/]
        J[logs/]
        K[data/ipc/{folder}/]
        L[data/sessions/{folder}/]
    end

    A <-->|loadState/saveState| E
    B <-->|JSON stringify/parse| E
    C <-->|getAllSessions/setSession| F
    D <-->|getAllRegisteredGroups/setRegisteredGroup| G

    D -.->|resolveGroupFolderPath| I
    I --> J
    D -.->|resolveGroupIpcPath| K
    D -.->|session directory| L
```

## Configuration & Triggers

### Trigger Pattern Matching

```mermaid
flowchart LR
    A[Incoming Message] --> B{Is main group?}
    B -->|Yes| C[Process immediately]
    B -->|No| D{requiresTrigger?}
    D -->|No| C
    D -->|Yes| E{TRIGGER_PATTERN<br/>test content}
    E -->|Match| C
    E -->|No match| F[Skip - accumulate context]
```

### Message Formatting

```xml
<!-- src/router.ts formatMessages output -->
<messages>
  <message sender="Alice" time="2026-02-24T10:30:00Z">
    Hello @NanoClaw, can you help?
  </message>
  <message sender="Bob" time="2026-02-24T10:31:00Z">
    I need assistance too
  </message>
</messages>
```

## Error Handling & Retries

### Retry Strategy

```mermaid
flowchart TD
    A[Container error] --> B{Had output?}
    B -->|Yes| C[Don't rollback cursor<br/>Prevent duplicates]
    B -->|No| D[Rollback cursor]

    D --> E[scheduleRetry]
    E --> F[retryCount++]
    F --> G{retryCount > 5?}
    G -->|Yes| H[Drop messages<br/>Reset count]
    G -->|No| I[Calculate delay]
    I --> J[delay = 5s * 2^retryCount]
    J --> K[setTimeout enqueue]

    C --> L[Return success]
```

## Startup & Shutdown

### Startup Sequence

```mermaid
sequenceDiagram
    participant M as main()
    participant CRT as Container Runtime
    participant DB as Database
    participant LD as loadState()
    participant CH as Channels
    participant SS as Subsystems

    M->>CRT: ensureContainerSystemRunning()
    CRT->>CRT: Start Docker/apple-container
    CRT->>CRT: cleanupOrphans()

    M->>DB: initDatabase()
    DB->>DB: Create tables
    DB->>DB: Run migrations
    DB->>DB: migrateJsonState()

    M->>LD: loadState()
    LD->>DB: Load router_state
    LD->>DB: Load sessions
    LD->>DB: Load registered_groups

    M->>CH: Connect WhatsApp
    CH->>CH: Baileys auth
    CH->>CH: Sync groups

    M->>SS: Start Scheduler
    M->>SS: Start IPC Watcher
    M->>SS: Set up GroupQueue
    M->>SS: Recover pending messages
    M->>SS: Start Message Loop
```

### Graceful Shutdown

```mermaid
sequenceDiagram
    participant SIG as SIGTERM/SIGINT
    participant IDX as index.ts
    participant GQ as GroupQueue
    participant CH as Channels

    SIG->>IDX: shutdown(signal)
    IDX->>GQ: queue.shutdown(10000)
    GQ->>GQ: shuttingDown = true
    GQ->>GQ: Log active containers
    Note over GQ: Don't kill containers<br/>Let them finish naturally

    IDX->>CH: ch.disconnect()
    CH->>CH: Close connections

    IDX->>IDX: process.exit(0)
```

## Directory Structure

```
nanoclaw/
├── src/                    # Core orchestrator code
│   ├── index.ts           # Main orchestrator
│   ├── channels/          # Channel implementations
│   │   └── whatsapp.ts   # Baileys integration
│   ├── container-runner.ts  # Container management
│   ├── group-queue.ts    # Concurrency control
│   ├── db.ts             # Database operations
│   ├── ipc.ts            # IPC watcher
│   ├── task-scheduler.ts # Scheduled tasks
│   └── router.ts         # Message formatting
├── container/             # Agent container
│   ├── Dockerfile
│   ├── agent-runner/     # Agent code
│   └── skills/           # Shared skills
├── groups/               # Group data
│   ├── main/            # Main group workspace
│   ├── {folder}/        # Other groups
│   └── global/          # Shared read-only memory
├── data/                 # Runtime data
│   ├── ipc/             # IPC directories
│   ├── sessions/        # Per-group sessions
│   └── store/           # Database, auth
└── docs/                # Documentation
```

## Key Design Principles

1. **Isolation**: Each group runs in its own container with isolated filesystem
2. **Security**: Main group has elevated privileges; non-main groups are sandboxed
3. **Persistence**: SQLite for metadata, filesystem for group data
4. **IPC**: File-based IPC for container↔orchestrator communication
5. **Streaming**: Agent output streams back in real-time
6. **Queue**: GroupQueue manages concurrency and retries
7. **Recovery**: Cursor-based message tracking with rollback on error
