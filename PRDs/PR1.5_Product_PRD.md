# PR1.5: AI SDK Tool Integration - Product Requirements Document

**Version**: 2.0
**PR**: PR1.5 (Migration & Feature Parity)
**Prerequisite**: PR1 merged
**Status**: Ready for Implementation
**Last Updated**: December 3, 2025

---

## Executive Summary

### Purpose

PR1.5 completes the Vercel AI SDK migration by adding **tool support** to the chat system created in PR1. This enables the AI to view chart context, edit files, and look up version information - restoring full feature parity with the legacy Go-based chat system.

### Business Value

- **Full feature parity**: All existing tool capabilities work in the new AI SDK system
- **Simplified architecture**: Go becomes pure application logic (no LLM calls)
- **Foundation for PR2**: Establishes tool → Go HTTP pattern for future tools
- **Deprecation path**: Legacy Go LLM code can be marked for removal

### Scope Summary

PR1.5 delivers:
1. **4 AI SDK tools** - getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion
2. **Go HTTP server** on port 8080 with 3 tool execution endpoints
3. **TypeScript-only getChartContext** (direct call to getWorkspace(), no Go endpoint)
4. **Shared LLM client library** for consistent AI configuration
5. **System prompt migration** from Go to TypeScript
6. **Removal of @anthropic-ai/sdk** from TypeScript codebase

### Key Architecture Decisions

**Go does NOT make LLM calls after PR1.5.**

All "thinking" (chat, tool orchestration, reasoning) moves to AI SDK Core (Node). Go becomes a pure service layer for deterministic operations only.

**Workspace creation is NOT a tool.**

Workspaces are created by the existing homepage TypeScript flow before the AI SDK chat begins. The AI SDK chat only needs tools for operations **within** an existing workspace.

---

## Problem Statement

### Current State (After PR1)

PR1 created a new AI SDK chat system, but it lacks tools:

```
NEW AI SDK Chat (PR1)              EXISTING System (Go-based)
────────────────────               ─────────────────────────
/api/chat → OpenRouter             Server Actions → DB → pg_notify → Go
NO TOOLS                           3 tools (text_editor, version tools)
```

Users cannot:
- View chart context via the AI SDK chat
- Edit chart files via the AI SDK chat
- Look up subchart or Kubernetes versions

### Pain Points

1. **Incomplete migration**: New chat system missing core functionality
2. **Two systems**: Users must use legacy system for tool operations
3. **No deprecation path**: Cannot retire legacy Go LLM code

### Desired State (After PR1.5)

A unified AI SDK chat system with full tool support:

```
AI SDK Chat Path (unified)
────────────────────────────
/api/chat → OpenRouter
    ↓ tool calls
AI SDK tools (4 total)
    ├─ getChartContext → TypeScript direct (getWorkspace())
    └─ textEditor, version tools → Go HTTP
                                    ↓
                              Go HTTP Server (port 8080)
                                    ↓
                              Existing Go functions
                                    ↓
                              Database
```

**Note**: Workspace creation happens BEFORE chat begins (via homepage flow).

---

## User Stories

### US-1: View Chart Context (Core Capability)

**As a** Chartsmith user
**I want** the AI to understand my current chart's contents
**So that** it can provide relevant suggestions and modifications

**Acceptance Criteria**:
- AI can load current workspace metadata via getChartContext tool
- AI can view chart files and their contents
- AI understands chart structure (deployments, services, etc.)
- Context automatically loaded when discussing existing charts
- Tool calls TypeScript getWorkspace() directly (no Go endpoint)

**Note**: Workspace already exists when chat begins (created via homepage flow).

### US-2: File Editing with Text Editor Tool

**As a** Chartsmith user
**I want** the AI to view, create, and edit specific files
**So that** I have fine-grained control over chart contents

**Acceptance Criteria**:
- AI can view file contents (`view` command)
- AI can create new files (`create` command)
- AI can replace text in files (`str_replace` command)
- Error messages for non-existent files guide correct action
- Fuzzy matching handles minor discrepancies in old_str
- Tool calls Go HTTP endpoint (`POST /api/tools/editor`)

**Commands Supported**:
| Command | Purpose | Required Parameters |
|---------|---------|---------------------|
| `view` | Read file contents | workspaceId, path |
| `create` | Create new file | workspaceId, path, content |
| `str_replace` | Replace text | workspaceId, path, oldStr, newStr |

**Note**: No `insert` command - this does not exist in the current implementation.

