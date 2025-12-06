# PR2.0: AI SDK Reintegration into Main Workspace Path - Technical PRD

**Status**: Planning
**Created**: 2025-12-05
**Supersedes**: PR1.8, PR1.9, PR1.10 (parallel path completion approach)
**Depends On**: PR1.7 (completed)

---

## Executive Summary

This PRD defines **Path B**: collapsing the AI SDK transport layer from the `/test-ai-chat` parallel path back into the main `/workspace/[id]` path. Rather than building 27+ features from scratch to achieve parity (Path A: 113-163 hours), we integrate the working AI SDK chat endpoint into the existing, feature-complete main path UI (Path B: estimated 30-40 hours).

### Key Decision: Why Path B?

| Aspect | Path A (Continue Parallel) | Path B (Reintegrate) |
|--------|---------------------------|----------------------|
| **Effort** | 113-163 hours | 30-40 hours |
| **Risk** | High - rebuilding proven features | Low - adapter pattern |
| **Original Requirements** | Exceeds scope | Matches scope exactly |
| **Time to Feature-Complete** | 15-20+ days | 8-12 days |

**The original requirements document asked to "replace custom chat UI with Vercel AI SDK" - not rebuild the entire application.**

---

## Current State Analysis

### What Works in Main Path (`/workspace/[id]`)

| Feature | Status | Notes |
|---------|--------|-------|
| Full Plan Display UI | ✅ | `PlanChatMessage.tsx` with status badges, file lists |
| Plan Proceed/Ignore | ✅ | User control over AI-proposed changes |
| Role/Persona Selector | ✅ | Auto/Developer/Operator dropdown |
| Rollback Functionality | ✅ | `RollbackModal.tsx` with countdown |
| Plan Status Tracking | ✅ | Full lifecycle state machine |
| Per-file Accept/Reject | ✅ | Granular change control |
| Terminal Component | ✅ | Helm render output display |
| Adaptive Layout | ✅ | New chart mode vs editor mode |
| Command Menu (Cmd+K) | ✅ | Quick actions palette |
| Debug Panel | ✅ | Developer tools |
| Full Data Loading | ✅ | 5 parallel fetches on mount |
| Atom Hydration | ✅ | Plans, renders, conversions |
| Centrifugo Integration | ✅ | Real-time updates |
| Message Persistence | ✅ | Full field structure |

### What Works in Parallel Path (`/test-ai-chat`)

| Feature | Status | Notes |
|---------|--------|-------|
| AI SDK Chat Endpoint | ✅ | `/api/chat` with streaming |
| Tool Infrastructure | ✅ | textEditor, getChartContext, etc. |
| useChat Hook | ✅ | AI SDK v5 integration |
| Centrifugo File Updates | ✅ | Real-time artifact-updated events |
| Pending Changes UI | ✅ | Yellow dots, commit/discard |
| Basic Message Rendering | ✅ | ReactMarkdown with tool hiding |

### What's Missing (The Gap)

The parallel path lacks **27 features** that the main path already has. Instead of rebuilding these, we adapt the AI SDK transport to work with the existing UI.

---

## Technical Architecture

### Current Main Path Flow

```
┌─────────────────┐     ┌─────────────────────┐     ┌────────────────┐
│  ChatContainer  │────►│createChatMessageAction│────►│   PostgreSQL   │
│  (User sends)   │     │  (server action)      │     │   + enqueue    │
└─────────────────┘     └─────────────────────┘     └────────────────┘
                                                              │
                                                              ▼
┌─────────────────┐     ┌─────────────────────┐     ┌────────────────┐
│  ChatMessage    │◄────│  Centrifugo events  │◄────│   Go Worker    │
│  (UI updates)   │     │  (WebSocket)        │     │   (LLM calls)  │
└─────────────────┘     └─────────────────────┘     └────────────────┘
```

**Characteristics:**
- Async queue-based processing
- Go worker handles LLM calls and intent classification
- Centrifugo pushes incremental updates to UI
- Full message schema with 15+ fields

### Current Parallel Path Flow

```
┌─────────────────┐     ┌─────────────────────┐     ┌────────────────┐
│  useChat hook   │────►│    /api/chat        │────►│  Claude/OpenAI │
│  (sendMessage)  │     │    (Next.js API)    │     │   (streaming)  │
└─────────────────┘     └─────────────────────┘     └────────────────┘
         │                        │
         │                        ▼
         │                ┌────────────────┐
         │                │   AI SDK Tools │────► Go HTTP Endpoints
         │                │   (textEditor) │      (file operations)
         │                └────────────────┘
         │
         ▼
┌─────────────────┐
│  UIMessage UI   │◄──── Streaming response parts
│  (ReactMarkdown)│
└─────────────────┘
```

