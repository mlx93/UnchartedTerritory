# Current Chartsmith Architecture (December 2024)

This document describes the current architecture after PR2.0/PR3.0 AI SDK integration, reflecting the hybrid system that combines the new Vercel AI SDK streaming path with the existing Go worker system.

## Quick Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              REACT FRONTEND                                  │
│  ┌─────────────────┐              ┌─────────────────┐                       │
│  │   Chat UI       │              │  Jotai Atoms    │◄─── useCentrifugo     │
│  │   Components    │◄────────────►│  (State Mgmt)   │     (WebSocket)       │
│  └────────┬────────┘              └─────────────────┘          ▲            │
└───────────┼─────────────────────────────────────────────────────┼────────────┘
            │ useAISDKChatAdapter                                 │
            ▼                                                     │
┌─────────────────────────────────────────────────────────────────┼────────────┐
│                     NEXT.JS + VERCEL AI SDK LAYER               │            │
│                                                                 │            │
│  ┌──────────────────────────────────────────────────────────────┼─────────┐  │
│  │                      /api/chat Route                         │         │  │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┴───────┐ │  │
│  │  │ Intent Classify │───►│   streamText    │───►│  6 TypeScript Tools │ │  │
│  │  │ (plan/proceed/  │    │   (AI SDK v5)   │    │  with Go HTTP calls │ │  │
│  │  │  ai-sdk/render) │    └────────┬────────┘    └─────────────────────┘ │  │
│  │  └─────────────────┘             │                                     │  │
│  └──────────────────────────────────┼─────────────────────────────────────┘  │
│                                     │                                        │
│                                     ▼                                        │
│                          ┌─────────────────┐                                 │
│                          │   OpenRouter    │  ◄── Multi-Provider LLM         │
│                          │  Claude Sonnet  │      (Switch models live!)      │
│                          │  GPT-4o, etc.   │                                 │
│                          └─────────────────┘                                 │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ HTTP (Tool Execution)
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           GO HTTP API LAYER                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  /api/tools/editor  │  /api/validate  │  /api/intent/classify          │ │
│  │  /api/tools/context │  /api/tools/convert  │  /api/plan/create-from-tools │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│        ┌────────────────────────────┴────────────────────────────┐           │
│        ▼                                                         ▼           │
│  ┌─────────────────┐                                    ┌─────────────────┐  │
│  │  Plan Workflow  │                                    │   Centrifugo    │  │
│  │  (buffered tool │                                    │   (Real-time)   │──┼──► WebSocket
│  │   calls, review │                                    │                 │  │
│  │   → applied)    │                                    └─────────────────┘  │
│  └────────┬────────┘                                                         │
│           │                                                                  │
└───────────┼──────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              POSTGRESQL                                       │
│  workspace_plan (buffered_tool_calls)  │  workspace_chat  │  workspace_file  │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key Innovations:**
| Feature | Benefit |
|---------|---------|
| **Vercel AI SDK Layer** | Streaming, tool orchestration, multi-step agents |
| **OpenRouter Multi-LLM** | Switch between Claude, GPT-4, Mistral on the fly |
| **TypeScript Tools → Go HTTP** | Best of both: TS flexibility + Go performance |
| **Intent Classification** | Smart routing: plan vs execute vs conversational |
| **Plan Workflow** | Review before apply, buffered tool calls, undo support |
| **6 Tools** (vs 3) | +validateChart, +convertK8s, +getChartContext |

---

