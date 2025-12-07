# PR3.4: Text-Only Plan Creation Fix

## Problem

When sending plan-like requests (e.g., "change the default replicaCount in the values.yaml to 3") via the AI SDK path, the plan record was not being created. This happened because:

1. Groq's intent classifier marked the request as both `isPlan: true` AND `isConversational: true`
2. The routing logic sent it to the `ai-sdk` path (due to `isConversational`)
3. The AI model responded with text but didn't emit tool calls
4. No plan record was created because `bufferedToolCalls.length === 0`
5. The `PlanChatMessage` component never rendered (no Proceed/Ignore buttons)

## Solution

Track the original intent classification and create a text-only plan as a fallback when the model doesn't emit tool calls.

## Changes Made

### File: `chartsmith/chartsmith-app/app/api/chat/route.ts`

#### 1. Added `wasIntentPlan` tracking variable (line 133-135)

```typescript
// PR3.4: Track if original intent was a plan request (even if routed to ai-sdk)
// This allows creating text-only plans when model doesn't emit tool calls
let wasIntentPlan = false;
```

#### 2. Set `wasIntentPlan` from intent classification (line 182-183)

```typescript
// PR3.4: Track if this was a plan intent for fallback text-only plan creation
wasIntentPlan = intent.isPlan;
```

#### 3. Updated `onFinish` callback in ai-sdk path (lines 285-330)

Now creates a text-only plan when:
- `wasIntentPlan` is `true` (intent was classified as a plan request)
- `bufferedToolCalls.length === 0` (model didn't emit tool calls)
- `workspaceId && chatMessageId` are present

## How It Fixes the Issue

For the prompt **"change the default replicaCount in the values.yaml to 3"**:

1. ✅ Groq classifies it as `isPlan: true` (possibly also `isConversational: true`)
2. ✅ If routed to `ai-sdk` path (due to `isConversational`), `wasIntentPlan` is set to `true`
3. ✅ When streaming finishes with no tool calls, the fallback creates a text-only plan
4. ✅ `createPlanFromToolCalls` is called with empty tool calls + response text as description
5. ✅ Go backend creates plan record, sets `response_plan_id`, publishes Centrifugo event
6. ✅ Frontend receives event and renders `PlanChatMessage` with Proceed/Ignore buttons

## No Impact on Existing Flows

### "Create a simple nginx deployment"
- Routes to the `plan` branch (`isPlan && !isConversational`)
- Has its own text-only plan creation in `onFinish`
- The new fallback only triggers in the `ai-sdk` path when needed

### Tool-emitting requests
- If the model emits tool calls, they're buffered and used for plan creation
- The fallback only triggers when `bufferedToolCalls.length === 0`

## Test Results

- **Unit Tests**: 116/116 passing ✅
- **E2E Tests**: Revalidating after fix

## Related Files

- `chartsmith/chartsmith-app/app/api/chat/route.ts` - Main API route with fix
- `chartsmith/chartsmith-app/lib/ai/plan.ts` - `createPlanFromToolCalls` function
- `chartsmith/chartsmith-app/components/PlanChatMessage.tsx` - Plan UI component

## Date

December 7, 2025