### US-3: Subchart Version Lookup

**As a** Chartsmith user
**I want** the AI to find the latest version of Helm subcharts
**So that** I can add current dependencies to my chart

**Acceptance Criteria**:
- User asks about subchart versions (e.g., "What's the latest PostgreSQL chart version?")
- AI queries ArtifactHub API via Go HTTP endpoint
- Latest version returned with chart name
- Works for common charts (redis, postgresql, nginx, etc.)
- Returns "?" for unknown charts
- Tool calls Go HTTP endpoint (`POST /api/tools/versions/subchart`)

### US-4: Kubernetes Version Information

**As a** Chartsmith user
**I want** to know current Kubernetes version information
**So that** I can target the correct API versions in my charts

**Acceptance Criteria**:
- User asks about K8s versions (e.g., "What's the latest Kubernetes version?")
- AI provides version in requested format (major, minor, patch)
- Default returns full patch version (e.g., "1.32.1")
- Information accurate for chart compatibility decisions
- Tool calls Go HTTP endpoint (`POST /api/tools/versions/kubernetes`)

**Current Behavior** (Maintained for Stability):
| Field | Response |
|-------|----------|
| major | "1" |
| minor | "1.32" |
| patch | "1.32.1" |

### US-5: Error Handling

**As a** Chartsmith user
**I want** clear error messages when tool operations fail
**So that** I understand what happened and how to recover

**Acceptance Criteria**:
- Network errors display user-friendly messages
- Authorization failures return appropriate status (401/404)
- Tool-specific errors explain the issue
- Error format consistent across all tools
- Retry suggestions provided where applicable

---

## Functional Requirements

### FR-1: Tool Registration in Chat Route

#### FR-1.1: Tool Integration
- Modify `/api/chat/route.ts` to register all 4 tools
- Tools passed to `streamText()` via tools parameter
- Each tool has description, parameters schema, execute function

#### FR-1.2: Request Body Change from PR1
**PR1 request body**: `{ messages, provider, model }`
**PR1.5 request body**: `{ messages, model, workspaceId, revisionNumber }`

Tools need `workspaceId` to operate on the correct workspace.

#### FR-1.3: Auth Header Forwarding
The AI SDK `tool({ execute })` pattern only receives tool parameters, not HTTP request context.
Solution: Extract auth header in route handler and pass via closure when creating tools.

#### FR-1.4: Tool Execute Patterns
**For Go-backed tools** (textEditor, latestSubchartVersion, latestKubernetesVersion):
1. Tool receives parameters from AI
2. Execute function calls Go HTTP endpoint with auth header from closure
3. Go handler performs operation
4. JSON response returned to AI
5. AI incorporates result into conversation

**For TypeScript-only tools** (getChartContext):
1. Tool receives parameters from AI
2. Execute function calls TypeScript function directly (getWorkspace())
3. JSON response returned to AI
4. AI incorporates result into conversation

### FR-2: getChartContext Tool (TypeScript-Only)

#### FR-2.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| workspaceId | string | Yes | Workspace ID to load |

#### FR-2.2: Response
Returns workspace object with charts, files, revisions, and plans.

#### FR-2.3: Implementation
- **No Go HTTP endpoint** - calls TypeScript `getWorkspace()` directly
- Simpler architecture, no network hop needed

### FR-3: textEditor Tool (Go HTTP)

#### FR-3.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| command | enum | Yes | view, create, str_replace |
| workspaceId | string | Yes | Workspace context |
| path | string | Yes | File path relative to chart root |
| content | string | For create | Full file content |
| oldStr | string | For str_replace | Text to find |
| newStr | string | For str_replace | Replacement text |

#### FR-3.2: Command Behaviors

**view**:
- If file exists: return content
- If file missing: return "Error: File does not exist. Use create instead."

**create**:
- If file exists: return "Error: File already exists. Use view and str_replace instead."
- If file missing: create with provided content

**str_replace**:
- If oldStr found: replace and return new content
- If oldStr not found: use fuzzy matching with 10s timeout
- If no match: return "Error: String to replace not found in file."

### FR-4: latestSubchartVersion Tool (Go HTTP)

#### FR-4.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| chartName | string | Yes | Subchart name (e.g., "postgresql") |
| repository | string | No | Specific repository |

#### FR-4.2: Response
```
{
  "success": true,
  "version": "15.2.0",
  "name": "postgresql"
}
```