## Detailed Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   FRONTEND (Next.js + React)                                     │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  ┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐   │
│  │    React Chat           │     │   useAISDKChatAdapter   │     │     useCentrifugo       │   │
│  │    Components           │◄───►│   (AI SDK useChat)      │     │   (Real-time Events)    │   │
│  │                         │     │                         │     │                         │   │
│  │  - ChatContainer        │     │  - Message adaptation   │     │  - WebSocket client     │   │
│  │  - PlanChatMessage      │     │  - Persona propagation  │     │  - AI SDK coordination  │   │
│  │  - ChatMessage          │     │  - Cancel support       │     │  - Event handlers       │   │
│  └────────────┬────────────┘     └────────────┬────────────┘     └────────────┬────────────┘   │
│               │                               │                               │                 │
│               │                               │                               │                 │
│  ┌────────────▼────────────────────────────────────────────────────────────────▼────────────┐   │
│  │                                 Jotai Atoms (State Management)                            │   │
│  │                                                                                           │   │
│  │  messagesAtom  │  workspaceAtom  │  planByIdAtom  │  validationAtoms  │  rendersAtom     │   │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │ HTTP / Server Actions
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   NEXT.JS API LAYER                                              │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                        /api/chat (AI SDK Route) [NEW PATH]                              │     │
│  │                                                                                         │     │
│  │   1. Intent Classification ──────► Go /api/intent/classify                              │     │
│  │      │                                                                                  │     │
│  │      ├── off-topic → Polite decline                                                     │     │
│  │      ├── plan → Plan generation (NO TOOLS, plan-only prompt)                            │     │
│  │      ├── proceed → Execute with tools                                                   │     │
│  │      ├── render → Trigger render pipeline                                               │     │
│  │      └── ai-sdk → Process with AI SDK + tools                                           │     │
│  │                                                                                         │     │
│  │   2. AI SDK streamText ───► OpenRouter API ───► LLM (Claude, GPT-4, etc.)               │     │
│  │                                                                                         │     │
│  │   3. Tools (6 total):                                                                   │     │
│  │      ├── getChartContext ─────────► Go HTTP                                             │     │
│  │      ├── textEditor ──────────────► Go HTTP (/api/tools/editor)                         │     │
│  │      ├── latestSubchartVersion ───► Go HTTP                                             │     │
│  │      ├── latestKubernetesVersion ─► Go HTTP                                             │     │
│  │      ├── convertK8sToHelm ────────► Go HTTP (PR3.0)                                     │     │
│  │      └── validateChart ───────────► Go HTTP (/api/validate, PR4)                        │     │
│  │                                                                                         │     │
│  │   4. Buffered Tools → createPlanFromToolCalls → Go /api/plan/create-from-tools          │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                            Server Actions [EXISTING PATH]                               │     │
│  │                                                                                         │     │
│  │   - createChatMessageAction ──────► INSERT + pg_notify()                                │     │
│  │   - createWorkspaceFromPromptAction                                                     │     │
│  │   - proceedPlanAction ────────────► Execute buffered tool calls                         │     │
│  │   - executeViaAISDK ──────────────► Text-only plan execution                            │     │
│  │   - createRevisionAction ─────────► Legacy Go worker path                               │     │
│  │   - getWorkspaceAction / getWorkspaceMessagesAction                                     │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                  │                                                               │
                  │ OpenRouter API                                                │ Go HTTP / SQL
                  ▼                                                               ▼
┌─────────────────────────────────┐                    ┌───────────────────────────────────────────┐
│     OPENROUTER (Multi-LLM)      │                    │              POSTGRESQL                    │
│                                 │                    │                                           │
│  - anthropic/claude-sonnet      │                    │  ┌─────────────────────────────────┐     │
│  - openai/gpt-4o                │                    │  │ workspace_chat                  │     │
│  - Google, Mistral, etc.        │                    │  │ ├── response_plan_id            │     │
│  - Provider switching (PR4)     │                    │  │ ├── response_conversion_id      │     │
│                                 │                    │  │ └── followup_actions            │     │
└─────────────────────────────────┘                    │  └─────────────────────────────────┘     │
                                                       │                                           │
                                                       │  ┌─────────────────────────────────┐     │
                                                       │  │ workspace_plan                  │     │
                                                       │  │ ├── status (review/applying/    │     │
                                                       │  │ │          applied/ignored)     │     │
                                                       │  │ ├── buffered_tool_calls (JSONB) │     │
                                                       │  │ └── action_files                │     │
                                                       │  └─────────────────────────────────┘     │
                                                       │                                           │
                                                       │  ┌─────────────────────────────────┐     │
                                                       │  │ work_queue                      │     │
                                                       │  │ └── LISTEN/NOTIFY (Legacy)      │     │
                                                       │  └─────────────────────────────────┘     │
                                                       │                                           │
                                                       └─────────────────────────────┬─────────────┘
                                                                                     │
                                                                                     │ LISTEN
                                                                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       GO BACKEND                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                               HTTP API Server (pkg/api)                                 │     │
