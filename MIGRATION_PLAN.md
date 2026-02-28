# NanoClaw Migration Plan: Claude SDK/CLI → OpenCode SDK/CLI

## Overview

Replace the Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`) and CLI (`@anthropic-ai/claude-code`) with OpenCode SDK (`@opencode-ai/sdk`) and CLI (`opencode`) to reduce costs while preserving as much functionality as possible.

**Current Architecture:**

```
WhatsApp → SQLite → Host → Container (Claude Agent SDK) → Response
```

**Target Architecture:**

```
WhatsApp → SQLite → Host → Container (OpenCode SDK) → Response
```

---

## Why OpenCode?

- **Multi-provider support**: 75+ LLM providers (Anthropic, OpenAI, Google, local models)
- **Cost flexibility**: Use free models or cheaper alternatives
- **Privacy-focused**: No code/context storage
- **Open-source**: Community-driven

---

## Key Differences

| Aspect       | Claude Agent SDK       | OpenCode SDK              |
| ------------ | ---------------------- | ------------------------- |
| Architecture | Direct API             | Client-server (port 4096) |
| Providers    | Anthropic only         | 75+ providers             |
| Agent teams  | Native support         | **Not supported**         |
| Hooks        | PreCompact, PreToolUse | **Not supported**         |
| MCP servers  | Native integration     | **Not supported**         |
| Tools        | Built-in + custom      | Via server API            |
| Session      | File-based             | Server-managed            |

---

## Implementation Phases

### Phase 1: Research & Design

- [ ] Step 1: Deep-dive into OpenCode capabilities
- [ ] Step 2: Design architecture for integration

### Phase 2: Core Implementation

- [ ] Step 3: Create OpenCode agent runner prototype
- [ ] Step 4: Update Docker configuration
- [ ] Step 5: Adapt tool system

### Phase 3: Feature Migration

- [ ] Step 6: Session management
- [ ] Step 7: MCP server integration
- [ ] Step 8: Task scheduler
- [ ] Step 9: Skills system

### Phase 4: Infrastructure

- [ ] Step 10: Hooks functionality
- [ ] Step 11: IPC protocol
- [ ] Step 12: Testing & validation

### Phase 5: Release

- [ ] Step 13: Documentation
- [ ] Step 14: Gradual rollout

---

## Detailed Steps

### Step 1: Research OpenCode Capabilities

**Effort:** 2-3 days

Research questions:

- Does OpenCode support all needed tools (Bash, Read, Write, Edit, Glob, Grep, WebSearch, WebFetch)?
- How does tool streaming work?
- Can OpenCode run in containers?
- Session persistence mechanism?
- Workarounds for agent teams and hooks?

**Deliverable:** Research report documenting findings and recommendations.

---

### Step 2: Design Architecture

**Effort:** 3-4 days

Two main approaches:

**Option A: Per-container server**

```
Host → Container 1 (OpenCode Server + Agent) → Group 1
Host → Container 2 (OpenCode Server + Agent) → Group 2
```

- Pros: Isolation, no network exposure
- Cons: Higher resource usage

**Option B: Host-based server**

```
Host (OpenCode Server) → Container 1 (Client) → Group 1
Host (OpenCode Server) → Container 2 (Client) → Group 2
```

- Pros: Lower resource usage
- Cons: Network exposure, single point of failure

**Recommendation:** Start with Option A for security isolation.

**Deliverable:** Architecture document in `docs/OPENCODE_ARCHITECTURE.md`.

---

### Step 3: Create OpenCode Agent Runner Prototype

**Effort:** 1-2 weeks

Replace `container/agent-runner/src/index.ts`:

```typescript
// Current (Claude SDK)
import { query, HookCallback, PreCompactHookInput, PreToolUseHookInput } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({ prompt, options })) {
  // Handle streaming
}

// New (OpenCode SDK)
import { createOpencode, createOpencodeClient } from '@opencode-ai/sdk';

const { client } = await createOpencode();
const result = await client.session.prompt({ path: { id: sessionId }, body: { parts: [...] } });
```

Key implementation tasks:

- Replace `query()` with OpenCode client calls
- Implement tool abstractions
- Handle streaming output
- Maintain IPC protocol (OUTPUT_START_MARKER, OUTPUT_END_MARKER)

**Deliverable:** Working prototype that can process basic prompts.

---

### Step 4: Update Docker Configuration

**Effort:** 1-2 days

File: `container/Dockerfile`

```dockerfile
# Remove Claude CLI
# RUN npm install -g @anthropic-ai/claude-code

