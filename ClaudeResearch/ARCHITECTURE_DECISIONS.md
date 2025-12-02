# Chartsmith Enhancement: Architecture Decisions Document

## Executive Summary

This document outlines the architectural decisions for enhancing Chartsmith across two pull requests:

| PR | Scope | Timeline |
|----|-------|----------|
| **PR1** | Vercel AI SDK Migration + Multi-Provider Support | Days 1-3 |
| **PR2** | Chart Validation Agent + Live Provider Switching | Days 4-6 |

The combined work delivers a modernized, maintainable, and feature-rich AI-powered Helm chart assistant.

---

## ⚠️ Critical Architecture Notes (Updated After Codebase Analysis)

### Actual Current Architecture

After thorough codebase analysis, the Chartsmith architecture differs from initial assumptions:

| Initial Assumption | Actual Reality |
|-------------------|----------------|
| Next.js calls LLM directly | **Go backend handles ALL LLM calls** |
| HTTP REST between frontend/backend | **PostgreSQL LISTEN/NOTIFY + Centrifugo** |
| Standard streaming (SSE) | **Centrifugo WebSocket pub/sub** |
| `/api/chat` route exists | **No such route** - workspace-specific routes only |
| Go backend has HTTP server | **No HTTP server** - purely worker-based |
| Single LLM provider | **Dual provider: Anthropic Claude + Groq Llama** |

### Current LLM Providers

| Provider | SDK | Models | Usage |
|----------|-----|--------|-------|
| **Anthropic** | `anthropic-sdk-go v0.2.0-alpha.11` | `claude-3-7-sonnet-20250219`, `claude-3-5-sonnet-20241022` | Planning, execution, chat, tools |
| **Groq** | `jpoz/groq` | `llama-3.3-70b-versatile` | Intent detection, feedback, file conversion |
| **Voyage** | Custom HTTP | Voyage embeddings | Vector search for recommendations |

### PR1 Creates Parallel System

PR1 does NOT replace the existing architecture. It creates a **NEW parallel chat system**:

```
EXISTING (Unchanged):  Frontend → Workspace API → PostgreSQL → Go Worker → Anthropic
NEW (PR1 Creates):     Frontend → NEW /api/chat → OpenRouter (via AI SDK)
```

### PR2 Architecture Decision Required

PR2's `validateChart` tool needs to call Go backend for helm/kube-score execution. But Go has no HTTP server.

**Decision required before PR2 implementation - see `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md`**:

| Option | Approach | Sync? | New Pattern? |
|--------|----------|-------|--------------|
| **A** | Add HTTP endpoint to Go | Yes | Yes (recommended) |
| **B** | Use PostgreSQL queue | No | No (consistent) |
| **C** | Next.js spawns Go CLI | Yes | Partial |

---

## Goals & Constraints

### Primary Goals
1. **Reduce Maintenance Burden**: Replace custom chat implementation with standardized SDK
2. **Enable Provider Flexibility**: Support multiple LLM providers via single integration
3. **Add Validation Intelligence**: AI-powered chart validation with actionable feedback
4. **Improve Developer Experience**: Better streaming, state management, and UX

### Constraints
- Maintain existing VS Code extension authentication (no changes)
- Preserve all current Chartsmith functionality
- Keep Go backend as primary for validation operations (learning opportunity)
- 6-day total development timeline
- Use OpenRouter for multi-provider access (no Anthropic API key required)

---

## Architecture Overview

### Current State
```
┌─────────────────────────────────────────────────────────────────┐
│                      CURRENT ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│  Frontend          │  Custom chat UI, Jotai state, minimal SDK  │
│  API Layer         │  PostgreSQL queue + Centrifugo WebSocket   │
│  Backend (Go)      │  ALL LLM calls, tools, chart processing    │
│  Providers         │  Anthropic Claude + Groq Llama             │
└─────────────────────────────────────────────────────────────────┘
```

**Existing Go Tools** (3 total):
- `text_editor_20241022` - File view/create/str_replace operations
- `latest_subchart_version` - ArtifactHub subchart lookup
- `latest_kubernetes_version` - K8s version info

