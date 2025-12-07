# E2E Test Status - December 6, 2025

**Status**: All 3 tests still failing, but root causes identified

---

## Test Results Summary

| Test | Status | Root Cause | Fix Needed |
|------|--------|------------|------------|
| `chat-scrolling.spec.ts` | ❌ Failing | Scroll container not scrollable initially | Wait for enough content |
| `import-artifactory.spec.ts` | ❌ Failing | Terminal not appearing | Timing/wait issues |
| `upload-chart.spec.ts` | ❌ Failing | Plan message not appearing | Plan creation timing |

---

## Detailed Analysis

### Test 1: chat-scrolling.spec.ts

**Current Issue**: 
- Test fails because scroll container doesn't have scrollable content initially
- `canScroll: false` - container needs more content before scrolling is possible

**Fix Applied**:
- ✅ Added wait for scrollable content
- ✅ Added Playwright test flag detection
- ✅ Fixed test helper availability

**Still Failing Because**:
- Need to wait for AI response to complete and add enough content
- Need to ensure messages have sufficient height

**Next Steps**:
- Increase wait time for content
- Add check for scrollable content before proceeding
- Consider sending multiple messages to ensure scrollable content

---

### Test 2: import-artifactory.spec.ts

**Current Issue**:
- Terminal (`[data-testid="chart-terminal"]`) not appearing after render request
- Test waits 30 seconds but terminal never shows

**Possible Causes**:
1. Render not triggered (AI SDK path might not trigger renders)
2. `responseRenderId` not set on message
3. Terminal condition not met: `responseRenderId && !isAutorender && !responsePlanId && isComplete`

**Fix Applied**:
- ✅ Changed selector from `.font-mono` to `[data-testid="chart-terminal"]`
- ✅ Increased timeout to 30 seconds

**Still Failing Because**:
- AI SDK path might not trigger renders automatically
- Need to check if render is actually triggered
- May need to explicitly trigger render or check render conditions

**Next Steps**:
- Check if render is triggered in AI SDK path
- Verify `responseRenderId` is set
- Check Centrifugo events for render completion

---

### Test 3: upload-chart.spec.ts

**Current Issue**:
- Plan message (`[data-testid="plan-message"]`) not appearing
- AI responds with plan text, but `PlanChatMessage` component never renders

**Root Cause**:
The plan creation flow is asynchronous:

1. **User sends message**: "change the default replicaCount in the values.yaml to 3"
2. **AI SDK streams response**: Plan description text
3. **Streaming completes**: `onFinish` callback fires
4. **Plan created**: Go `/api/plan/create-from-tools` endpoint called
5. **Message updated**: Go sets `response_plan_id` on chat message
6. **Centrifugo event**: `chatmessage-updated` event published
7. **Frontend updates**: `messagesAtom` updated with `responsePlanId`
8. **Component renders**: `PlanChatMessage` appears

**The Problem**:
- Test checks for plan message immediately after sending
- Plan creation happens in `onFinish` (after streaming completes)
- There's a delay between streaming completion and plan appearing
- Test timeout (5 seconds) might be too short

**Fix Applied**:
- ✅ Added wait for plan message
- ✅ Added wait for plan status update
- ✅ Improved proceed button waits

**Still Failing Because**:
- Need to wait for streaming to complete first
- Need to wait for plan creation (Go endpoint call)
- Need to wait for Centrifugo event
- Need to wait for frontend state update

**Next Steps**:
- Increase wait time for plan to appear (30+ seconds)
- Wait for streaming to complete first
- Check for Centrifugo connection
- Add retry logic for plan appearance

---

## Recommended Fixes

### Fix 1: upload-chart.spec.ts - Plan Timing

```typescript
// After sending message, wait for streaming to complete
await page.waitForFunction(() => {
  const messages = document.querySelectorAll('[data-testid="chat-message"]');
  const lastMessage = messages[messages.length - 1];
  // Check if streaming is complete (no "Generating response..." spinner)
  const spinner = lastMessage.querySelector('[data-testid="loading-spinner"]');
  return !spinner;
}, { timeout: 60000 });

// Then wait for plan message to appear (with longer timeout)
await page.waitForSelector('[data-testid="plan-message"]', { timeout: 30000 });
```

### Fix 2: import-artifactory.spec.ts - Render Check

```typescript
// Check if render was triggered
await page.waitForFunction(() => {
  const messages = document.querySelectorAll('[data-testid="chat-message"]');
  const lastMessage = messages[messages.length - 1];
  // Check if message has responseRenderId (via data attribute or content)
  return lastMessage.textContent?.includes('rendering') || 
         lastMessage.querySelector('[data-testid="chart-terminal"]') !== null;
}, { timeout: 30000 });
```

### Fix 3: chat-scrolling.spec.ts - Content Wait

```typescript
// Wait for enough content to make scrolling possible
await page.waitForFunction(() => {
  const container = document.querySelector('[data-testid="scroll-container"]') as HTMLElement;
  if (!container) return false;
  // Need at least 500px of scrollable content
  return container.scrollHeight > container.clientHeight + 500;
}, { timeout: 30000 });
```

---

## Architecture Notes

### Plan Creation Flow (AI SDK Path)

```
User Message
    ↓
AI SDK streamText (intent: "plan")
    ↓
Streaming completes (10-20 seconds)
    ↓
onFinish callback fires
    ↓
createPlanFromToolCalls() called
    ↓
Go /api/plan/create-from-tools (1-2 seconds)
    ↓
Go sets response_plan_id (database update)
    ↓
Centrifugo chatmessage-updated event (real-time)
    ↓
Frontend messagesAtom updated
    ↓
ChatMessage detects responsePlanId
    ↓
PlanChatMessage renders
```

**Total Time**: ~15-25 seconds from message send to plan appearance

### Render Flow (AI SDK Path)

```
User Message: "render this chart"
    ↓
AI SDK streamText
    ↓
Streaming completes
    ↓
??? Render triggered ???
    ↓
response_render_id set
    ↓
Terminal appears
```

**Issue**: AI SDK path might not trigger renders automatically. Need to check if render is triggered or if it needs to be explicit.

---

## Next Steps

1. **Increase timeouts** for all three tests (30-60 seconds)
2. **Add proper waits** for streaming completion
3. **Add retry logic** for plan/render appearance
4. **Check Centrifugo** connection status
5. **Verify AI SDK path** actually triggers renders

---

**Status**: Fixes identified, ready to implement longer waits and better timing checks.

