# PR3.0 Architecture: Legacy vs AI SDK Path

## Overview

This document provides a visual comparison of the legacy Go worker path and the new AI SDK path, highlighting the key modules changed and the reasoning behind architectural decisions.

---

## High-Level Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LEGACY PATH (Go Worker)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input                                                                 │
│      │                                                                      │
│      ▼                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │  TypeScript      │───▶│   PostgreSQL     │───▶│   Go Worker      │      │
│  │  createChat      │    │   work_queue     │    │   (Listener)     │      │
│  │  MessageAction   │    │   + NOTIFY       │    │                  │      │
│  └──────────────────┘    └──────────────────┘    └────────┬─────────┘      │
│                                                           │                 │
│                                                           ▼                 │
│                                                  ┌──────────────────┐       │
│                                                  │  Groq LLM        │       │
│                                                  │  (Intent Class.) │       │
│                                                  └────────┬─────────┘       │
│                                                           │                 │
│                          ┌────────────────────────────────┼────────┐        │
│                          │                                │        │        │
│                          ▼                                ▼        ▼        │
│                   ┌─────────────┐                  ┌──────────┐ ┌───────┐   │
│                   │ Off-Topic   │                  │  Plan    │ │Render │   │
│                   │ Decline     │                  │ Creation │ │       │   │
│                   └─────────────┘                  └────┬─────┘ └───────┘   │
│                                                         │                   │
│                                                         ▼                   │
│                                                  ┌──────────────────┐       │
│                                                  │  Claude LLM      │       │
│                                                  │  (Streaming)     │       │
│                                                  └────────┬─────────┘       │
│                                                           │                 │
│                                                           ▼                 │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │  Frontend        │◀───│   Centrifugo     │◀───│  Update DB       │      │
│  │  (Jotai Atoms)   │    │   (WebSocket)    │    │  + Publish       │      │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                           AI SDK PATH (PR2.0 + PR3.0)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input                                                                 │
│      │                                                                      │
│      ▼                                                                      │
│  ┌──────────────────┐         ┌──────────────────┐                         │
│  │  TypeScript      │────────▶│  Go HTTP API     │  ◀── NEW: /api/intent   │
│  │  /api/chat       │         │  Intent Classify │                         │
│  │  route.ts        │◀────────│                  │                         │
│  └────────┬─────────┘         └──────────────────┘                         │
│           │                                                                 │
│           │ (if not off-topic/proceed/render)                               │
│           ▼                                                                 │
│  ┌──────────────────┐                                                       │
│  │  AI SDK          │                                                       │
│  │  streamText()    │                                                       │
│  │  + Tools         │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           ▼                                                                 │
│  ┌──────────────────┐                                                       │
│  │  Buffered Tools  │  ◀── NEW: Buffer create/str_replace                  │
│  │  (In-Memory)     │       Execute view immediately                        │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           │ onFinish (if buffered calls exist)                              │
│           ▼                                                                 │
│  ┌──────────────────┐         ┌──────────────────┐                         │
│  │  TypeScript      │────────▶│  Go HTTP API     │  ◀── NEW: /api/plan     │
│  │  createPlan      │         │  Create Plan     │                         │
│  │  FromToolCalls   │◀────────│  + Store Buffers │                         │
│  └──────────────────┘         └────────┬─────────┘                         │
│                                        │                                    │
│                                        ▼                                    │
│  ┌──────────────────┐         ┌──────────────────┐                         │
│  │  Frontend        │◀────────│   Centrifugo     │                         │
│  │  PlanChatMessage │         │   (WebSocket)    │                         │
│  │  Proceed/Ignore  │         └──────────────────┘                         │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           │ User clicks Proceed                                             │
│           ▼                                                                 │
│  ┌──────────────────┐         ┌──────────────────┐                         │
│  │  TypeScript      │────────▶│  Go HTTP API     │                         │
│  │  proceedPlan     │         │  /api/tools      │                         │
│  │  Action          │         │  (Execute tools) │                         │
│  └──────────────────┘         └──────────────────┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Modules Changed

### New Files (PR3.0)