│  │                                                                                         │     │
│  │   Tool Endpoints:                  Plan Endpoints:                 Intent/Validate:     │     │
│  │   ├── /api/tools/editor            ├── /api/plan/create-from-tools  /api/intent/classify│     │
│  │   ├── /api/tools/context           ├── /api/plan/publish-update     /api/validate       │     │
│  │   ├── /api/tools/versions          └── /api/plan/update-action-      (helm lint,       │     │
│  │   └── /api/tools/convert                file-status                   helm template,   │     │
│  │                                                                        kube-score)      │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                          Queue Workers (pkg/listener) [LEGACY]                          │     │
│  │                                                                                         │     │
│  │   Handlers:                                LLM (pkg/llm):                               │     │
│  │   ├── new-conversion.go                    ├── Anthropic SDK (direct)                   │     │
│  │   ├── new-plan.go                          ├── System prompts                           │     │
│  │   ├── execute-plan.go                      └── 3 Tools:                                 │     │
│  │   ├── apply-plan.go                            ├── text_editor                          │     │
│  │   ├── render-workspace.go                      ├── latest_subchart_version              │     │
│  │   ├── publish-workspace.go                     └── latest_kubernetes_version            │     │
│  │   └── slack-notification.go                                                             │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                            Realtime (pkg/realtime)                                      │     │
│  │                                                                                         │     │
│  │   Centrifugo Events:                                                                    │     │
│  │   ├── plan-updated          ─┐                                                          │     │
│  │   ├── chatmessage-updated    │                                                          │     │
│  │   ├── artifact-updated       ├──► Centrifugo Pub/Sub ──► WebSocket ──► Frontend         │     │
│  │   ├── revision-created       │                                                          │     │
│  │   ├── render-stream          │                                                          │     │
│  │   ├── render-file            │                                                          │     │
│  │   ├── conversion-file       ─┘                                                          │     │
│  │   └── conversion-status                                                                 │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │                            Validation Pipeline (pkg/validation)                         │     │
│  │                                                                                         │     │
│  │   helm lint ──► helm template ──► kube-score                                            │     │
│  │                                                                                         │     │
│  └────────────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │ WebSocket (Centrifugo)
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        CENTRIFUGO                                                │
│                                                                                                  │
│  - Real-time message delivery                                                                    │
│  - User-scoped channels (workspaceId#userId)                                                     │
│  - Token-based authentication                                                                    │
│  - Automatic reconnection handling                                                               │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Key Architectural Changes from Original Design

### 1. Dual LLM Path
- **NEW**: AI SDK via OpenRouter (multi-provider: Claude, GPT-4, etc.)
- **LEGACY**: Anthropic SDK direct calls in Go workers

### 2. AI SDK Tools (TypeScript)
Now 6 tools available in the AI SDK path:
| Tool | Purpose | Go Endpoint |
|------|---------|-------------|
| `getChartContext` | Load workspace files/metadata | `/api/tools/context` |
| `textEditor` | View, create, edit files | `/api/tools/editor` |
| `latestSubchartVersion` | ArtifactHub subchart lookup | `/api/tools/versions` |
| `latestKubernetesVersion` | K8s version info | `/api/tools/versions` |
| `convertK8sToHelm` | K8s manifest → Helm (PR3.0) | `/api/tools/convert` |
| `validateChart` | Lint, template, kube-score (PR4) | `/api/validate` |

### 3. Intent Classification
Pre-routes messages before AI SDK processing:
- **off-topic**: Polite decline (no LLM call)
- **plan**: Plan generation phase (NO tools, plan-only prompt)
- **proceed**: Execute existing plan with tools
- **render**: Trigger render pipeline
- **ai-sdk**: Default - process with AI SDK + tools

### 4. Plan Workflow
- **Buffered Tool Calls**: AI SDK streams → buffer tool calls → store in `workspace_plan.buffered_tool_calls`
- **Two-Phase Plans**: Text-only plans (empty tool calls) supported for "describe first, execute later"
- **Plan Execution**: `proceedPlanAction` executes buffered calls when user clicks "Create Chart"
- **Status Flow**: `review` → `applying` → `applied` (or `ignored`)

### 5. Real-Time Coordination
- AI SDK streaming messages tracked in `currentStreamingMessageIdAtom`
- Centrifugo skips updates for messages being streamed by AI SDK
- BUT allows metadata updates (responsePlanId, responseConversionId)

### 6. Provider Switching (PR4)
- Live provider/model switching via `switchProvider()` in adapter
- Selected provider/model stored in hook state
- Passed in body on each sendMessage call

## Data Flow Examples

### Example 1: New Plan Request (AI SDK Path)
```
1. User types "Add a ConfigMap for my app"
2. useAISDKChatAdapter.sendMessage() called
3. createAISDKChatMessageAction persists user message to DB
4. POST /api/chat with {messages, workspaceId, chatMessageId}
5. Intent classification: isPlan=true → route "plan"
6. streamText with plan-only prompt (NO TOOLS)
7. AI streams plan text → UI updates
8. onFinish: createPlanFromToolCalls(empty, text) → plan created
9. Centrifugo: plan-updated event → PlanChatMessage renders
10. User clicks "Create Chart" → proceedPlanAction
11. executeViaAISDK → AI SDK with tools enabled
12. textEditor calls → Go /api/tools/editor → files created
13. Plan status → applied
```

### Example 2: Conversational with Tool Use
```
1. User types "What's in my values.yaml?"
2. Intent: isConversational=true → route "ai-sdk"
3. streamText with tools enabled
4. AI calls getChartContext → returns file contents
5. AI responds with file analysis
6. No plan created (just conversational)
```

### Example 3: Legacy Go Worker Path
```
1. createChatMessageAction with persona="operator"
2. INSERT to workspace_chat + pg_notify('new_intent')
3. Go listener picks up from work_queue
4. pkg/llm with Anthropic SDK
5. Tool execution: text_editor, etc.
6. Centrifugo: chatmessage-updated
```

## Key Files Reference

| Component | Location |
|-----------|----------|
| AI SDK Chat Route | `chartsmith-app/app/api/chat/route.ts` |
| Chat Adapter Hook | `chartsmith-app/hooks/useAISDKChatAdapter.ts` |
| Centrifugo Hook | `chartsmith-app/hooks/useCentrifugo.ts` |
| AI SDK Tools | `chartsmith-app/lib/ai/tools/` |
| Intent Classification | `chartsmith-app/lib/ai/intent.ts` |
| Plan Creation | `chartsmith-app/lib/ai/plan.ts` |
| Proceed Action | `chartsmith-app/lib/workspace/actions/proceed-plan.ts` |
| Go HTTP Handlers | `pkg/api/handlers/` |
| Go Workers | `pkg/listener/` |
| Go LLM | `pkg/llm/` |
| Realtime Events | `pkg/realtime/` |