**Characteristics:**
- Synchronous streaming via AI SDK
- Tools call Go HTTP endpoints directly
- No intent classification (LLM handles routing)
- Simplified message format (UIMessage with parts)

### Target Architecture (Path B)

```
┌─────────────────┐     ┌─────────────────────┐     ┌────────────────┐
│  ChatContainer  │────►│  useAISDKAdapter    │────►│   /api/chat    │
│  (existing UI)  │     │  (new hook)         │     │   (AI SDK)     │
└─────────────────┘     └─────────────────────┘     └────────────────┘
         │                        │
         ▼                        ▼
┌─────────────────┐     ┌─────────────────────┐
│  ChatMessage    │     │   Message Mapper    │────► Adapts UIMessage
│  PlanChatMessage│◄────│   (UIMessage→Message)│     to existing format
│  Terminal, etc. │     └─────────────────────┘
└─────────────────┘
         ▲
         │
┌─────────────────┐
│   Centrifugo    │◄──── Artifact updates (files)
│   (WebSocket)   │      Plan updates
└─────────────────┘      Render events
```

**Key Principle:** Keep all existing UI components. Only change the transport layer.

---

## Message Format Mapping

### Critical Challenge: UIMessage → Message

The main path expects `Message` type; AI SDK provides `UIMessage` type.

| Field | Message (Main) | UIMessage (AI SDK) | Mapping Strategy |
|-------|----------------|-------------------|------------------|
| `id` | `string` | `string` | Direct copy |
| `prompt` | `string` | `parts[0].text` (role='user') | Extract from parts |
| `response` | `string?` | `parts[].text` (role='assistant') | Join text parts |
| `isIntentComplete` | `boolean?` | `status === 'ready'` | Derive from status |
| `isCanceled` | `boolean?` | N/A (use `stop()`) | Track in adapter state |
| `responsePlanId` | `string?` | Centrifugo `plan-updated` events | **NOT from AI SDK** - Centrifugo driven |
| `responseRenderId` | `string?` | Centrifugo `render-stream` events | **NOT from AI SDK** - Centrifugo driven |
| `responseConversionId` | `string?` | Centrifugo `conversion-status` events | **NOT from AI SDK** - Centrifugo driven |
| `responseRollbackToRevisionNumber` | `number?` | N/A | **Not supported in AI SDK mode** |
| `revisionNumber` | `number?` | Body parameter | Pass through |
| `followupActions` | `any[]?` | N/A | **Not supported in AI SDK mode** |
| `createdAt` | `Date?` | N/A | Generate on send |
| `messageFromPersona` | `string?` | Body parameter `persona` | **Must pass through adapter** |

### Status Flag Mapping (Critical)

The AI SDK `status` field must map to existing UI flags:

| AI SDK Status | `isThinking` | `isStreaming` | `isIntentComplete` | `isComplete` | UI Behavior |
|---------------|--------------|---------------|--------------------|--------------|----|
| `'submitted'` | `true` | `false` | `false` | `false` | Show spinner, "thinking..." |
| `'streaming'` | `false` | `true` | `false` | `false` | Show streaming text, cancel button |
| `'ready'` | `false` | `false` | `true` | `true` | Show complete message |
| `'error'` | `false` | `false` | `true` | `true` | Show error state |

### Cancel State Handling

When user clicks Cancel:
1. Call `stop()` from useChat
2. Set `isCanceled: true` in adapter state
3. Persist to database via `updateChatMessageResponseAction` with partial response
4. UI shows "Message generation canceled" (matches existing ChatMessage behavior)

### Fields NOT Supported in AI SDK Mode

These fields rely on Go worker intent classification and are **explicitly not available** in AI SDK mode:

| Field | Reason | Impact |
|-------|--------|--------|
| `followupActions` | Generated by Go intent classifier | No followup buttons shown |
| `responseRollbackToRevisionNumber` | Go worker rollback detection | Rollback via existing revision history only |
| `isApplied`, `isApplying`, `isIgnored` | Plan workflow states | Use Commit/Discard instead |

### Adapter Interface

```typescript
// hooks/useAISDKChatAdapter.ts

interface AdaptedChatState {
  // Messages in existing format
  messages: Message[];

  // Send function matching existing signature - MUST include persona
  sendMessage: (content: string, persona?: ChatMessageFromPersona) => Promise<void>;

  // Status flags matching existing patterns
  isStreaming: boolean;
  isThinking: boolean;

  // Cancel state (tracked in adapter, not AI SDK)
  isCanceled: boolean;

  // Cancel function - sets isCanceled and calls stop()
  cancel: () => void;

  // Error state
  error: Error | null;
}

function useAISDKChatAdapter(
  workspaceId: string,
  revisionNumber: number,
  session: Session,
  initialMessages: Message[]
): AdaptedChatState
```