| File | Purpose | Path |
|------|---------|------|
| **Intent HTTP Handler** | Expose Groq intent classification via HTTP | `pkg/api/handlers/intent.go` |
| **Plan HTTP Handler** | Create plans from buffered tool calls | `pkg/api/handlers/plan.go` |
| **Conversion HTTP Handler** | Trigger K8s conversion pipeline | `pkg/api/handlers/conversion.go` |
| **Intent Client** | TypeScript client for intent classification | `lib/ai/intent.ts` |
| **Plan Client** | TypeScript client for plan creation | `lib/ai/plan.ts` |
| **Buffered Tools** | Tools that buffer instead of execute | `lib/ai/tools/bufferedTools.ts` |
| **Tool Interceptor** | Buffer management for tool calls | `lib/ai/tools/toolInterceptor.ts` |
| **Proceed Plan Action** | Execute buffered tools on user approval | `lib/workspace/actions/proceed-plan.ts` |

### Modified Files (PR3.0)

| File | Changes |
|------|---------|
| `app/api/chat/route.ts` | Add intent classification, use buffered tools, create plans on finish |
| `pkg/api/server.go` | Register new HTTP endpoints |
| `lib/chat/messageMapper.ts` | Add followup actions generation |
| `hooks/useAISDKChatAdapter.ts` | Add followup actions to response persistence |
| `components/WorkspaceContent.tsx` | Add page reload guard |
| `lib/workspace/actions/commit-pending-changes.ts` | Set rollback field on first message |

### Reused Components (No Changes)

| Component | Why Reused |
|-----------|------------|
| `PlanChatMessage.tsx` | Already handles plan display, Proceed/Ignore |
| `RollbackModal.tsx` | Already handles rollback UI |
| `ConversionProgress.tsx` | Already handles conversion progress display |
| `ChatMessage.tsx` | Already detects `responsePlanId` and renders plan |
| `pkg/llm/intent.go` | Existing Groq intent classification logic |
| `pkg/workspace/plan.go` | Existing plan creation logic |
| `useCentrifugo.ts` | Existing real-time event handling |

---

## Data Flow Diagrams

### Flow 1: User Asks to Create a File

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  "Create a deployment.yaml for nginx"                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. /api/chat receives request                                              │
│  2. Calls Go /api/intent/classify                                           │
│  3. Groq returns: { isPlan: true, isConversational: false, ... }            │
│  4. Route: "ai-sdk" (let AI SDK handle)                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. AI SDK streamText() with buffered tools                                 │
│  6. AI calls textEditor({ command: "create", path: "templates/deploy..." }) │
│  7. Tool BUFFERS the call, returns { success: true, buffered: true }        │
│  8. AI continues explaining what it will create                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  9. onFinish: bufferedToolCalls.length > 0                                  │
│  10. Call Go /api/plan/create-from-tools with buffered calls                │
│  11. Go creates plan record, stores buffered calls in JSONB                 │
│  12. Go sets response_plan_id on chat message                               │
│  13. Go publishes Centrifugo event                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  14. Frontend receives chatmessage-updated event                            │
│  15. ChatMessage detects responsePlanId                                     │
│  16. Renders PlanChatMessage with Proceed/Ignore buttons                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                          ┌─────────┴─────────┐
                          │                   │
                          ▼                   ▼
┌─────────────────────────────────┐ ┌─────────────────────────────────┐
│  User clicks PROCEED            │ │  User clicks IGNORE             │
│  17. proceedPlanAction()        │ │  17. ignorePlanAction()         │
│  18. Fetch buffered_tool_calls  │ │  18. Set plan status: ignored   │
│  19. Execute each via Go API    │ │  19. No file changes            │
│  20. Files written to pending   │ └─────────────────────────────────┘
│  21. User can Commit/Discard    │
└─────────────────────────────────┘
```

### Flow 2: Off-Topic Message

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  "What's the weather today?"                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. /api/chat receives request                                              │
│  2. Calls Go /api/intent/classify                                           │
│  3. Groq returns: { isOffTopic: true, isPlan: false, ... }                  │
│  4. Route: "off-topic"                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. Return polite decline immediately                                       │
│  6. NO AI SDK call (saves cost/time)                                        │
│  7. "I'm designed to help with Helm charts..."                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flow 3: K8s Conversion

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  "Convert these K8s manifests to Helm" + [deployment.yaml, service.yaml]    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. AI SDK calls convertK8sToHelm tool                                      │
│  2. Tool calls Go /api/conversion/start                                     │
│  3. Go creates conversion record                                            │
│  4. Go enqueues "new_conversion" work                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. Go worker picks up conversion job                                       │
│  6. 6-step pipeline: Analyze → Sort → Template → Normalize → Simplify → Done│
│  7. Each step publishes Centrifugo events                                   │
│  8. Frontend ConversionProgress.tsx updates in real-time                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Decision Tree: How We Arrived at This Architecture

### Decision 1: How to Handle Plans in AI SDK Path?

```
                        ┌─────────────────────────┐
                        │ AI SDK needs plan UI    │
                        │ like legacy path        │
                        └───────────┬─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │ Option A  │   │ Option B  │   │ Option C  │
            │ AI calls  │   │ Post-     │   │ Build new │
            │ proposePlan│   │ process   │   │ plan UI   │
            │ tool      │   │ tool calls│   │ for AI SDK│
            └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                  │               │               │
                  ▼               ▼               ▼
            Changes AI      Reuses Go       Duplicates
            behavior,       logic,          400+ lines,
            new prompts     AI unchanged    maintenance
                  │               │               │
                  ▼               ▼               ▼
              REJECTED      ✅ SELECTED       REJECTED
