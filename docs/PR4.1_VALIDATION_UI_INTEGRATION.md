# PR4.1: Validation Results UI Integration (Option A)

## Overview

**Goal**: Display the `ValidationResults` component when `validateChart` tool is called, without full database persistence.

**Approach**: Extract validation results from tool execution in `onStepFinish`, populate `validationsAtom` directly, and set `responseValidationId` on the message client-side.

**Effort**: ~2-3 hours  
**Risk**: Low (client-side only, no DB schema changes)  
**Limitation**: Validation results don't persist across page refresh (acceptable for MVP)

---

## Current Gap

```
validateChart tool returns result
        ↓
AI interprets as text ← STOPS HERE
        ↓ (never happens)
ValidationResults component renders
```

**Why**: No code path to:
1. Store result in `validationsAtom`
2. Set `responseValidationId` on message
3. Trigger component render

---

## Solution Architecture

```
validateChart tool returns result
        ↓
onStepFinish detects validateChart result
        ↓
Generate UUID for validationId
        ↓
Store in validationsAtom via handleValidationUpdatedAtom
        ↓
Set responseValidationId on message (via callback)
        ↓
ValidationResults component renders
```

---

## Implementation Plan

### Phase 1: Extend useAISDKChatAdapter

**File**: `chartsmith/chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Changes**:

1. Import validation atoms:
```typescript
import { handleValidationUpdatedAtom, ValidationData } from '@/atoms/validationAtoms';
```

2. Add atom setter in hook:
```typescript
const [, handleValidationUpdated] = useAtom(handleValidationUpdatedAtom);
```

3. Add state to track validation IDs for current message:
```typescript
const pendingValidationIdRef = useRef<string | null>(null);
```

4. Add callback to process validation results:
```typescript
const processValidationResult = useCallback((
  toolName: string,
  toolResult: unknown,
  messageId: string
) => {
  if (toolName === 'validateChart' && toolResult) {
    // Check if result has validation structure
    const result = toolResult as { validation?: ValidationResult };
    if (result.validation) {
      const validationId = crypto.randomUUID();
      
      // Store in atoms
      handleValidationUpdated({
        id: validationId,
        result: result.validation,
        workspaceId,
        timestamp: new Date(),
      });
      
      // Track for message update
      pendingValidationIdRef.current = validationId;
    }
  }
}, [handleValidationUpdated, workspaceId]);
```

5. Modify `updateChatMessageResponseAction` call to include `responseValidationId`:
```typescript
// In the useEffect that handles status === 'ready'
updateChatMessageResponseAction(
  session,
  currentChatMessageId,
  textContent,
  true,
  followupActions,
  pendingValidationIdRef.current // NEW: pass validation ID
).then(() => {
  pendingValidationIdRef.current = null;
  // ... rest of handler
});
```

---

### Phase 2: Extend updateChatMessageResponseAction

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/update-chat-message-response.ts`

**Current signature**:
```typescript
export async function updateChatMessageResponseAction(
  session: Session,
  chatMessageId: string,
  response: string,
  isIntentComplete: boolean,
  followupActions?: RawFollowupAction[]
): Promise<void>
```

**New signature**:
```typescript
export async function updateChatMessageResponseAction(
  session: Session,
  chatMessageId: string,
  response: string,
  isIntentComplete: boolean,
  followupActions?: RawFollowupAction[],
  responseValidationId?: string  // NEW
): Promise<void>
```

**SQL Update**:
```sql
UPDATE workspace_chat 
SET 
  response = $1,
  is_intent_complete = $2,
  followup_actions = $3,
  response_validation_id = $4  -- NEW (nullable column)
WHERE id = $5
```

---

### Phase 3: Add DB Column (Optional but Recommended)

**File**: New migration or inline ALTER

```sql
ALTER TABLE workspace_chat 
ADD COLUMN IF NOT EXISTS response_validation_id TEXT;
```