### Persona Propagation (Critical)

The adapter MUST pass persona to the `/api/chat` endpoint:

```typescript
// In sendMessage:
await aiSend(
  { text: content },
  {
    body: {
      workspaceId,
      revisionNumber,
      persona  // 'auto' | 'developer' | 'operator'
    }
  }
);
```

The `/api/chat` route must then include persona in the system prompt:

```typescript
// In route.ts:
const systemPrompt = persona === 'developer'
  ? CHARTSMITH_DEVELOPER_PROMPT
  : persona === 'operator'
  ? CHARTSMITH_OPERATOR_PROMPT
  : CHARTSMITH_TOOL_SYSTEM_PROMPT;  // auto mode
```

---

## Implementation Phases

### Phase 1: Adapter Hook Creation (2-3 days)

**Deliverables:**
1. `hooks/useAISDKChatAdapter.ts` - Core adapter hook
2. `lib/chat/messageMapper.ts` - UIMessage ↔ Message conversion
3. Unit tests for message mapping

**Key Implementation Details:**

```typescript
// hooks/useAISDKChatAdapter.ts
export function useAISDKChatAdapter(
  workspaceId: string,
  revisionNumber: number,
  session: Session,
  initialMessages: Message[]
): AdaptedChatState {
  // AI SDK hook for transport
  const { messages: aiMessages, sendMessage: aiSend, status, stop, error } = useChat({
    transport: new DefaultChatTransport({ api: "/api/chat" }),
    experimental_throttle: STREAMING_THROTTLE_MS,
  });

  // Convert AI SDK messages to existing format
  const adaptedMessages = useMemo(() => {
    return [
      ...initialMessages,  // Historical messages from DB
      ...mapUIMessagesToMessages(aiMessages)  // New streaming messages
    ];
  }, [initialMessages, aiMessages]);

  // Wrap send to include workspace context
  const sendMessage = useCallback(async (content: string, persona?: ChatMessageFromPersona) => {
    // 1. Persist user message to DB (for history)
    const dbMessage = await createAISDKChatMessageAction(session, workspaceId, content);

    // 2. Send to AI SDK with fresh body params
    await aiSend(
      { text: content },
      { body: { workspaceId, revisionNumber, persona } }
    );

    // 3. On completion, persist response
    // (handled in useEffect watching status)
  }, [workspaceId, revisionNumber, session, aiSend]);

  return {
    messages: adaptedMessages,
    sendMessage,
    isStreaming: status === 'streaming',
    isThinking: status === 'submitted',
    cancel: stop,
    error,
  };
}
```

### Phase 2: ChatContainer Integration (2-3 days)

**Deliverables:**
1. Modified `ChatContainer.tsx` with feature flag
2. Updated props/state management
3. Integration tests

**Feature Flag Pattern:**

```typescript
// components/ChatContainer.tsx
export function ChatContainer({ session }: ChatContainerProps) {
  const useAISDK = process.env.NEXT_PUBLIC_USE_AI_SDK_CHAT === 'true';
  const [workspace] = useAtom(workspaceAtom);
  const [messages, setMessages] = useAtom(messagesAtom);

  // Conditional transport
  const legacyChat = useLegacyChat(session, workspace);
  const aiSDKChat = useAISDKChatAdapter(
    workspace?.id ?? '',
    workspace?.currentRevisionNumber ?? 0,
    session,
    messages
  );

  const chatState = useAISDK ? aiSDKChat : legacyChat;

  // All existing UI remains unchanged
  return (
    <>
      <ChatMessages messages={chatState.messages} />
      <RoleSelector value={selectedRole} onChange={setSelectedRole} />
      <ChatInput
        onSend={(text) => chatState.sendMessage(text, selectedRole)}
        disabled={chatState.isStreaming}
      />
      {chatState.isThinking && <ThinkingIndicator />}
    </>
  );
}
```

### Phase 3: Plan Workflow Compatibility (2-3 days)

**Challenge:** Main path has deliberate plan review workflow (Proceed/Ignore); AI SDK applies changes directly to `content_pending`.

#### DECISION: Use Commit/Discard as Review Mechanism

**Chosen Approach: Commit/Discard (Option B)**

For PR2.0, the AI SDK path will use the existing Commit/Discard flow as its review mechanism:

