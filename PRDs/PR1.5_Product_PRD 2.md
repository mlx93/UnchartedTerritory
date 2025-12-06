# PR1.5: AI SDK Tool Integration - Product Requirements Document

**Version**: 1.0
**PR**: PR1.5 (Migration & Feature Parity)
**Prerequisite**: PR1 merged
**Status**: Ready for Implementation
**Last Updated**: December 2, 2025

---

## Executive Summary

### Purpose

PR1.5 completes the Vercel AI SDK migration by adding **tool support** to the chat system created in PR1. This enables the AI to execute actions such as creating charts, editing files, and looking up version information - restoring full feature parity with the legacy Go-based chat system.

### Business Value

- **Full feature parity**: All existing chat capabilities work in the new AI SDK system
- **Simplified architecture**: Go becomes pure application logic (no LLM calls)
- **Foundation for PR2**: Establishes tool → Go HTTP pattern for future tools
- **Deprecation path**: Legacy Go LLM code can be marked for removal

### Scope Summary

PR1.5 delivers:
1. **6 AI SDK tools** - createChart, getChartContext, updateChart, textEditor, latestSubchartVersion, latestKubernetesVersion
2. **Go HTTP server** on port 8080 with tool execution endpoints
3. **Shared LLM client library** for consistent AI configuration
4. **System prompt migration** from Go to TypeScript
5. **Removal of @anthropic-ai/sdk** from TypeScript codebase

### Key Architecture Decision

**Go does NOT make LLM calls after PR1.5.**

All "thinking" (chat, tool orchestration, reasoning) moves to AI SDK Core (Node). Go becomes a pure service layer for deterministic operations only.

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
- Create new charts via the AI SDK chat
- Edit chart files via the AI SDK chat
- Look up subchart or Kubernetes versions
- Perform any workspace operations

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
AI SDK tools (TypeScript)
    ↓ HTTP POST
Go HTTP Server (port 8080)
    ↓ handler functions
Existing Go functions
    ↓