### Target State (After PR2)
```
┌─────────────────────────────────────────────────────────────────┐
│                      TARGET ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│  Frontend          │  Vercel AI SDK useChat, tool rendering     │
│  API Layer         │  streamText, tool calling, multi-provider  │
│  Backend (Go)      │  Validation pipeline, helm/kube-score      │
│  Providers         │  OpenRouter → Claude, GPT-4, Gemini, etc.  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Decision Records

### ADR-001: Use Vercel AI SDK for Chat Implementation

**Context**: Chartsmith uses custom chat UI and streaming implementation with direct Anthropic SDK calls.

**Decision**: Migrate to Vercel AI SDK (`ai` + `@ai-sdk/react`)

**Rationale**:
- Standardized hooks reduce custom code by ~60%
- Built-in streaming optimization
- Framework-agnostic, well-maintained by Vercel
- Native tool calling support for validation agent
- Easy provider switching via unified API

**Consequences**:
- (+) Significantly reduced maintenance burden
- (+) Better streaming performance out of box
- (+) Type-safe message handling
- (-) Learning curve for SDK patterns
- (-) Some custom UI adaptations needed

---

### ADR-002: Use OpenRouter as Primary Provider

**Context**: Need multi-provider support without managing multiple API keys. User has OpenRouter key but no Anthropic key.

**Decision**: Use `@openrouter/ai-sdk-provider` as the primary provider interface

**Rationale**:
- Single API key for 300+ models
- Access to Claude, GPT-4, Gemini, etc.
- Native Vercel AI SDK integration
- Pay-per-use across all providers
- Simplified key management

**Configuration**:
```typescript
const openrouter = createOpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
});

// Access any model with same interface
const model = openrouter('anthropic/claude-3.5-sonnet');
```

**Consequences**:
- (+) No need for individual provider API keys
- (+) Easy model switching without code changes
- (+) Unified billing and usage tracking
- (-) Slight latency overhead vs direct APIs
- (-) Dependent on OpenRouter availability

---

### ADR-003: Provider Switching UX - Phased Approach

**Context**: Need to demonstrate provider flexibility while managing scope.

**Decision**: Implement provider switching in two phases:
- **PR1**: Per-conversation selector (shown at start of new chat)
- **PR2**: Live mid-conversation switching (Cursor-style dropdown)

**Rationale**:
- PR1 approach is simpler, proves the SDK value immediately
- PR2 can polish the UX without blocking core migration
- Per-conversation is a natural fit for chat context

**PR1 Implementation**:
```typescript
// Provider selected before first message
{messages.length === 0 && <ProviderSelector onChange={setProvider} />}
```

**PR2 Implementation**:
```typescript
// Always-visible dropdown, handles context transfer
<LiveProviderSwitcher 
  current={provider} 
  onChange={switchProviderMidChat} 