| Main Path (Go Worker) | AI SDK Path |
|----------------------|-------------|
| AI creates plan record | AI writes to `content_pending` |
| User sees PlanChatMessage UI | User sees pending indicators (yellow dots) |
| User clicks "Proceed" | User clicks "Commit" |
| User clicks "Ignore" | User clicks "Discard" |
| Creates new revision | Creates new revision |

**Rationale:**
- Commit/Discard already works in the parallel path
- Achieves same outcome (user reviews before finalizing)
- Avoids complex changes to tools and Go endpoints
- Can add full Plan UI parity in a follow-up PR if needed

**What This Means for UI:**
- `PlanChatMessage` component will **NOT show** for AI SDK messages (no `responsePlanId`)
- Users review changes in the editor with pending indicators
- Commit/Discard buttons in tab bar provide the approval step

**Explicit Non-Goals for PR2.0:**
- Creating `workspace_plan` records from AI SDK tool calls
- Showing PlanChatMessage UI for AI SDK messages
- Proceed/Ignore buttons for AI SDK messages

**Future Enhancement (Post-PR2.0):**
If full Plan UI parity is desired later, implement:
```typescript
// lib/ai/tools/textEditor.ts - Future enhancement
tool({
  execute: async (params) => {
    // 1. Create plan record BEFORE applying changes
    const plan = await createPlanAction(workspaceId, description, changes);

    // 2. Write to content_pending (current behavior)
    const result = await callGoEndpoint('/api/tools/editor', ...);

    // 3. Return planId so message can link to plan
    return { ...result, planId: plan.id };
  }
})
```
This would require Go endpoint changes and prompt updates - out of scope for PR2.0.

### Centrifugo Event Parity (Critical)

The main path relies on Centrifugo for real-time updates. The AI SDK path must continue to receive these events:

| Event Type | Handler | AI SDK Path Status |
|------------|---------|-------------------|
| `artifact-updated` | `handleArtifactUpdated` | ✅ Already working (PR1.7) |
| `chatmessage-updated` | `handleChatMessageUpdated` | ⚠️ May conflict with adapter state |
| `plan-updated` | `handlePlanUpdatedAtom` | ✅ Works but no plans created |
| `render-stream` | `handleRenderStreamEvent` | ✅ Works - render output display |
| `render-file` | Render file handler | ✅ Works - rendered files list |
| `revision-created` | Refetch workspace | ✅ Works - after Commit |
| `conversion-file` | `handleConversionFileUpdatedMessage` | ✅ Works - conversion progress |
| `conversion-status` | `handleConversationUpdatedMessage` | ✅ Works - conversion status |

**Key Consideration: Chat Message Updates**

The `chatmessage-updated` event from Centrifugo may conflict with AI SDK streaming state. The adapter must:

1. **Ignore Centrifugo updates for currently-streaming messages** (AI SDK owns that state)
2. **Accept Centrifugo updates for historical messages** (DB is authoritative)
3. **Merge states correctly** when streaming completes

```typescript
// In useCentrifugo.ts - handle AI SDK mode
const handleChatMessageUpdated = (data: RawChatMessage) => {
  // If this message is currently streaming via AI SDK, skip Centrifugo update
  const isCurrentlyStreaming = currentStreamingMessageId === data.id;
  if (isCurrentlyStreaming) return;

  // Otherwise, update from Centrifugo (normal flow)
  setMessages(prev => mergeMessage(prev, data));
};
```

### Phase 4: Testing & Validation (2-3 days)

**Test Matrix:**

| Scenario | Test Method |
|----------|-------------|
| Basic chat send/receive | E2E test |
| Streaming display | Visual verification |
| Persona in request body | Unit test |
| Cancel → isCanceled state | Unit + E2E |
| Commit/Discard flow | E2E test |
| Conversion events | E2E test |
| Rollback functionality | E2E test |
| Error handling | Unit test |
| Feature flag toggle | Integration test |
| Historical message load | E2E test |
| Centrifugo event handling | Integration test |

### Phase 5: Feature Flag Rollout (1 day)

**Environment Configuration:**

```bash
# .env.local (development)
NEXT_PUBLIC_USE_AI_SDK_CHAT=true

# .env.staging
NEXT_PUBLIC_USE_AI_SDK_CHAT=true

# .env.production (gradual rollout)
NEXT_PUBLIC_USE_AI_SDK_CHAT=false  # Start disabled
```

**Rollback Procedure:**
1. Set `NEXT_PUBLIC_USE_AI_SDK_CHAT=false`
2. Redeploy
3. Users immediately fall back to Go worker path

### Phase 6: Cleanup (1-2 days)