# Add OpenCode CLI
RUN npm install -g opencode

# Keep agent-browser (it's compatible)
RUN npm install -g agent-browser
```

File: `container/agent-runner/package.json`

```json
{
  "dependencies": {
    "@opencode-ai/sdk": "^1.0.0",
    "@anthropic-ai/claude-agent-sdk": "^0.2.34" // Keep for comparison, remove later
  }
}
```

**Deliverable:** Container builds successfully with OpenCode.

---

### Step 5: Adapt Tool System

**Effort:** 2-3 weeks

Tool mapping strategy:

| Claude Tool     | OpenCode Equivalent | Implementation         |
| --------------- | ------------------- | ---------------------- |
| Bash            | OpenCode exec       | Direct mapping         |
| Read/Write/Edit | OpenCode file API   | Direct mapping         |
| Glob/Grep       | OpenCode search API | Direct mapping         |
| WebSearch       | None                | Custom (SerpAPI, etc.) |
| WebFetch        | None                | Custom (fetch wrapper) |
| Task/TaskOutput | None                | **Deprecated**         |
| TeamCreate      | None                | **Deprecated**         |
| agent-browser   | CLI tool            | Keep via Bash          |

**Deprecated tools will be removed from allowedTools list.**

**Deliverable:** All working tools mapped to OpenCode equivalents.

---

### Step 6: Session Management

**Effort:** 1 week

OpenCode sessions are server-managed, not file-based.

Tasks:

- Update SQLite schema if needed
- Map session IDs between host and container
- Handle session compaction differently
- Maintain backward compatibility

```typescript
// Current (Claude)
const sessionPath = path.join(sessionsDir, sessionId, 'transcript');
// New (OpenCode)
const sessionId = await client.session.create({ body: { title: '...' } });
const result = await client.session.prompt({ path: { id: sessionId }, body: {...} });
```

**Deliverable:** Sessions persist correctly across restarts.

---

### Step 7: MCP Server Integration

**Effort:** 2 weeks

Current MCP tools:

- `send_message` - Send WhatsApp message
- `schedule_task` - Schedule recurring task
- `list_tasks` - List scheduled tasks
- `pause_task` / `resume_task` / `cancel_task` - Task management
- `register_group` - Register WhatsApp group

Options for migration:

**Option A:** Subprocess bridge

```
OpenCode → subprocess → MCP server → IPC → Host
```

**Option B:** Direct tool implementation

```
OpenCode → custom tool → IPC → Host
```

**Option C:** OpenCode extensions (if available)

**Deliverable:** All MCP tools functional with OpenCode.

---

### Step 8: Task Scheduler

**Effort:** 1 week

Update `src/task-scheduler.ts`:

- Create new OpenCode sessions for tasks
- Handle context_mode (group vs isolated)
- Ensure output streaming works
- Test cron/interval/once scheduling

```typescript
// Task execution with OpenCode
const session = await client.session.create({ body: { title: task.prompt }});
await client.session.prompt({ path: { id: session.id }, body: { parts: [...] }});
```

**Deliverable:** Scheduled tasks execute correctly.

---

### Step 9: Skills System

**Effort:** 1-2 weeks

Update skills that depend on Claude-specific features:

**Affected skills:**

- `/add-gmail` - Uses MCP (may need rework)
- `/add-voice-transcription` - May need updates
- `/add-discord` - Should work
- `/x-integration` - Uses Claude tool types
- `/add-telegram-swarm` - **Broken** (relies on agent teams)

**Deliverable:** All skills tested and working.

---

### Step 10: Hooks Functionality

**Effort:** 1 week

Current hooks:

- **PreCompact**: Archive transcripts before compaction
- **PreToolUse**: Sanitize Bash environment (remove secrets)

OpenCode doesn't have hooks. Implement as middleware:

```typescript
// PreToolUse equivalent
async function sanitizeBash(command: string): Promise<string> {
  const unsetPrefix = `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN 2>/dev/null; `;
  return unsetPrefix + command;
}

