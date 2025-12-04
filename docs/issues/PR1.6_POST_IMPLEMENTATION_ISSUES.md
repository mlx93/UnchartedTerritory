# PR1.6 Post-Implementation Issues

**Date**: December 4, 2025
**Status**: ROOT CAUSE IDENTIFIED - Ready for fix
**Related**: PR1.6 Feature Parity, PR1.65 UI Parity Plan
**Research**: [2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md](../research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md)

## Summary

PR1.6 successfully implemented the core infrastructure (workspace creation, file explorer integration, chat persistence), but several streaming and tool execution issues remain unresolved.

## Root Cause (Confirmed via Research)

**The `body` parameter in `useChat` is captured at initialization and becomes stale.**

From AI SDK v5 documentation:
> "The body configuration is captured once when the hook is initialized and doesn't update with subsequent component re-renders."

This explains Issues #1, #2, and #5 - the first message fires with potentially stale/missing `workspaceId`, causing tools not to be created on the server.

---

## Issue 1: No Streaming on First Message from Entry Point

### Description
When a user enters a prompt on `/test-ai-chat` and gets redirected to `/test-ai-chat/[workspaceId]?prompt=...`, the first message does not stream in real-time. Users must refresh the page to see the AI response.

### Current Behavior
1. User enters prompt on `/test-ai-chat`
2. `createWorkspaceFromPromptAction` creates workspace
3. User is redirected to `/test-ai-chat/[workspaceId]?prompt=...`
4. Auto-send fires via `useEffect`, but no visible loading state
5. Response appears only after page refresh

### Expected Behavior
- Prompt should appear immediately in UI
- "Thinking..." bubble should show with spinner
- Response should stream in real-time

### Partial Fix Applied
Added `pendingPrompt` state to show prompt and thinking bubble immediately, but underlying streaming issue persists.

### Files Involved
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

---

## Issue 2: Tool Hallucination (Wrong Answers)

### Description
The AI sometimes hallucinates tool calls using XML syntax instead of actually executing our tools, resulting in wrong/outdated answers.

### Current Behavior
AI responds with hallucinated XML:
```xml
<function_calls>
  <invoke name="lookup_chart_version">
    <parameter name="chart_name">postgresql</parameter>
  </invoke>
</function_calls>
<function_result>
Latest version: 16.2.7  <!-- WRONG - outdated -->
</function_result>
```

### Expected Behavior
AI should use actual tool `latestSubchartVersion` which returns correct version (18.1.13):
```json
{"type":"tool-output-available","toolCallId":"...","output":{"success":true,"version":"18.1.13","name":"postgresql"}}
```

### Root Cause Hypothesis
The `workspaceId` may not be passed correctly in the request body to `/api/chat`, causing tools not to be created:

```typescript
// In /api/chat/route.ts
const tools = workspaceId 
  ? createTools(authHeader, workspaceId, revisionNumber || 1)
  : undefined;  // If workspaceId is undefined, no tools are created
```

### Investigation Needed
1. Verify `body` params in `useChat` are being sent with requests
2. Check API logs to see if `workspaceId` is received
3. Determine if AI SDK v5 `body` option works as expected

### Files Involved
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx` (useChat config)
- `chartsmith-app/app/api/chat/route.ts` (tool creation)

---

## Issue 3: Transport Protocol Mismatch (Fixed)

### Description
`TextStreamChatTransport` expects text stream format, but API returns `toUIMessageStreamResponse()` which uses UI Message Stream format.

### Symptom
Raw SSE data displayed in chat instead of parsed response:
```
data: {"type":"start"}
data: {"type":"start-step"}
data: {"type":"tool-input-start","toolCallId":"..."}
```

### Fix Applied
Removed `TextStreamChatTransport` and used `api` prop directly:

```typescript
// BEFORE (broken)
const transport = new TextStreamChatTransport({ api: "/api/chat", body: {...} });
useChat({ transport, ... });

// AFTER (fixed)
useChat({ api: "/api/chat", body: {...}, ... });
```

### Files Fixed
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`
- `chartsmith-app/components/chat/AIChat.tsx`

---

## Issue 4: Conversation Order Reverses on Refresh

### Description
When the page is refreshed, the conversation history appears in reversed order.

### Current Behavior
Messages from `initialMessages` (loaded from database) display in incorrect order relative to current session messages.