```

**Why Option B?**
- Original requirement: "Keep existing system prompts and behavior"
- AI doesn't know about plans, just calls `textEditor`
- Backend decides when to create plan (exact parity with legacy)

---

### Decision 2: Where to Buffer Tool Calls?

```
                        ┌─────────────────────────┐
                        │ Need to store tool      │
                        │ calls for later exec    │
                        └───────────┬─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │ Option A  │   │ Option B  │   │ Option C  │
            │ In-memory │   │ DB immed- │   │ DB with   │
            │ only      │   │ iately    │   │ plan      │
            └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                  │               │               │
                  ▼               ▼               ▼
            Lost if          Extra writes,   Single atomic
            request fails    complexity      transaction
                  │               │               │
                  ▼               ▼               ▼
              REJECTED        REJECTED      ✅ SELECTED
```

**Why Option C?**
- Atomic: Plan + buffered calls saved together
- No orphaned data if streaming fails
- Matches legacy: plans created after AI finishes

---

### Decision 3: Intent Classification Approach?

```
                        ┌─────────────────────────┐
                        │ Need smart routing      │
                        │ like legacy path        │
                        └───────────┬─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │ Option A  │   │ Option B  │   │ Option C  │
            │ Port Go   │   │ AI SDK    │   │ Skip      │
            │ Groq to   │   │ system    │   │ intent    │
            │ HTTP API  │   │ prompt    │   │ entirely  │
            └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                  │               │               │
                  ▼               ▼               ▼
            Exact parity,   Less deter-     No off-topic
            proven logic    ministic        handling
                  │               │               │
                  ▼               ▼               ▼
            ✅ SELECTED       REJECTED        REJECTED
```

**Why Option A?**
- Requirement: Strict parity with legacy
- 300-500ms latency acceptable
- Reuses battle-tested Groq logic

---

### Decision 4: Which Tools Trigger Plans?

```
                        ┌─────────────────────────┐
                        │ Which tools need user   │
                        │ approval before exec?   │
                        └───────────┬─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │ Option A  │   │ Option B  │   │ Option C  │
            │ Only      │   │ All file  │   │ Config    │
            │ textEditor│   │ tools     │   │ per tool  │
            │ mutations │   │           │   │           │
            └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                  │               │               │
                  ▼               ▼               ▼
            Conservative,   Over-broad,     Complex,
            exact parity    slow UX         over-eng.
                  │               │               │
                  ▼               ▼               ▼
            ✅ SELECTED       REJECTED        REJECTED
```

**Why Option A?**
- Only `textEditor` with `create`/`str_replace` modifies files
- `view` is read-only → execute immediately
- Other tools are read-only → execute immediately

---

## Summary: Legacy vs AI SDK

| Aspect | Legacy Path | AI SDK Path (PR3.0) |
|--------|-------------|---------------------|
| **Transport** | PostgreSQL queue → Go worker | HTTP streaming via AI SDK |
| **Intent Classification** | Go worker calls Groq | TypeScript calls Go HTTP endpoint |
| **LLM Provider** | Go worker calls Claude | AI SDK calls Claude/OpenRouter |
| **Tool Execution** | Go worker executes directly | Buffered, executed on Proceed |
| **Plan Creation** | Go worker creates plan | Go HTTP endpoint creates plan |
| **Plan UI** | PlanChatMessage.tsx | Same (reused) |
| **Real-time Updates** | Centrifugo | Same (reused) |
| **State Management** | Jotai atoms | Same (reused) |

**Key Insight**: The AI SDK path reuses all existing Go logic via HTTP endpoints. The only new logic is the buffering/interception layer in TypeScript. This minimizes risk and ensures exact feature parity.

---

*Last Updated: December 5, 2025*