Database
```

---

## User Stories

### US-1: Create Chart via Chat (Core Demo Requirement)

**As a** Chartsmith user
**I want** to create a new Helm chart by describing it in chat
**So that** I can quickly scaffold charts without manual file creation

**Acceptance Criteria**:
- User describes desired chart (e.g., "Create a nginx deployment chart")
- AI invokes createChart tool automatically
- Workspace created in database with Chart.yaml, values.yaml, templates/
- User sees confirmation with workspace details
- New chart appears in workspace list
- File structure matches existing "New Chart" button behavior

**Trigger Conditions**:
- `revisionNumber === 0` (new chart mode)
- User expresses creation intent ("create", "new chart", "build a chart")

### US-2: View Chart Context

**As a** Chartsmith user
**I want** the AI to understand my current chart's contents
**So that** it can provide relevant suggestions and modifications

**Acceptance Criteria**:
- AI can load current workspace metadata
- AI can view chart files and their contents
- AI understands chart structure (deployments, services, etc.)
- Context automatically loaded when discussing existing charts

### US-3: Update Chart via Chat

**As a** Chartsmith user
**I want** to modify my chart through natural language
**So that** I can make changes without editing YAML directly

**Acceptance Criteria**:
- User describes desired changes (e.g., "Add 3 replicas")
- AI applies changes to appropriate files
- New revision created with changes
- User sees before/after diff
- Chart re-renders with updates

### US-4: File Editing with Text Editor Tool

**As a** Chartsmith user
**I want** the AI to view, create, and edit specific files
**So that** I have fine-grained control over chart contents

**Acceptance Criteria**:
- AI can view file contents (`view` command)
- AI can create new files (`create` command)
- AI can replace text in files (`str_replace` command)
- Error messages for non-existent files guide correct action
- Fuzzy matching handles minor discrepancies in old_str

**Commands Supported**:
| Command | Purpose | Required Parameters |
|---------|---------|---------------------|
| `view` | Read file contents | workspaceId, path |
| `create` | Create new file | workspaceId, path, content |
| `str_replace` | Replace text | workspaceId, path, oldStr, newStr |

**Note**: No `insert` command - this does not exist in the current implementation.

### US-5: Subchart Version Lookup

**As a** Chartsmith user
**I want** the AI to find the latest version of Helm subcharts
**So that** I can add current dependencies to my chart

**Acceptance Criteria**:
- User asks about subchart versions (e.g., "What's the latest PostgreSQL chart version?")
- AI queries ArtifactHub API
- Latest version returned with chart name
- Works for common charts (redis, postgresql, nginx, etc.)
- Returns "?" for unknown charts

### US-6: Kubernetes Version Information

**As a** Chartsmith user
**I want** to know current Kubernetes version information
**So that** I can target the correct API versions in my charts

**Acceptance Criteria**:
- User asks about K8s versions (e.g., "What's the latest Kubernetes version?")
- AI provides version in requested format (major, minor, patch)
- Default returns full patch version (e.g., "1.32.1")
- Information accurate for chart compatibility decisions

**Current Behavior** (Maintained for Stability):
| Field | Response |
|-------|----------|
| major | "1" |
| minor | "1.32" |
| patch | "1.32.1" |

### US-7: Error Handling

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
- Modify `/api/chat/route.ts` to register all 6 tools
- Tools passed to `streamText()` via tools parameter
- Each tool has description, parameters schema, execute function

#### FR-1.2: Tool Execute Pattern
All tools follow consistent pattern:
1. Tool receives parameters from AI
2. Execute function calls Go HTTP endpoint
3. Go handler performs operation
4. JSON response returned to AI
5. AI incorporates result into conversation

### FR-2: createChart Tool

#### FR-2.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| name | string | Yes | Chart name (e.g., "nginx", "my-app") |
| description | string | Yes | What the chart should do |
| type | enum | No | deployment, statefulset, cronjob, custom |

#### FR-2.2: Response
```
{
  "success": true,
  "workspaceId": "string",
  "revisionNumber": 0,
  "message": "Chart created successfully"
}
```

#### FR-2.3: Behavior
- Creates workspace in database
- Copies bootstrap chart structure
- Creates initial revision (revision 0)
- Triggers render via pg_notify

### FR-3: getChartContext Tool

#### FR-3.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| workspaceId | string | Yes | Workspace ID to load |
| includeFiles | boolean | No | Include file contents |

#### FR-3.2: Response
Returns workspace object with charts, files, revisions, and plans.

### FR-4: updateChart Tool

#### FR-4.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| workspaceId | string | Yes | Workspace to update |
| changes | array | Yes | List of changes to apply |
| message | string | Yes | Commit message |

#### FR-4.2: Changes Format
```
{
  "type": "setValue" | "addResource" | "removeResource" | "updateMetadata",
  "path": "string",
  "value": "any"
}
```

### FR-5: textEditor Tool

#### FR-5.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| command | enum | Yes | view, create, str_replace |
| workspaceId | string | Yes | Workspace context |
| path | string | Yes | File path relative to chart root |
| content | string | For create | Full file content |
| oldStr | string | For str_replace | Text to find |
| newStr | string | For str_replace | Replacement text |

#### FR-5.2: Command Behaviors

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

### FR-6: latestSubchartVersion Tool

#### FR-6.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| chartName | string | Yes | Subchart name (e.g., "postgresql") |
| repository | string | No | Specific repository |

#### FR-6.2: Response
```
{
  "success": true,
  "version": "15.2.0",
  "name": "postgresql"
}
```

#### FR-6.3: Implementation Notes
- Queries ArtifactHub API for version lookup
- Special handling for "replicated" charts (GitHub API)
- 45-minute cache for Replicated charts

### FR-7: latestKubernetesVersion Tool

#### FR-7.1: Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| semverField | enum | No | major, minor, or patch (default: patch) |

#### FR-7.2: Response
```
{
  "success": true,
  "version": "1.32.1",
  "field": "patch"
}
```

#### FR-7.3: Implementation Notes
- Returns hardcoded values (intentional for stability)
- No external API call (current behavior maintained)

### FR-8: Shared LLM Client Library

#### FR-8.1: Exports
- `runChat()` - Wrapper around streamText with tools
- `getModel()` - Provider factory for OpenRouter
- `AVAILABLE_MODELS` - Configuration array

#### FR-8.2: Usage
All chat functionality flows through this library for consistent configuration.

### FR-9: System Prompt Migration

#### FR-9.1: Content Migration
- Copy prompts from `pkg/llm/system.go`
- Adapt for AI SDK tool format
- Document all 6 available tools

#### FR-9.2: Tool Guidance
System prompt must include:
- When to use each tool
- New chart mode behavior (revisionNumber === 0)
- Tool invocation best practices

### FR-10: Remove @anthropic-ai/sdk

#### FR-10.1: Removal Scope
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
TEST "Create chart via chat"
  1. POST "Create a nginx chart" to /api/chat
  2. Assert createChart tool invoked
  3. Assert Go endpoint called
  4. Assert workspace exists in database
  5. Assert response includes workspace details
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
| Create chart | Say "Create nginx chart" | New workspace created |
| View file | Say "Show me deployment.yaml" | File contents displayed |
| Edit file | Say "Change replicas to 3" | File updated, new revision |
| Subchart version | Say "PostgreSQL version?" | Returns latest version |
| K8s version | Say "Latest K8s version?" | Returns 1.32.1 |

---

## Success Criteria

### Must Pass (All Required)

- [ ] "Create a nginx chart" works end-to-end via AI SDK chat
- [ ] AI SDK chat invokes createChart and Go creates valid workspace
- [ ] createChart tool invokes Go endpoint correctly
- [ ] Chart appears in database after creation
- [ ] latestSubchartVersion returns valid version data
- [ ] latestKubernetesVersion returns valid version data
- [ ] textEditor view/create/str_replace work correctly
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration tests pass
- [ ] Error responses follow standard format

### Quality Gates

- [ ] All 4 feature-parity tools implemented (createChart, textEditor, latestSubchartVersion, latestKubernetesVersion)
- [ ] All 6 tools implemented for full quality (adds getChartContext, updateChart)
- [ ] System prompts document all 6 tools
- [ ] callGoEndpoint utility shared across all tools
- [ ] All Go endpoints enforce workspace ownership
- [ ] Extension token authentication works

---

## Priority Classification

### MUST HAVE (Minimum Viable PR1.5)

| Priority | Task | Rationale |
|----------|------|-----------|
| 1 | createChart + Go endpoint | Demo requirement |
| 2 | llmClient.ts + tool registration | Foundation |
| 3 | System prompts | Tool guidance |
| 4 | latestSubchartVersion | Feature parity |
| 5 | latestKubernetesVersion | Feature parity |
| 6 | textEditor | Feature parity |
| 7 | Error response contract | Production quality |
| 8 | Integration test | Validation |

### NICE TO HAVE

| Priority | Task | Rationale |
|----------|------|-----------|
| 9 | getChartContext | New capability |
| 10 | updateChart | New capability |
| 11 | Documentation updates | Cleanup |

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

## Appendix: Existing Tools Being Ported

| Legacy Go Tool | AI SDK Tool | Location |
|----------------|-------------|----------|
| text_editor_20241022 | textEditor | pkg/llm/execute-action.go |
| latest_subchart_version | latestSubchartVersion | pkg/llm/conversational.go |
| latest_kubernetes_version | latestKubernetesVersion | pkg/llm/conversational.go |

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
