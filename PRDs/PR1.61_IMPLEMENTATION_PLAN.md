# PR1.61 - AI SDK Body Parameter Fix Implementation Plan

## Overview

PR1.61 is a hotfix for critical issues discovered after PR1.6 (Feature Parity) completion. The root cause is the AI SDK v5 `useChat` hook's `body` parameter being captured at initialization and becoming stale on subsequent requests.

**Issues Addressed:**
- Issue #1: No streaming on first message from entry point
- Issue #2: Tool hallucination (AI outputs fake XML instead of using real tools)
- Issue #5: First message vs subsequent message behavior difference

## Current State Analysis

### The Problem

In `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx:69-89`:

```typescript
const {
  messages,
  sendMessage,
  status,
} = useChat({
  api: "/api/chat",
  body: {
    provider: selectedProvider,
    model: selectedModel,
    workspaceId: workspace.id,      // Captured at first render - STALE
    revisionNumber,                  // Captured at first render - STALE
  },
  // ...
});
```

When users navigate from `/test-ai-chat` landing page:
1. Workspace is created
2. User is redirected to `/test-ai-chat/[workspaceId]?prompt=...`
3. `useChat` initializes with body containing `workspaceId`
4. `useEffect` fires `sendMessage()` immediately
5. **BUT** the body values were captured at initialization and may be stale

### Impact

On the API side (`chartsmith-app/app/api/chat/route.ts:113-115`):

```typescript
const tools = workspaceId
  ? createTools(authHeader, workspaceId, revisionNumber || 1)
  : undefined;  // No tools = AI hallucinates XML
```

When `workspaceId` is undefined/stale, tools are not created, causing the AI to "hallucinate" tool syntax from training data instead of using the actual tool schemas.

### Key Discoveries