#### FR-4.3: Implementation Notes
- Queries ArtifactHub API for version lookup
- Special handling for "replicated" charts (GitHub API)
- 45-minute cache for Replicated charts

### FR-5: latestKubernetesVersion Tool (Go HTTP)

#### FR-5.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| semverField | enum | No | major, minor, or patch (default: patch) |

#### FR-5.2: Response
```
{
  "success": true,
  "version": "1.32.1",
  "field": "patch"
}
```

#### FR-5.3: Implementation Notes
- Returns hardcoded values (intentional for stability)
- No external API call (current behavior maintained)

### FR-6: Shared LLM Client Library

#### FR-6.1: Relationship with PR1's provider.ts
PR1 created `lib/ai/provider.ts` with `getModel()` and `AVAILABLE_PROVIDERS`.
PR1.5's `llmClient.ts` **imports from and extends** `provider.ts` - it does NOT replace it.

#### FR-6.2: Exports
- `runChat()` - NEW wrapper around streamText with tools support
- `getModel()` - Re-exported from PR1's provider.ts
- `AVAILABLE_PROVIDERS` - Re-exported from PR1's provider.ts

#### FR-6.3: Usage
All chat functionality flows through this library for consistent configuration.

### FR-7: System Prompt Migration

#### FR-7.1: Content Migration
- Copy prompts from `pkg/llm/system.go`
- Adapt for AI SDK tool format
- Document all 4 available tools

#### FR-7.2: Tool Guidance
System prompt must include:
- When to use each tool
- Workspace already exists when chat begins
- Tool invocation best practices
- Reference to existing <chartsmithArtifact> and <chartsmithActionPlan> tags

### FR-8: Remove @anthropic-ai/sdk

#### FR-8.1: Removal Scope
- Remove from package.json
- Delete lib/llm/prompt-type.ts
- Verify no remaining imports

---

## Non-Functional Requirements

### NFR-1: Performance

| Metric | Target |
|--------|--------|
| Tool execution latency | < 500ms (excluding LLM) |
| HTTP round-trip to Go | < 100ms |
| File operation (view) | < 200ms |
| File operation (edit) | < 500ms |
| ArtifactHub lookup | < 2s |

### NFR-2: Reliability

- Tool calls retry on transient failures
- Graceful degradation on Go endpoint unavailability
- No data loss on partial failures
- Transaction rollback on multi-step failures

### NFR-3: Security

- Extension token validation for all tool endpoints
- Workspace ownership verification
- 401/404 for unauthorized access (not 403 for security)
- No sensitive data in error messages

### NFR-4: Compatibility

- All tools work with both GPT-4o and Claude 3.5 Sonnet
- Tool schemas compatible with AI SDK format
- JSON request/response (no streaming for tool calls)

---

## Testing Requirements

### Unit Tests

#### Go Handler Tests
- Each handler validates input correctly
- Each handler returns proper error codes
- Each handler calls expected functions
- Authorization middleware works

#### TypeScript Tool Tests
- Tool schemas validate correctly
- Execute functions handle errors
- Response parsing works

### Integration Tests

#### End-to-End Tool Flow
```
TEST "View chart context"
  1. POST "What files are in my chart?" to /api/chat
  2. Assert getChartContext tool invoked
  3. Assert TypeScript getWorkspace() called (no Go HTTP)
  4. Assert response includes file list
```

```
TEST "Edit file via chat"
  1. POST "Change replicas to 3" to /api/chat
  2. Assert textEditor tool invoked
  3. Assert Go endpoint called
  4. Assert file updated
```

#### Authentication Tests
```
TEST "Unauthorized request returns 401"
TEST "Wrong workspace returns 404"
TEST "Valid token returns 200"
```

### Manual Testing Checklist

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| View context | Say "What's in my chart?" | File list and metadata displayed |
| View file | Say "Show me deployment.yaml" | File contents displayed |
| Edit file | Say "Change replicas to 3" | File updated |
| Subchart version | Say "PostgreSQL version?" | Returns latest version |
| K8s version | Say "Latest K8s version?" | Returns 1.32.1 |

---

## Success Criteria

### Must Pass (All Required)

- [ ] getChartContext returns workspace data via TypeScript getWorkspace()
- [ ] textEditor view/create/str_replace work via Go HTTP endpoint
- [ ] latestSubchartVersion returns valid version data via Go HTTP endpoint
- [ ] latestKubernetesVersion returns valid version data via Go HTTP endpoint
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration tests pass
- [ ] Error responses follow standard format