**Why**: Even though we don't persist full validation results, storing the ID on the message allows:
- Component to render after page refresh (with empty/loading state)
- Future upgrade path to full persistence

---

### Phase 4: Handle Tool Results in Adapter

**Option A**: Use `onStepFinish` in `useChat` config

The `useChat` hook doesn't expose `onStepFinish` directly. Instead, we need to detect validation results from the streamed parts.

**Better Approach**: Monitor `aiMessages` for tool results

```typescript
// In useAISDKChatAdapter, add useEffect to detect validation results
useEffect(() => {
  if (status !== 'ready' || !currentChatMessageId) return;
  
  const lastAssistant = aiMessages.filter(m => m.role === 'assistant').pop();
  if (!lastAssistant?.parts) return;
  
  // Look for validateChart tool results
  for (const part of lastAssistant.parts) {
    if (part.type === 'tool-result') {
      const toolPart = part as unknown as {
        toolName?: string;
        result?: { validation?: ValidationResult };
      };
      
      if (toolPart.toolName === 'validateChart' && toolPart.result?.validation) {
        processValidationResult(
          'validateChart',
          toolPart.result,
          currentChatMessageId
        );
        break; // Only process first validation result
      }
    }
  }
}, [status, aiMessages, currentChatMessageId, processValidationResult]);
```

---

### Phase 5: Update Message Mapper (if needed)

**File**: `chartsmith/chartsmith-app/lib/chat/messageMapper.ts`

Check if `responseValidationId` is being preserved during message merging:

```typescript
// In mergeMessages function, ensure responseValidationId is preserved
if (matchingStreaming && matchingStreaming.response) {
  return {
    ...histMsg,
    response: matchingStreaming.response,
    isComplete: matchingStreaming.isComplete,
    isIntentComplete: matchingStreaming.isIntentComplete,
    isCanceled: matchingStreaming.isCanceled,
    responseValidationId: histMsg.responseValidationId, // PRESERVE from DB
  };
}
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `hooks/useAISDKChatAdapter.ts` | Add validation detection, atom updates, callback |
| `lib/workspace/actions/update-chat-message-response.ts` | Add `responseValidationId` parameter |
| `lib/chat/messageMapper.ts` | Preserve `responseValidationId` in merge |
| DB migration (optional) | Add `response_validation_id` column |

---

## Testing Plan

### Manual Test Steps

1. **Start fresh workspace**
   - Create new workspace with a chart

2. **Trigger validation**
   - Type: "Validate my chart"
   - Observe AI calling validateChart tool

3. **Verify component renders**
   - Should see `ValidationResults` component with:
     - Overall status badge (PASS/WARNING/FAIL)
     - Issue list with severity icons
     - kube-score summary (if available)
     - "Show details" expandable section

4. **Verify interactivity**
   - Click issues to expand details
   - Click "Show details" for full JSON

5. **Page refresh behavior**
   - Refresh page
   - Validation component should show "Loading..." (no DB persistence)
   - This is acceptable for Option A

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Validation passes | Green badge, "All checks passed!" |
| Validation fails | Red badge, issue list |
| kube-score not installed | Component renders without kube-score section |
| Network error | Tool returns error, AI explains in text |

---

## Rollout

1. **Implement changes** (2-3 hours)
2. **Manual testing** (30 min)
3. **Deploy** (no migrations required if skipping DB column)
4. **Monitor** for errors

---

## Future Upgrade Path (Option B)

If persistence is needed later:

1. Create `workspace_validation` table
2. Create Go endpoint `POST /api/validation/create`
3. Add Centrifugo event `validation-updated`
4. Populate atoms from DB on page load
5. Remove client-side UUID generation

---

## Success Criteria

- [ ] `validateChart` tool execution shows `ValidationResults` component
- [ ] Status badge shows correct color (green/yellow/red)
- [ ] Issues display with severity icons
- [ ] kube-score summary appears when available
- [ ] Expand/collapse works on issues
- [ ] "Show details" shows full JSON
- [ ] No console errors

---

*Document created: December 7, 2025*