/>
```

---

### ADR-004: Validation Pipeline in Go Backend

**Context**: Chart validation requires running CLI tools (helm lint, helm template, kube-score). Frontend cannot execute these.

**Decision**: Implement validation pipeline in Go backend, call from frontend via Vercel AI SDK tools.

**⚠️ UPDATE: Communication pattern requires decision** - see `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md`

The Go backend currently has NO HTTP server. Three options exist:

**Option A (Recommended)**: Add HTTP endpoint to Go
```
AI SDK Tool → fetch() → NEW Go HTTP /validate → helm/kube-score
```

**Option B**: Use existing PostgreSQL queue pattern
```
AI SDK Tool → pg_notify('validate_chart') → Go Worker → Centrifugo → Frontend
```

**Option C**: Next.js spawns Go CLI
```
AI SDK Tool → Next.js /api/validate → exec(Go binary) → stdout JSON
```

**Rationale for validation in Go**:
- Go is well-suited for CLI orchestration
- Aligns with assignment goal (learn Go)
- Keeps sensitive operations server-side
- Natural fit for existing Go codebase
- Tool calling pattern enables AI interpretation

**Architecture (Option A)**:
```
┌─────────────┐     Tool Call     ┌──────────────┐
│  AI (Chat)  │ ─────────────────▶│  Go Backend  │
│             │                   │  NEW HTTP    │
│  "Validate  │   POST /validate  │  endpoint    │
│   my chart" │ ◀─────────────────│  helm lint   │
│             │   JSON Results    │  kube-score  │
└─────────────┘                   └──────────────┘
```

---

### ADR-005: Validation Depth Level

**Context**: Multiple validation depths possible (basic → advanced)

**Decision**: Implement "Intermediate" level:
1. `helm lint` - Syntax and structure
2. `helm template` - Rendering verification  
3. `kube-score` - Kubernetes best practices

**Rationale**:
- Basic (lint only) doesn't catch K8s issues
- Advanced (Trivy, Checkov) adds security scanning but increases scope/complexity
- Intermediate provides meaningful value without scope creep
- Can extend to Advanced in future PRs

**Not Included (Future)**:
- Trivy vulnerability scanning
- Checkov security policies
- OPA/Rego policy checks

---

### ADR-006: Preserve VS Code Extension Auth

**Context**: VS Code extension uses token-based authentication via `/api/auth/status`

**Decision**: Make no changes to authentication system

**Rationale**:
- Auth system is outside migration scope
- Currently working, no bugs reported
- Changes could break extension users
- Focus resources on core objectives

**Verification**: Ensure `/api/auth/status` endpoint remains functional after migration.

---

### ADR-007: OpenAI as Default Provider

**Context**: Need a sensible default when no provider explicitly selected.

**Decision**: Default to OpenAI (`gpt-4o`) via OpenRouter, with Anthropic Claude as demonstrated alternative.

**Rationale**:
- GPT-4o is widely available, fast, capable
- Lower cost than Claude for default usage
- Demo will show switching to Claude
- User has OpenRouter key (covers both)

**Configuration**:
```env
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o
```

---

## Feature Breakdown

### PR1: Vercel AI SDK Migration

| Feature | Description | Files Affected |
|---------|-------------|----------------|
| SDK Installation | Add ai, @ai-sdk/react, @openrouter/ai-sdk-provider | package.json |
| useChat Migration | Replace custom chat with useChat hook | components/Chat.tsx |
| API Route | New streamText-based route handler | app/api/chat/route.ts |
| Provider Factory | Abstraction for model selection | lib/ai-provider.ts |
| Provider Selector | Per-conversation model picker UI | components/ProviderSelector.tsx |
| Message Rendering | Parts-based rendering for streaming | components/Message.tsx |
| State Cleanup | Remove custom state management | Multiple files |

### PR2: Validation Agent + Polish

| Feature | Description | Files Affected |
|---------|-------------|----------------|
| Validation Endpoint | Go API for helm lint/template/kube-score | pkg/api/validate.go |
| Validation Pipeline | Orchestrate validation tools | pkg/validation/pipeline.go |
| Tool Definition | Vercel AI SDK tool for validation | lib/tools/validate.ts |
| Tool Rendering | UI for validation results | components/ValidationResults.tsx |
| Live Switching | Mid-conversation provider switching | components/LiveProviderSwitcher.tsx |
| Context Handling | Preserve context on provider switch | lib/ai-provider.ts |

---

## API Contracts

### Chat API (PR1)

**Endpoint**: `POST /api/chat`

**Request**:
```json
{
  "messages": [
    { "role": "user", "content": "Create a nginx deployment" }
  ],
  "provider": "anthropic",
  "model": "anthropic/claude-3.5-sonnet"
}
```

**Response**: Data stream (Vercel AI SDK format)

---

### Validation API (PR2)

**Endpoint**: `POST /api/validate`

**Request**:
```json
{
  "chartPath": "/charts/my-app",
  "values": { "replicaCount": 3 },
  "strictMode": false
}
```

**Response**:
```json
{
  "validation": {
    "overall_status": "warning",
    "results": {
      "helm_lint": { "status": "pass", "issues": [] },
      "helm_template": { "status": "pass", "rendered_resources": 5 },
      "kube_score": {
        "status": "warning",
        "score": 7,
        "issues": [
          {
            "severity": "critical",
            "resource": "Deployment/my-app",
            "check": "container-resources",
            "message": "Container does not have memory limit",
            "suggestion": "Add resources.limits.memory"
          }
        ]
      }
    }
  }
}
```

---

## Testing Strategy

### Existing Tests
- Location: `test_chart/`, `testdata/`
- Must pass after migration
- Update mocks if SDK changes response format

### New Tests (PR1)
- useChat hook integration tests
- API route unit tests with mock providers
- Provider factory tests

### New Tests (PR2)
- Go validation pipeline unit tests
- Tool execution integration tests
- kube-score output parsing tests

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SDK version incompatibility | Medium | High | Pin versions, test thoroughly |
| OpenRouter downtime | Low | Medium | Document fallback to direct OpenAI |
| Go CLI tool failures | Medium | Medium | Graceful error handling, partial results |
| Breaking existing chat flows | Medium | High | Preserve message format, test regressions |
| Timeline overrun | Medium | Medium | Scope PR2 features carefully |

---

## Future Enhancements (Post-PR2)

These features are candidates for future work but explicitly out of scope:

1. **Values Schema Validation**: AI-generated JSON schemas for values.yaml
2. **Environment Drift Detection**: Compare charts across environments
3. **Dependency Vulnerability Alerts**: Security scanning via Trivy
4. **Natural Language Chart Querying**: Ask questions about chart structure
5. **Advanced Security Scanning**: Checkov/OPA integration
6. **Conversation History Persistence**: Save/load chat sessions

---

## Success Metrics

### PR1 Success
- [ ] All existing tests pass
- [ ] Chat functionality equivalent to current
- [ ] Streaming performance equal or better
- [ ] Provider switching works (per-conversation)
- [ ] Clean commit history

### PR2 Success
- [ ] Validation agent responds to "validate" requests
- [ ] helm lint, helm template, kube-score all integrated
- [ ] AI interprets results and suggests fixes
- [ ] Live provider switching works
- [ ] Demo video shows full flow

---

## References

- [Vercel AI SDK Docs](https://ai-sdk.dev/docs)
- [OpenRouter Provider](https://github.com/OpenRouterTeam/ai-sdk-provider)
- [Helm CLI Reference](https://helm.sh/docs/helm/)
- [kube-score](https://kube-score.com/)
- [Chartsmith Repository](https://github.com/replicatedhq/chartsmith)