### Quality Gates

- [ ] All 4 tools implemented (getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion)
- [ ] getChartContext is TypeScript-only (no Go HTTP endpoint)
- [ ] System prompts document all 4 tools
- [ ] callGoEndpoint utility shared across Go-backed tools
- [ ] All Go endpoints enforce workspace ownership
- [ ] Extension token authentication works
- [ ] **ARCHITECTURE.md updated** (Replicated requirement)

---

## Priority Classification

### MUST HAVE (Minimum Viable PR1.5)

| Priority | Task | Rationale |
|----------|------|-----------|
| 1 | getChartContext (TypeScript-only) | Core capability |
| 2 | textEditor + Go endpoint | Feature parity |
| 3 | latestSubchartVersion + Go endpoint | Feature parity |
| 4 | latestKubernetesVersion + Go endpoint | Feature parity |
| 5 | llmClient.ts + tool registration | Foundation |
| 6 | System prompts | Tool guidance |
| 7 | Error response contract | Production quality |
| 8 | Integration test | Validation |
| 9 | **ARCHITECTURE.md update** | **Replicated requirement** |

### NICE TO HAVE

| Priority | Task | Rationale |
|----------|------|-----------|
| 10 | Deprecation banner | Cleanup |

---

## Dependencies

### External Dependencies
- PR1 merged and deployed
- Go backend accessible
- PostgreSQL database running
- OpenRouter API available

### Internal Dependencies
- Existing Go workspace functions
- Existing file operation functions
- Existing recommendations package

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Go HTTP server startup issues | Medium | High | Start in goroutine, log errors |
| Tool schema incompatibility | Low | Medium | Validate against AI SDK docs |
| Auth token validation differs | Medium | Medium | Match exact Next.js pattern |
| Workspace creation SQL differs | Medium | High | Copy SQL from TypeScript |

---

## Out of Scope (PR1.5)

The following are explicitly NOT included:
- createChart tool (workspace creation handled by homepage flow)
- updateChart tool (use iterative textEditor instead)
- createEmptyWorkspace tool (workspace creation handled by homepage flow)
- validateChart tool (PR2)
- Live provider switching during conversation (PR2)
- Chart validation agent (PR2)
- Conversation persistence
- Tool result caching
- Real-time K8s version API

---

## What PR1.5 Enables for PR2

After PR1.5, PR2 can:
- Add validateChart tool to existing tools array
- Add /api/validate endpoint to existing Go HTTP server
- Reuse proven tool → Go HTTP → response pattern
- Focus on validation logic, not infrastructure

---

## Appendix: Tools Summary

### Tools Implemented (4 total)

| AI SDK Tool | Implementation | Location |
|-------------|----------------|----------|
| getChartContext | TypeScript-only (no Go) | lib/ai/tools/getChartContext.ts |
| textEditor | Go HTTP endpoint | pkg/api/handlers/editor.go |
| latestSubchartVersion | Go HTTP endpoint | pkg/api/handlers/versions.go |
| latestKubernetesVersion | Go HTTP endpoint | pkg/api/handlers/versions.go |

### Legacy Go Tools Being Wrapped

| Legacy Go Tool | AI SDK Tool | Location |
|----------------|-------------|----------|
| text_editor_20241022 | textEditor | pkg/llm/execute-action.go |
| latest_subchart_version | latestSubchartVersion | pkg/llm/conversational.go |
| latest_kubernetes_version | latestKubernetesVersion | pkg/llm/conversational.go |

### Tools NOT Implemented (handled by existing flows)

| Tool | Reason |
|------|--------|
| createChart | Workspace creation uses homepage flow + queue, not tool call |
| updateChart | Updates use iterative textEditor, not batch operations |
| createEmptyWorkspace | Workspace creation uses homepage flow |

---

## Related Documents

| Document | Purpose |
|----------|---------|
| PR1.5_Tech_PRD.md | Technical specification |
| PR1.5_PLAN.md | High-level task list |
| PR1.5_HIGH_LEVEL_SUMMARY_REVISED.md | Architecture overview |
| CHARTSMITH_ARCHITECTURE_DECISIONS.md | Key decisions |
| docs/research/2025-12-02-pr1.5-implementation-guide.md | Verified implementation guidance |
| docs/research/2025-12-02-pr1.5-codebase-verification.md | Codebase analysis |

---

*Document End - PR1.5: AI SDK Tool Integration Product PRD*
