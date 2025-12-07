# Plan/Execute Architecture Bugs

**Date**: 2025-12-07
**Status**: Tracking

---

## Bug 1: 2nd message appears on top after page refresh

**Severity**: Medium
**Scope**: PR4.2 or separate fix

**Likely Cause**: After page refresh, the `useAISDKChatAdapter` auto-send logic (lines 164-194) might be triggering again for the initial message if the response wasn't fully persisted to the database, creating a duplicate streaming message that gets merged incorrectly.

**Fix Location**: `hooks/useAISDKChatAdapter.ts` - need better detection of whether a message already has a response in DB before auto-sending.

---

## Bug 2 & 3: Files stuck in 'creating' status (deployment.yaml, ingress.yaml never complete)

**Severity**: High
**Scope**: PR4.2

**Root Cause**: In `lib/workspace/actions/execute-via-ai-sdk.ts`, line 290 uses the wrong property name:

```typescript
// Line 290 - WRONG:
const input = toolResult.input as { path?: string; command?: string };

// Should be:
const args = toolResult.args as { path?: string; command?: string };
```

AI SDK v5's `onStepFinish` callback provides `toolResult.args` (not `input`), so the file path is undefined and the status update never happens.

**Fix**:
```typescript
// execute-via-ai-sdk.ts Lines 290-295
if (toolResult.toolName === 'textEditor') {
    // AI SDK v5 uses 'args' for tool result parameters
    const args = toolResult.args as { path?: string; command?: string };
    console.log('[executeViaAISDK] textEditor completed:', args.command, args.path);
    if (args.path && (args.command === 'create' || args.command === 'str_replace')) {
```

**Note**: If PR4.2 Option B is implemented (eliminating Path B entirely), this code becomes dead code and the bug is resolved by deletion.

---

## Bug 4: Initial plan shows hypothetical file list that may not match execution

**Severity**: Medium
**Scope**: PR4.2

**Root Cause**: `extractExpectedFilesViaLLM` tries to predict files upfront (lines 227-239). This is by design to show a preview, but:
- The prediction might not match actual execution order
- When execution starts, `onChunk` marks files as 'creating' but due to Bug 3, they never transition to 'created'

**Resolution**: PR4.2 Option B eliminates this by buffering tool calls during plan generation, making the file list exact.

---

## Bug 5: str_replace tool calls missing oldStr parameter

**Severity**: High
**Scope**: Separate fix (not 4.1 or 4.2)

**Observed**: 2025-12-07

**Symptoms**:
```
[proceedPlanAction] Failed to execute tool call toolu_vrtx_01Tcb6UCPL5v3V8AnJZoJNZ3:
Error: oldStr is required for str_replace command
```

**Context**: User said "validate my chart". Intent classified as `isRender: true`, routed to `ai-sdk` path with tools enabled. AI called `validateChart`, then attempted to fix issues using `str_replace` but emitted malformed tool calls without `oldStr`.

**Interesting Observation**: After page refresh, this doesn't happen on 2nd command - suggests state-dependent path selection issue.

**Root Cause Hypothesis**:
1. AI is generating invalid tool call schema (missing required `oldStr` param)
2. Buffered tools don't validate args before buffering
3. Execution fails when Go backend validates the args

**Potential Fixes**:

Option A - Validate in bufferedTools.ts before buffering:
```typescript
if (params.command === 'str_replace' && !params.oldStr) {
  return {
    success: false,
    message: 'str_replace requires oldStr parameter - please specify the exact text to replace',
  };
}
```

Option B - Improve prompt to prevent malformed calls:
```
When using str_replace, you MUST provide both oldStr (the exact text to find) and newStr (the replacement text). Never call str_replace without oldStr.
```

Option C - Both (defense in depth)

**Why refresh fixes it**: After refresh, the message may have a response persisted, so it doesn't trigger the same code path. Or the AI gets different context on retry.

---