// PreCompact equivalent
async function archiveBeforeClose(): Promise<void> {
  // Archive transcript to conversations/
}
```

**Deliverable:** Equivalent functionality via wrappers.

---

### Step 11: IPC Protocol

**Effort:** 1 week

Maintain current protocol for host compatibility:

```
---NANOCLAW_OUTPUT_START---
{status: "success", result: "...", newSessionId: "..."}
---NANOCLAW_OUTPUT_END---
```

Map OpenCode events to this format:

- OpenCode result → Output marker
- OpenCode errors → Error marker
- OpenCode session events → Session updates

**Deliverable:** Protocol unchanged, host code works as-is.

---

### Step 12: Testing & Validation

**Effort:** 2-3 weeks

**Functional tests:**

- [ ] Basic message processing
- [ ] Session persistence
- [ ] All tool types
- [ ] Scheduled tasks (cron, interval, once)
- [ ] IPC tools (send_message, schedule_task, etc.)
- [ ] Error handling and recovery

**Integration tests:**

- [ ] End-to-end messaging flow
- [ ] Container lifecycle
- [ ] Concurrent load

**Performance tests:**

- [ ] Latency comparison (< 2x Claude)
- [ ] Resource usage (< 2x Claude)

**Security tests:**

- [ ] Secrets isolation
- [ ] Filesystem isolation
- [ ] Container escape prevention

**Deliverable:** All tests passing.

---

### Step 13: Documentation

**Effort:** 1 week

**Files to update:**

- `README.md` - Setup with OpenCode
- `CLAUDE.md` - Add OpenCode notes
- `docs/REQUIREMENTS.md` - Architecture updates
- `docs/OPENCODE_MIGRATION.md` - New migration guide

**Content:**

- Setup instructions
- Configuration (providers, models, credentials)
- Migration guide
- Feature comparison (what changed/deprecated)
- Troubleshooting

**Deliverable:** Complete documentation.

---

### Step 14: Gradual Rollout

**Effort:** 1 week

Feature flag for dual support:

```typescript
// src/config.ts
export const AGENT_RUNTIME = process.env.AGENT_RUNTIME || "claude"; // 'claude' | 'opencode'
```

**Phased release:**

1. Alpha: Internal testing with OpenCode runtime
2. Beta: Optional flag for adventurous users
3. GA: Default to OpenCode, allow Claude fallback

**Deliverable:** Safe rollout with easy rollback.

---

## Feature Deprecations

The following features will NOT work with OpenCode:

| Feature                | Impact | Workaround                  |
| ---------------------- | ------ | --------------------------- |
| Agent teams/subagents  | High   | Single-agent workflows only |
| MCP native integration | Medium | Subprocess bridge           |
| PreCompact hook        | Low    | Middleware wrapper          |
| PreToolUse hook        | Low    | Middleware wrapper          |

**Action:** Update documentation to reflect deprecations. Consider removing agent teams from codebase in future release.

---

## Risk Assessment

**Overall Level: HIGH**

**Key Risks:**

1. **Architectural mismatch** - Server model vs direct API
2. **Feature gaps** - Agent teams, hooks unavailable
3. **Performance** - OpenCode server overhead
4. **Security** - Network exposure in Option B
5. **Testing** - Extensive validation needed

**Mitigation:**

- Phased rollout with feature flags
- Maintain Claude as fallback
- Extensive testing before full deployment
- Clear documentation of trade-offs

---

## Fallback Plan

### Option 1: Dual Runtime (Recommended)

Support both Claude and OpenCode via environment variable:

```bash
# Use OpenCode (new default)
AGENT_RUNTIME=opencode npm run dev

# Use Claude (fallback)
AGENT_RUNTIME=claude npm run dev
```

### Option 2: Phased Migration

1. Chat only (no tools)
2. Basic tools
3. Advanced tools
4. Full migration

### Option 3: Complete Rollback

1. Restore previous version from Git
2. Restore SQLite from backup
3. Clear OpenCode session data

---

## Effort Summary

| Phase                        | Effort    |
| ---------------------------- | --------- |
| Phase 1: Research & Design   | 1-2 weeks |
| Phase 2: Core Implementation | 2-3 weeks |
| Phase 3: Feature Migration   | 2-3 weeks |
| Phase 4: Infrastructure      | 1-2 weeks |
| Phase 5: Release             | 1-2 weeks |

**Total: 7-12 weeks**

---

## Open Questions

Before proceeding, clarify:

1. **Which providers will you use?** (Anthropic, OpenAI, local, etc.)
2. **Is agent teams deprecation acceptable?**
3. **Performance trade-off tolerance?** (willing to accept 2x latency for cost savings?)
4. **Timeline constraints?** (any deadlines?)
5. **Rollback requirements?** (need to maintain Claude indefinitely?)

---

## Next Steps

Start with **Step 1: Research OpenCode Capabilities**

This involves:

1. Installing OpenCode CLI and SDK
2. Running basic examples
3. Testing tool support
4. Evaluating session management
5. Documenting findings

Shall we begin with Step 1?
