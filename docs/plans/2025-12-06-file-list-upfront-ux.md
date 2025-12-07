# File List Upfront UX - Implementation Status

**Date**: 2025-12-06
**Status**: Implemented

## Goal

When user clicks "Create Chart", the full file list should appear FIRST with "pending" status, THEN as each file is worked on it gets a spinner, THEN checkmark when done.

This mirrors the Go implementation in `main-testing-fixes` branch where:
1. `execute-plan.go` discovers files via LLM streaming `<chartsmithActionPlan>` tags
2. `apply-plan.go` processes files one at a time, marking `creating` BEFORE processing and `created` AFTER

## Expected UX Flow

1. User clicks "Create Chart"
2. Button shows "Creating..." spinner
3. Plan status changes to `applying`
4. UI shows "selecting files..." briefly
5. Full list of ALL files appears with pending status (before any file modifications)
6. Current file being worked on shows spinner (only one at a time)
7. Each file gets a checkmark when done
8. Files are ordered by planned execution order

## Implementation Details

### 1. LLM-based File Extraction (`execute-via-ai-sdk.ts`)

**Location**: `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts:24-57`

Uses `generateText` to intelligently extract file paths from plan description:
- **Generic prompt** - not Helm-specific, works for any project type
- Instructs LLM to return files in logical creation order
- Parses JSON array response
- Falls back to regex-based extraction if LLM fails

```typescript
async function extractExpectedFilesViaLLM(
  planDescription: string,
  provider?: string
): Promise<string[]>
```

**Key design decision**: The prompt is intentionally generic so it works for:
- Helm charts
- Kubernetes manifests
- Any other file structure the plan describes

### 2. Spinner Timing Fix - onChunk + onStepFinish

**Location**: `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts:272-307`

The key insight from researching `main-testing-fixes`:
- Go marks files as `creating` BEFORE processing (apply-plan.go:110-128)
- Go marks files as `created` AFTER processing (apply-plan.go:339-356)

We achieve this using two AI SDK callbacks:

```typescript
// onChunk - fires as stream is received, BEFORE tool execution
onChunk: async ({ chunk }) => {
  if (chunk.type === 'tool-call' && chunk.toolName === 'textEditor') {
    // Mark file as "creating" (spinner) BEFORE execution
    await addOrUpdateActionFile(workspaceId, planId, args.path, fileAction, 'creating');
  }
}

// onStepFinish - fires AFTER tool execution completes
onStepFinish: async ({ toolResults }) => {
  // Mark file as "created" (checkmark) AFTER execution
  await addOrUpdateActionFile(workspaceId, planId, args.path, fileAction, 'created');
}
```

**Why this works**:
- `onChunk` receives `tool-call` events as the model generates them
- The tool `execute` function runs after `onChunk` but before `onStepFinish`
- This creates a time window where the file shows a spinner

### 3. Pending File Visual Styling (`PlanChatMessage.tsx`)

**Location**: `chartsmith-app/components/PlanChatMessage.tsx:308-348`

| Status | Icon | Border | Text Color |
|--------|------|--------|------------|
| `pending` | Dashed circle | Dashed border | Gray (muted) |
| `creating` | Spinning circle | Solid | Normal |
| `created` | Green checkmark | Solid | Normal |

```tsx
{action.status === 'pending' ? (
  <div className="rounded-full h-3 w-3 border border-primary/30 border-dashed" />
) : action.status === 'creating' ? (
  <div className="rounded-full h-3 w-3 border border-primary/70 border-t-transparent"
       style={{ animation: 'spin 1s linear infinite' }} />
) : action.status === 'created' ? (
  <div className="text-green-500">
    <svg className="h-3 w-3">✓</svg>
  </div>
) : ...}
```

## Files Modified

| File | Changes |
|------|---------|
| `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts` | LLM extraction, active file tracking, ordered file list |
| `chartsmith-app/components/PlanChatMessage.tsx` | Pending file visual styling |

## Execution Flow

```
User clicks "Create Chart"
         │
         ▼
┌─────────────────────────────────────┐
│ 1. Update plan status to 'applying' │
│    → UI shows "Creating..." button  │
│    → UI shows "selecting files..."  │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ 2. Extract files via LLM            │
│    → Parse plan description         │
│    → Return ordered file list       │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ 3. Add all files as "pending"       │
│    → Each file added to actionFiles │
│    → Publish update to UI           │
│    → UI shows full file list        │
│    → All files have dashed circles  │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ 4. Execute via AI SDK               │
│    For each file:                   │
│    ├─ Mark as "creating" (spinner)  │
│    ├─ Execute textEditor tool       │
│    ├─ Mark as "created" (checkmark) │
│    └─ Publish update to UI          │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ 5. Update plan status to 'applied'  │
│    → All files show checkmarks      │
└─────────────────────────────────────┘
```

## Comparison with Go Implementation (main-testing-fixes)

| Aspect | Go (main-testing-fixes) | TypeScript (AI SDK) |
|--------|------------------------|---------------------|
| File discovery | LLM streams `<chartsmithActionPlan>` XML tags | LLM extracts via `generateText` upfront |
| Creating status | Set in `apply-plan.go:110-128` BEFORE `processActionFile` | Set in `onChunk` callback BEFORE tool execution |
| Created status | Set in `apply-plan.go:339-356` AFTER processing | Set in `onStepFinish` callback AFTER tool execution |
| Realtime updates | Via `realtime.SendEvent` | Via `callGoEndpoint("/api/plan/publish-update")` |

## Key Design Decisions

1. **Generic LLM prompt for file extraction**: Not Helm-specific, so it works for any project type (Helm charts, K8s manifests, etc.)

2. **Separate callbacks for creating/created**: Using `onChunk` (before) and `onStepFinish` (after) mirrors Go's two-phase approach

3. **Single active file**: Only one file shows a spinner at a time, making it clear which file is currently being modified

4. **Visual distinction for pending**: Dashed borders and muted colors make it clear which files haven't been touched yet

## Testing

To verify the implementation works:
1. Create a new workspace
2. Ask to create a Helm chart (e.g., "Create a Helm chart for nginx")
3. Wait for plan to appear
4. Click "Create Chart"
5. Observe:
   - "selecting files..." appears briefly
   - Full file list appears with dashed circles
   - One file at a time gets a spinner
   - Files get checkmarks as completed
   - Files are in logical order (Chart.yaml first)
