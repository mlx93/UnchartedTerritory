# PR3.1: AI SDK UX Improvements

**Date**: 2025-12-06
**Status**: Implemented
**Related**: PR3.0 Buffered Tool Calls Fix

## Overview

This document describes the UX improvements made to the AI SDK plan flow after the PR3.0 backend fixes. These changes ensure a smooth user experience when creating workspaces and executing plans via the AI SDK path.

## Issues Addressed

### Issue 1: Response Text Overwritten
**Problem**: AI streaming responses were replaced with "doing the render now..." text
**Root Cause**: `useCentrifugo.ts:134` conditionally replaced response when `responseRenderId` existed
**Fix**: Removed the conditional replacement, always use actual response

### Issue 2: Terminal Block Appearing Incorrectly
**Problem**: The Terminal component showing helm commands appeared during AI SDK plan creation
**Root Cause**: Terminal shown when `responseRenderId && !isAutorender`, but AI SDK path set both
**Fix**: Added `!message?.responsePlanId && message?.isComplete` checks to Terminal conditional

### Issue 3: Auto-Render Triggered Too Early
**Problem**: Workspace creation triggered an auto-render before AI SDK started streaming
**Root Cause**: `workspace.ts:40` hardcoded `shouldEnqueueRender = true`
**Fix**: Skip auto-render for AI SDK path: `shouldEnqueueRender = knownIntent !== ChatMessageIntent.NON_PLAN`

### Issue 4: AI Streaming Text Not Displaying Live
**Problem**: Streaming AI response only visible after page refresh
**Root Cause**: `ChatMessage` read from Jotai atom (database) instead of merged streaming messages
**Fix**: Added `messageOverride` prop to `ChatMessage`, passed from `ChatContainer` during AI SDK mode

### Issue 5: Proceed Button Not Disappearing
**Problem**: Clicking Proceed didn't update the UI - button remained visible
**Root Cause**: Plan status updated in database but no Centrifugo events published
**Fix**: Added `/api/plan/publish-update` Go endpoint, called after each status change in `proceedPlanAction`

### Issue 6: Duplicate Loading Indicators
**Problem**: Both "thinking..." (in message) and "Generating response..." (at bottom) showed simultaneously
**Root Cause**: Legacy indicator showed when `!isIntentComplete`, even if response was streaming
**Fix**: Added `&& !message.response` to the condition in both `ChatMessage.tsx` and `NewChartChatMessage.tsx`

## Files Modified

### TypeScript (Frontend)

| File | Changes |
|------|---------|
| `hooks/useCentrifugo.ts` | Line 134: Removed response overwrite logic |
| `components/ChatMessage.tsx` | Lines 38-39, 79, 91-93: Added `messageOverride` prop; Line 178: Added Terminal visibility checks; Line 243: Fixed duplicate indicator |
| `components/ChatContainer.tsx` | Line 142: Pass `messageOverride` for AI SDK mode |
| `components/NewChartChatMessage.tsx` | Line 217: Fixed duplicate indicator |
| `lib/workspace/workspace.ts` | Lines 40-41: Skip auto-render for AI SDK path |
| `lib/workspace/actions/proceed-plan.ts` | Lines 30-45: Added `publishPlanUpdate` helper; Lines 110, 168, 181: Call after status changes |

### Go (Backend)

| File | Changes |
|------|---------|
| `pkg/api/handlers/plan.go` | Lines 260-289: Added `PublishPlanUpdate` HTTP handler |
| `pkg/api/server.go` | Line 32: Registered `/api/plan/publish-update` endpoint |

## Architecture Changes

### Before (Problematic Flow)
```
User sends prompt
    ↓
createWorkspace() → shouldEnqueueRender = true
    ↓
renderWorkspace() → sets responseRenderId ← PROBLEM: Too early!
    ↓
Centrifugo: chatmessage-updated
    ↓
useCentrifugo: response = "doing the render now..." ← PROBLEM: Overwrites AI response!
    ↓
ChatMessage: shows Terminal ← PROBLEM: Irrelevant for AI SDK!
    ↓
AI SDK streams (response lost)
    ↓
Plan appears (delayed, no context)
```

### After (Fixed Flow)
```
User sends prompt
    ↓
createWorkspace() → shouldEnqueueRender = false (AI SDK path)
    ↓
AI SDK streams response → visible in real-time via messageOverride
    ↓
onFinish: createPlanFromToolCalls()
    ↓
Plan appears with Proceed button
    ↓
User clicks Proceed
    ↓
proceedPlanAction:
  1. Status → 'applying' + publishPlanUpdate() → Button disappears
  2. Execute tool calls
  3. Status → 'applied' + publishPlanUpdate() → Shows completion
```

## New API Endpoint

### POST /api/plan/publish-update

Publishes a plan update event via Centrifugo to notify the frontend of status changes.

**Request**:
```json
{
  "workspaceId": "string",
  "planId": "string"
}
```

**Response**:
```json
{
  "success": true
}
```

**Usage**: Called from TypeScript `proceedPlanAction` after each plan status change (applying, applied, review on failure).

## Testing Checklist

- [x] Create new workspace with prompt → AI response streams live
- [x] No Terminal block appears during AI SDK flow
- [x] Plan appears with Proceed button after streaming
- [x] No "doing the render now..." text appears
- [x] Only one loading indicator shows during streaming
- [x] Clicking Proceed → button disappears immediately
- [x] Plan status updates reflect in UI via Centrifugo

## Known Limitations / Future Work

### File-by-File Progress Updates
Currently, when Proceed is clicked, all files change instantly without individual progress indicators. To show file-by-file progress:

1. Update `workspace_plan_action_file.status` to 'creating' before each tool call
2. Update to 'created' after success
3. Publish plan update after each file change
4. Frontend already handles these statuses (shows spinner for 'creating', checkmark for 'created')

This would require modifications to:
- `proceedPlanAction` to update action file status in DB
- Possibly add a new endpoint for action file status updates
- Ensure Centrifugo events include updated action files

## Related Documentation

- `docs/PR3.0_BUFFERED_TOOL_CALLS_FIX.md` - Backend fixes for buffered tool calls
- `docs/issues/PR3.0_REMAINING_ISSUES_ANALYSIS.md` - Initial issue analysis
- `thoughts/shared/research/2025-12-06-pr3-post-fix-issues-comprehensive.md` - Detailed research

---

*Document created: 2025-12-06*