**Files to Remove:**
- `/app/test-ai-chat/` directory
- `components/chat/AIChat.tsx`
- `components/chat/AIMessageList.tsx`
- Test-path-specific server actions

**Files to Update:**
- `ARCHITECTURE.md` - Document new flow
- `README.md` - Update setup instructions
- Remove feature flag after stable period

---

## What We Keep vs. What Changes

### Keep Unchanged

| Component | File | Reason |
|-----------|------|--------|
| ChatMessage | `components/ChatMessage.tsx` | Message rendering |
| PlanChatMessage | `components/PlanChatMessage.tsx` | Plan display |
| RollbackModal | `components/RollbackModal.tsx` | Rollback UI |
| Terminal | `components/Terminal.tsx` | Render output |
| CommandMenu | `components/CommandMenu.tsx` | Quick actions |
| All Jotai atoms | `atoms/workspace.ts` | State management |
| Centrifugo hook | `hooks/useCentrifugo.ts` | Real-time updates |
| Plan/Render actions | `lib/workspace/actions/*` | Non-chat operations |

### Modify

| Component | File | Change |
|-----------|------|--------|
| ChatContainer | `components/ChatContainer.tsx` | Add adapter integration |
| Page loader | `app/workspace/[id]/page.tsx` | Feature flag check |

### Create New

| Component | File | Purpose |
|-----------|------|---------|
| Adapter hook | `hooks/useAISDKChatAdapter.ts` | Transport adapter |
| Message mapper | `lib/chat/messageMapper.ts` | Format conversion |

### Remove (After Stable)

| Component | File | Reason |
|-----------|------|--------|
| Test page | `app/test-ai-chat/*` | Replaced by integration |
| AIChat | `components/chat/AIChat.tsx` | Merged into adapter |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Message format edge cases | Medium | Medium | Comprehensive mapping tests |
| Plan workflow mismatch | Medium | High | Option A (pre-confirm) or phased rollout |
| Streaming behavior differences | Low | Medium | Side-by-side visual testing |
| Database sync issues | Low | High | Centrifugo handles, extensive E2E tests |
| Feature flag complexity | Low | Low | Clean separation of concerns |

---

## Success Criteria

### Functional Parity
- [ ] All 27 main path features work with AI SDK transport
- [ ] Message history displays correctly (historical + streaming)
- [ ] Plans display with full UI (badges, files, actions)
- [ ] Proceed/Ignore workflow functions
- [ ] Rollback works from any revision
- [ ] Persona selector affects AI responses
- [ ] Terminal shows render output
- [ ] Command menu operates normally

### Performance
- [ ] Streaming latency ≤ current parallel path
- [ ] Time to first token ≤ 2 seconds
- [ ] No UI jank during streaming

### Reliability
- [ ] Feature flag enables instant rollback
- [ ] Error states handled gracefully
- [ ] No data loss on connection interruption

### Code Quality
- [ ] Adapter hook is testable in isolation
- [ ] Message mapper has 100% coverage
- [ ] No duplication between paths
- [ ] Clean removal of test path possible

---

## Timeline Summary

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Adapter Hook | 2-3 days | None |
| Phase 2: ChatContainer | 2-3 days | Phase 1 |
| Phase 3: Plan Workflow | 2-3 days | Phase 2 |
| Phase 4: Testing | 2-3 days | Phase 3 |
| Phase 5: Rollout | 1 day | Phase 4 |
| Phase 6: Cleanup | 1-2 days | Phase 5 stable |
| **Total** | **10-15 days** | |

---

## Appendix: Message Type Definitions

### Existing Message Type (Main Path)
```typescript
// components/types.ts
interface Message {
  id: string;
  prompt: string;
  response?: string;
  isComplete: boolean;
  isApplied?: boolean;
  isApplying?: boolean;
  isIgnored?: boolean;
  isCanceled?: boolean;
  createdAt?: Date;
  workspaceId?: string;
  userId?: string;
  isIntentComplete?: boolean;
  followupActions?: any[];
  responseRenderId?: string;
  responsePlanId?: string;
  responseConversionId?: string;
  responseRollbackToRevisionNumber?: number;
  planId?: string;
  revisionNumber?: number;
}
```

### AI SDK UIMessage Type
```typescript
// From 'ai' package
interface UIMessage {
  id: string;
  role: 'user' | 'assistant' | 'system';
  parts: MessagePart[];
}

interface MessagePart {
  type: 'text' | 'tool-invocation' | 'tool-result' | string;
  text?: string;
  toolName?: string;
  args?: Record<string, unknown>;
  output?: unknown;
}
```

---

*This PRD defines the technical approach for Path B reintegration. Implementation details will be tracked in the accompanying Implementation Plan document.*