### Investigation Needed
1. Check how `initialMessages` are ordered when fetched from database
2. Check how they're rendered alongside `messages` from useChat
3. Determine if ORDER BY clause is correct in `getWorkspaceMessagesAction`

### Files Involved
- `chartsmith-app/lib/workspace/actions/get-workspace-messages.ts`
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

---

## Issue 5: Subsequent Messages Work, First Message Doesn't

### Description
When already on a workspace URL page (`/test-ai-chat/[workspaceId]`), sending a second message works correctly with streaming and proper tool execution. Only the first message (from redirect) has issues.

### Observation
- 2nd+ messages: ✅ Thinking bubble, ✅ Streaming, ✅ Correct tool execution
- 1st message (from redirect): ❌ No thinking bubble, ❌ No streaming, ❌ Sometimes wrong answer

### Hypothesis
The `useChat` hook or `body` configuration may not be fully initialized when the auto-send `useEffect` fires on component mount.

---

## Debug Logging Added

The following console logs were added for debugging:

### Client-side (browser console)
```
[useChat] Status changed: ...
[useChat] Messages updated: ...
[useChat] Error: ...
[useChat] Finished
[useChat] Auto-sending prompt from URL: ...
[useChat] Body will include workspaceId: ... revisionNumber: ...
```

### Server-side (terminal)
```
[/api/chat] Request received: { hasMessages, provider, model, workspaceId, revisionNumber }
[/api/chat] Tools created: { hasTools, toolNames }
```

---

## Recommended Investigation Steps

1. **Check browser Network tab** for `/api/chat` POST request:
   - Is `workspaceId` in the request body?
   - Is the response streaming (chunked transfer)?

2. **Check browser Console** for debug logs:
   - What status transitions occur?
   - Are there any errors?

3. **Check server terminal** for API logs:
   - What values are received in the request body?
   - Are tools being created?

4. **Test timing issue**:
   - Add delay before auto-send to see if initialization is the issue
   - Check if `body` option updates correctly when workspace changes

---

## Related Documents

- `PRDs/PR1.65_UI_PARITY_PLAN.md` - UI feature parity items
- `docs/research/2025-12-04-PR1.6-FEATURE-PARITY-ANALYSIS.md` - Original analysis
- `PRDs/PR1.7_Tech_PRD.md` - Centrifugo integration (may resolve some issues)

---

## Research Questions (ANSWERED)

### Q1: Does AI SDK v5 `body` option get sent with every request?
**Answer**: Yes, but values are **frozen at initialization**. The body IS sent with every request, but the values captured during hook initialization never update.

### Q2: Is there a race condition when `sendMessage` is called on mount?
**Answer**: No race in `sendMessage` availability - hook initializes synchronously. The issue is stale `body` values, not timing.

### Q3: Correct pattern for dynamic params like `workspaceId`?
**Answer**: Use **request-level body** - pass values in the second argument to `sendMessage()`:
```typescript
sendMessage(
  { text: prompt },
  { body: { workspaceId, revisionNumber, provider, model } }
);
```

### Q4: Does production `/workspace/[id]` use Centrifugo or AI SDK streaming?
**Answer**: **Centrifugo ONLY** - production doesn't use `/api/chat` at all. Messages go through Go backend queue and stream via WebSocket events.

### Q5: Correct transport/response combination for tools?
**Answer**: Default transport (no explicit transport) + `toUIMessageStreamResponse()`. Issue #3 fix was correct.

---

## Recommended Fix

### Step 1: Remove body from useChat config
```typescript
// BEFORE (stale values)
useChat({
  api: "/api/chat",
  body: { workspaceId, revisionNumber, provider, model },
});

// AFTER (no hook-level body)
useChat({
  api: "/api/chat",
});
```

### Step 2: Pass body in sendMessage calls
```typescript
// Auto-send
sendMessage(
  { text: prompt },
  { body: { workspaceId: workspace.id, revisionNumber, provider, model } }
);

// Manual send
sendMessage(
  { text: chatInput },
  { body: { workspaceId: workspace.id, revisionNumber, provider, model } }
);
```

### Step 3: Verify in Network tab
After fix, `/api/chat` POST body should contain:
```json
{
  "messages": [...],
  "workspaceId": "abc123",  // NOT undefined
  "revisionNumber": 1,
  "provider": "anthropic",
  "model": "anthropic/claude-sonnet-4-20250514"
}
```

---

## Implementation File
The fix should be applied in:
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

See full implementation details in:
- `docs/research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md`