## Bug 6: Render intent routes to ai-sdk but creates plan unexpectedly

**Severity**: Low
**Scope**: Investigate

**Observed**: In Bug 5 logs, `isRender: true` routes to `ai-sdk` path, but stream finishes with `bufferedToolCallCount: 3` and creates a plan.

**Question**: Should render intent create plans? Or should it execute immediately without plan creation?

**Current Behavior**:
```
route: { type: 'render' }
...
[/api/chat] Render intent detected, passing to AI SDK
...
[/api/chat] Created plan with tool calls: bef9e1adb33d
```

**Expected Behavior**: TBD - need to clarify if render requests should go through plan/proceed flow or execute immediately.

---

## Bug 7: "Validate my chart" triggers circular loop of tool calls after refresh

**Severity**: High
**Scope**: Investigate / Prompt issue

**Observed**: 2025-12-07

**Symptoms**: After page refresh, when user says "validate my chart", the AI enters a circular loop of tool calls instead of simply calling `validateChart` and reporting results.

**Context**:
- First command (before refresh): Hits Bug #5 (malformed str_replace)
- After refresh: Avoids Bug #5 but enters circular tool call loop

**Root Cause Hypothesis**:
1. AI calls `validateChart` → gets validation errors
2. AI tries to fix errors by calling `textEditor` (str_replace or create)
3. AI calls `validateChart` again to verify fix
4. Loop continues