- **Database query order is correct**: `listMessagesForWorkspace` uses `ORDER BY created_at ASC` (chat.ts:56)
- **AI SDK v5 documented behavior**: Body values frozen at initialization is documented in [AI SDK Troubleshooting](https://ai-sdk.dev/docs/troubleshooting/use-chat-stale-body-data)
- **Correct pattern**: Use request-level body in `sendMessage()` second argument

## Desired End State

After this fix:
1. First message from redirect streams correctly
2. Tools execute properly (no XML hallucination)
3. `workspaceId` and `revisionNumber` are always current at request time
4. All subsequent messages continue working as before

### Verification

1. Navigate to `/test-ai-chat`
2. Enter prompt: "What's the latest PostgreSQL chart version?"
3. Submit and observe redirect
4. **Expected**: Streaming response with actual tool execution (`latestSubchartVersion`)
5. **Expected**: Correct answer (e.g., "18.1.13"), not hallucinated version

## What We're NOT Doing

- **Issue #4 (conversation order reversal)**: Separate investigation needed - ORDER BY is correct, may be UI rendering order
- **Centrifugo integration**: Deferred to PR1.7
- **Revision tracking**: Deferred to PR1.7
- **Any UI changes**: This is a minimal fix to body parameter handling only

## Implementation Approach

This is a **surgical fix** - minimal changes to address the root cause:
1. Remove `body` from `useChat` configuration
2. Create a helper function to generate fresh body params
3. Pass body in each `sendMessage()` call

---

## Important Note for Implementing Agent

**Line numbers in this plan are approximate.** Debug logging was added during investigation which may have shifted line positions. Always **locate code by pattern matching** rather than exact line numbers:

- Find `useChat({` to locate the hook configuration
- Find `// Auto-send prompt from URL` to locate the auto-send useEffect
- Find `const handleChatSubmit` to locate the manual send handler
- Find `// Debug: Log status` to locate debug logging (Phase 2)

---

## Phase 1: Fix Body Parameter Handling

### Overview

Modify `client.tsx` to pass body parameters at request time instead of hook initialization.

### Key Safety Addition

The auto-send useEffect adds a **`workspace?.id` guard** to prevent sending with undefined workspaceId:

```typescript
if (prompt && !hasAutoSent && messages.length === 0 && workspace?.id) {
//                                                      ^^^^^^^^^^^^
// This guard ensures we don't fire sendMessage before workspace is available
```

### Changes Required

#### 1.1 Remove body from useChat and add getChatBody helper

**File**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Changes**:
1. Add `useCallback` import
2. Remove `body` from `useChat` configuration
3. Add `getChatBody` helper function

**Import change (line 3)**:
```typescript
import React, { useState, useEffect, useRef, useCallback } from "react";
```

**Replace lines 69-89** (useChat configuration):
```typescript
// Helper to get fresh body params for each request
const getChatBody = useCallback(() => ({
  provider: selectedProvider,
  model: selectedModel,
  workspaceId: workspace.id,
  revisionNumber,
}), [selectedProvider, selectedModel, workspace.id, revisionNumber]);

const {
  messages,
  sendMessage,
  status,
  error,
} = useChat({
  api: "/api/chat",
  // NOTE: body is NOT passed here - it would be captured at initialization and become stale
  // Instead, we pass body in each sendMessage() call to ensure fresh values
  experimental_throttle: STREAMING_THROTTLE_MS,
  onError: (err) => {
    console.error('[useChat] Error:', err);
  },
  onFinish: () => {
    console.log('[useChat] Finished');
  },
});
```

#### 1.2 Update auto-send useEffect

**File**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Replace lines 114-123** (auto-send useEffect):
```typescript
// Auto-send prompt from URL if present
useEffect(() => {
  const prompt = searchParams.get("prompt");
  if (prompt && !hasAutoSent && messages.length === 0 && workspace?.id) {
    console.log('[useChat] Auto-sending prompt from URL:', prompt);
    console.log('[useChat] Body params:', getChatBody());
    setPendingPrompt(prompt); // Show prompt immediately in UI
    setHasAutoSent(true);
    sendMessage(
      { text: prompt },
      { body: getChatBody() }  // Fresh values at request time
    );
  }
}, [searchParams, hasAutoSent, messages.length, sendMessage, workspace?.id, getChatBody]);
```

#### 1.3 Update manual send handler

**File**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Replace lines 172-194** (handleChatSubmit):
```typescript
const handleChatSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  if (chatInput.trim() && !isLoading) {
    const messageText = chatInput.trim();
    setChatInput("");

    try {
      // 1. Persist user message using AI SDK-specific action (skips Go intent processing)
      const chatMessage = await createAISDKChatMessageAction(
        session,
        workspace.id,
        messageText
      );
      setCurrentChatMessageId(chatMessage.id);

      // 2. Send to AI SDK with fresh body params
      await sendMessage(
        { text: messageText },
        { body: getChatBody() }  // Fresh values at request time
      );
    } catch (error) {
      console.error("Failed to persist message:", error);
      // Still send to AI SDK even if persistence fails
      await sendMessage(
        { text: messageText },
        { body: getChatBody() }
      );
    }
  }
};
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] No linting errors: `cd chartsmith/chartsmith-app && npm run lint`
- [ ] App starts without errors: `cd chartsmith/chartsmith-app && npm run dev`

#### Manual Verification:
- [ ] Navigate to `/test-ai-chat`, enter prompt "What's the latest PostgreSQL chart version?"
- [ ] Observe redirect to workspace URL with prompt in query string
- [ ] Verify streaming response appears (not blank until refresh)
- [ ] Verify actual tool execution in server logs: `[/api/chat] Tools created: { hasTools: true, toolNames: [...] }`
- [ ] Verify correct answer (actual version from Bitnami, e.g., 18.1.13)
- [ ] Test second message in same session works correctly
- [ ] Verify Network tab shows `workspaceId` in POST body (not undefined)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that streaming and tool execution work correctly.

---

## Phase 2: Remove Debug Logging (Optional Cleanup)

### Overview

After confirming the fix works, optionally remove verbose debug logging added during investigation.

### Changes Required

#### 2.1 Clean up debug logs

**File**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Remove or reduce** the following useEffect blocks (lines 92-104):
```typescript
// Debug: Log status and message changes
useEffect(() => {
  console.log('[useChat] Status changed:', status);
}, [status]);

useEffect(() => {
  console.log('[useChat] Messages updated:', messages.length, messages);
}, [messages]);

useEffect(() => {
  if (error) {
    console.error('[useChat] Error state:', error);
  }
}, [error]);
```

**Keep error logging** but remove verbose status/message logging:
```typescript
// Log errors for debugging
useEffect(() => {
  if (error) {
    console.error('[useChat] Error:', error);
  }
}, [error]);
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith/chartsmith-app && npm run build`
- [ ] No linting errors: `cd chartsmith/chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Console output is cleaner (no spam on every status change)
- [ ] Errors are still logged appropriately

---

## Testing Strategy

### Unit Tests

No new unit tests required - this is a behavioral fix to existing code.

### Integration Tests

If integration tests exist for the chat flow, verify they pass after changes.

### Manual Testing Steps

1. **Fresh workspace flow**:
   - Go to `/test-ai-chat`
   - Enter: "What subchart versions are available for postgresql?"
   - Submit and wait for redirect
   - Verify streaming response with tool execution

2. **Existing workspace flow**:
   - Navigate directly to `/test-ai-chat/[existing-workspace-id]`
   - Enter: "List the files in this workspace"
   - Verify immediate streaming response
   - Verify `getChartContext` tool executes

3. **Multiple messages flow**:
   - Send first message (verify works)
   - Send second message (verify still works)
   - Refresh page
   - Send another message (verify still works)

4. **Network verification**:
   - Open Browser DevTools → Network tab
   - Filter for `chat` requests
   - Submit a message
   - Click on POST request to `/api/chat`
   - Verify Request Payload contains:
     ```json
     {
       "messages": [...],
       "workspaceId": "<actual-uuid>",
       "revisionNumber": 1,
       "provider": "anthropic",
       "model": "anthropic/claude-sonnet-4-20250514"
     }
     ```

## Performance Considerations

None - this fix does not affect performance. The `getChatBody` callback is memoized with `useCallback` and only recomputes when dependencies change.

## Migration Notes

No migration needed - this is a client-side code fix only.

## References

- Root cause research: `docs/research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md`
- Issue tracking: `docs/issues/PR1.6_POST_IMPLEMENTATION_ISSUES.md`
- AI SDK documentation: [Stale Body Data Troubleshooting](https://ai-sdk.dev/docs/troubleshooting/use-chat-stale-body-data)
- AI SDK cookbook: [Custom Body from useChat](https://ai-sdk.dev/cookbook/next/send-custom-body-from-use-chat)

---

## Summary of Changes

| File | Change |
|------|--------|
| `client.tsx:3` | Add `useCallback` to imports |
| `client.tsx:69-89` | Replace `useChat` config - remove body, add `getChatBody` helper |
| `client.tsx:114-123` | Update auto-send to use `getChatBody()` in sendMessage |
| `client.tsx:172-194` | Update manual send to use `getChatBody()` in sendMessage |
| `client.tsx:92-104` | (Optional) Remove debug logging |

**Total lines changed**: ~50 lines modified
**Risk level**: Low - surgical fix to well-understood issue
**Rollback**: Revert single file to restore original behavior
