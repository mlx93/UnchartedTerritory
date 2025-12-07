# Spinner/Status Update Bug - Investigation Needed

**Date**: 2025-12-06
**Status**: Bug - Needs Investigation

## Problem

When executing a plan via `executeViaAISDK`, the file list appears correctly with "pending" status (dashed circles), but:
1. Files are NOT transitioning to "creating" status (no spinner shown)
2. Files are NOT transitioning to "created" status (no green checkmarks)

The files stay in "pending" state throughout execution even though the actual file modifications are happening successfully.

## Error Context

Console shows:
```
[executeViaAISDK] Published pending file list to UI
[AI Provider] Using OpenRouter (primary) for model: anthropic/claude-sonnet-4
[executeViaAISDK] Starting execution with AI SDK...
[executeViaAISDK] Step finished with toolResults: 1
```

Note: `onStepFinish` is firing with `toolResults: 1`, but the status updates aren't happening.

## Suspected Issues

### 1. `onChunk` callback - args structure

The `onChunk` callback attempts to read `chunk.args` but:
- `chunk.args` may be undefined during streaming
- The structure of `chunk` for `tool-call` type may not have `args` directly

```typescript
onChunk: async ({ chunk }) => {
  if (chunk.type === 'tool-call' && chunk.toolName === 'textEditor') {
    const args = chunk.args as { path?: string; command?: string } | undefined;
    // args?.path may never be truthy if args structure is different
  }
}
```

### 2. `onStepFinish` callback - toolResults structure

The `onStepFinish` callback checks for `'args' in toolResult` but:
- AI SDK v5 `toolResults` may use a different property name (e.g., `input` instead of `args`)
- The structure needs to be verified against actual AI SDK v5 types

```typescript
onStepFinish: async ({ toolResults }) => {
  for (const toolResult of toolResults ?? []) {
    if (toolResult.toolName === 'textEditor' && 'args' in toolResult) {
      // This condition may never be true if property is named differently
    }
  }
}
```

## Files Involved

- `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts` - Lines 265-308
- Callbacks: `onChunk` (line 272) and `onStepFinish` (line 290)

## Investigation Needed

1. Log the actual structure of `chunk` in `onChunk` callback when `chunk.type === 'tool-call'`
2. Log the actual structure of `toolResult` in `onStepFinish` callback
3. Check AI SDK v5 TypeScript types for correct property names
4. Verify if `onChunk` fires at the right time (before tool execution) or if it's too early (args not yet available)

## Expected Behavior

1. When tool call is detected → file should show spinner ("creating")
2. When tool execution completes → file should show checkmark ("created")
3. Only one file should have spinner at a time