**The Real Issue**: The AI shouldn't be trying to fix validation errors automatically. "Validate my chart" should:
1. Call `validateChart` tool
2. Report results to user
3. STOP (don't try to fix anything)

**Potential Fixes**:

Option A - Improve prompt to prevent auto-fixing:
```
When the user asks to validate their chart, ONLY call validateChart and report the results.
Do NOT attempt to fix any issues automatically. Let the user decide what to fix.
```

Option B - Don't include `textEditor` tools when `isRender: true`:
```typescript
// In route.ts, for render intent, use read-only tools only
const tools = isRenderIntent
  ? createReadOnlyTools(...)  // validateChart, getChartContext, etc. - NO textEditor
  : createBufferedTools(...);
```

Option C - Set lower `stepCountIs()` limit for validation requests

**Related**: This is related to Bug #6 - render intent probably shouldn't create plans at all, and definitely shouldn't buffer file-modifying tool calls.

---

## Summary Table

| Bug | Severity | Fixed By | Status |
|-----|----------|----------|--------|
| #1 Duplicate message after refresh | Medium | Separate fix | Open |
| #2/#3 Files stuck in 'creating' | High | PR4.2 (eliminated) | Open |
| #4 Hypothetical file list mismatch | Medium | PR4.2 (eliminated) | Open |
| #5 str_replace missing oldStr | High | Separate fix | Open |
| #6 Render intent creates plan | Low | Investigate | Open |
| #7 Validate triggers circular tool loop | High | Prompt/Intent fix | Open |
| #8 Revision number stuck at 0 | Medium | Investigate | Open |
| #9 Provider switch still uses old code path | High | PR4.2 | Open |
| #10 GPT-4o fails to execute file changes via OpenRouter | High | Investigate | Open |

---

## Bug 9: Switching provider still uses text-only plan path (NO TOOLS)

**Severity**: High
**Scope**: PR4.2 (will be fixed by Option B implementation)

**Observed**: 2025-12-07

**Symptoms**: When user switches to a different model (e.g., openai/gpt-4o) and sends a message, the system still uses the old "NO TOOLS" plan generation path, creating text-only plans that require `executeViaAISDK` (Path B).

**Logs**:
```
[/api/chat] Request received: {
  provider: 'openai',
  model: 'openai/gpt-4o',
  ...
}
[/api/chat] Intent classification result: {
  intent: { isPlan: true, isConversational: false, ... },
  route: { type: 'plan' }
}
[/api/chat] Plan generation phase - NO TOOLS    <-- Still using old path!
...
[/api/chat] Created text-only plan: 21181cd5b0a0
...
[executeViaAISDK] Starting file extraction from plan...   <-- Falls into Path B
```

**Root Cause**: This is the exact behavior PR4.2 Option B is designed to fix. The `route.type === 'plan'` case currently has NO TOOLS, so all plans routed there become text-only plans regardless of provider.

**Why This Matters**:
- Confirms PR4.2 is needed - the bug affects all providers, not just Anthropic
- Shows that `executeViaAISDK` (Path B) is being actively triggered
- The file prediction via `extractExpectedFilesViaLLM` will run, potentially causing mismatch

**Resolution**: PR4.2 Phase 2 will change the plan route to use buffered tools, eliminating this issue.

**Note**: This is not a provider-specific bug - it's the same underlying architecture issue that PR4.2 addresses. The provider switch just makes it more visible because it's a fresh request path.

---

## Bug 10: GPT-4o fails to execute file changes via OpenRouter

**Severity**: High
**Scope**: Investigate - possibly OpenRouter/model compatibility issue

**Observed**: 2025-12-07

**Symptoms**: When switching to GPT-4o (`openai/gpt-4o`) via OpenRouter, file changes fail to execute. The plan is created (as text-only per Bug #9), but execution via `executeViaAISDK` doesn't complete file operations.

**Context from logs**:
```
[/api/chat] Request received: {
  provider: 'openai',
  model: 'openai/gpt-4o',
  ...
}
[/api/chat] Plan generation phase - NO TOOLS
[/api/chat] Created text-only plan: 21181cd5b0a0
[executeViaAISDK] Starting file extraction from plan...
```

**Root Cause Hypotheses**:

1. **Path B execution failure**: `executeViaAISDK` may have issues with GPT-4o's tool call format
2. **OpenRouter tool compatibility**: GPT-4o via OpenRouter may handle tool schemas differently than Anthropic models
3. **Tool call format mismatch**: GPT-4o may emit tool calls in a different format that doesn't match expected schema
4. **Related to Bug #9**: Since it's using Path B (text-only → prediction → execution), all Path B bugs apply

**Investigation Points**:
- Check if `extractExpectedFilesViaLLM` works with GPT-4o
- Check if `streamText` with tools works with GPT-4o via OpenRouter
- Check for any errors in the execution phase logs after "Starting file extraction"
- Test if GPT-4o works when routed through `ai-sdk` path (which has tools enabled)

**Potential Fixes**:
- PR4.2 may fix this by ensuring tools are always enabled during plan generation
- If GPT-4o tool format differs, may need model-specific handling in bufferedTools
- May need to verify OpenRouter's tool call compatibility for OpenAI models

**Related**: Bug #9 (same root path issue), PR4.2 (unified architecture should help)

---

## Bug 8: Revision number stuck at 0 for subsequent edits

**Severity**: Medium
**Scope**: Investigate

**Observed**: 2025-12-07

**Symptoms**: Go backend logs show `revisionNumber=0` for all text editor requests, even after multiple edits:
```
2025-12-07T14:52:49.716-06:00 DEBUG Text editor request
  command=view workspaceId=DOi12kZQysEf path=templates/deployment.yaml revisionNumber=0
```

**Expected Behavior**: After first chart creation, `revisionNumber` should increment (1, 2, 3...) for subsequent edits.

**Root Cause Hypothesis**:
1. Frontend is not incrementing `revisionNumber` after plan is applied
2. Or `revisionNumber` is being read from stale state
3. Or the workspace atom isn't being updated after plan completion

**Impact**:
- File operations may be reading/writing to wrong revision
- Could cause stale data issues or overwrites

**Investigation Points**:
- Check `PlanChatMessage.tsx` line 163: `const revisionNumber = workspaceToUse?.currentRevisionNumber ?? 0`
- Check if `workspaceAtom` is updated after `proceedPlanAction` completes
- Check if Go backend increments revision number on plan application
