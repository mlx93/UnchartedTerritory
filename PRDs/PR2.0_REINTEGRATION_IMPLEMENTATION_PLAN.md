# PR2.0: AI SDK Reintegration - Implementation Plan

**Status**: Ready for Implementation
**Created**: 2025-12-05
**Tech PRD**: `PRDs/PR2.0_REINTEGRATION_Tech_PRD.md`
**Estimated Effort**: 30-40 hours (10-15 days)

---

## Overview

This plan implements **Path B**: integrating the AI SDK transport layer into the existing main workspace path (`/workspace/[id]`), preserving all existing UI components and features while replacing the Go worker chat backend with AI SDK streaming.

### Guiding Principles

1. **Adapter Pattern** - Create a thin layer that converts between AI SDK and existing interfaces
2. **Feature Flag** - Enable instant rollback via environment variable
3. **Keep Existing UI** - No changes to ChatMessage, PlanChatMessage, Terminal, etc.
4. **Incremental PRs** - Small, testable changes that can be reviewed independently

---

## Current State Analysis

### Files We're Keeping (Main Path - No Changes)

| File | Purpose | Lines |
|------|---------|-------|
| `components/ChatMessage.tsx` | Message rendering with plan/render refs | ~350 |
| `components/PlanChatMessage.tsx` | Plan display UI with actions | ~400 |
| `components/RollbackModal.tsx` | Rollback confirmation UI | ~80 |
| `components/Terminal.tsx` | Helm render output display | ~120 |
| `components/CommandMenu.tsx` | Cmd+K quick actions | ~200 |
| `components/ChatContainer.tsx` | Chat input + message list | ~250 (will modify) |
| `atoms/workspace.ts` | Jotai state management | ~280 |
| `hooks/useCentrifugo.ts` | Real-time WebSocket updates | ~650 |

### Files We're Creating

| File | Purpose |
|------|---------|
| `hooks/useAISDKChatAdapter.ts` | Bridge between AI SDK and existing interface |
| `lib/chat/messageMapper.ts` | UIMessage ↔ Message conversion utilities |
| `lib/chat/messageMapper.test.ts` | Unit tests for message mapping |

### Files We're Modifying

| File | Change |
|------|--------|
| `components/ChatContainer.tsx` | Add conditional AI SDK usage via adapter |
| `app/workspace/[id]/page.tsx` | Ensure proper initial data for adapter |

### Files We're Removing (Phase 6 - After Stable)

| File | Reason |
|------|--------|
| `app/test-ai-chat/` | Entire directory - merged into main path |
| `components/chat/AIChat.tsx` | Replaced by adapter |
| `components/chat/AIMessageList.tsx` | Replaced by adapter |

---

## Phase 1: Message Mapper & Adapter Hook

**Duration**: 2-3 days
**Goal**: Create the core infrastructure for message format conversion

### 1.1 Create Message Mapper

**File**: `chartsmith-app/lib/chat/messageMapper.ts`

```typescript
import { type UIMessage } from "ai";
import { type Message } from "@/components/types";

/**
 * Converts AI SDK UIMessage parts to a single response string
 */
export function extractTextFromParts(parts: UIMessage['parts']): string {
  if (!parts || parts.length === 0) return '';

  return parts
    .filter((part): part is { type: 'text'; text: string } => part.type === 'text')
    .map(part => part.text)
    .join('\n');
}

/**
 * Extracts user prompt from UIMessage
 */
export function extractPromptFromUIMessage(message: UIMessage): string {
  if (message.role !== 'user') return '';
  return extractTextFromParts(message.parts);
}

/**
 * Extracts assistant response from UIMessage
 */
export function extractResponseFromUIMessage(message: UIMessage): string {
  if (message.role !== 'assistant') return '';
  return extractTextFromParts(message.parts);
}

/**
 * Checks if UIMessage contains tool invocations
 */
export function hasToolInvocations(message: UIMessage): boolean {
  return message.parts?.some(part =>
    part.type === 'tool-invocation' || part.type === 'tool-result'
  ) ?? false;
}

/**
 * Extracts plan ID from tool results if textEditor created a plan
 */
export function extractPlanIdFromToolResults(message: UIMessage): string | undefined {
  const toolResults = message.parts?.filter(part => part.type === 'tool-result') ?? [];

  for (const result of toolResults) {
    if (result.toolName === 'textEditor' && result.output) {
      const output = result.output as { planId?: string };
      if (output.planId) return output.planId;
    }
  }

  return undefined;
}

/**
 * Converts a single AI SDK UIMessage to our Message format
 */
export function mapUIMessageToMessage(
  uiMessage: UIMessage,
  options: {
    workspaceId?: string;
    revisionNumber?: number;
    isComplete?: boolean;
  } = {}
): Message {
  const isUser = uiMessage.role === 'user';
  const isAssistant = uiMessage.role === 'assistant';

  return {
    id: uiMessage.id,
    prompt: isUser ? extractTextFromParts(uiMessage.parts) : '',
    response: isAssistant ? extractTextFromParts(uiMessage.parts) : undefined,
    isComplete: options.isComplete ?? true,
    isIntentComplete: options.isComplete ?? true,
    isCanceled: false,
    workspaceId: options.workspaceId,
    revisionNumber: options.revisionNumber,
    responsePlanId: isAssistant ? extractPlanIdFromToolResults(uiMessage) : undefined,
    createdAt: new Date(),
  };
}

/**
 * Converts array of AI SDK UIMessages to our Message format
 * Pairs user/assistant messages appropriately
 */
export function mapUIMessagesToMessages(
  uiMessages: UIMessage[],
  options: {
    workspaceId?: string;
    revisionNumber?: number;
    isStreaming?: boolean;
  } = {}
): Message[] {
  const messages: Message[] = [];

  for (let i = 0; i < uiMessages.length; i++) {
    const uiMsg = uiMessages[i];

    if (uiMsg.role === 'user') {
      // Start a new message with user prompt
      const message: Message = {
        id: uiMsg.id,
        prompt: extractTextFromParts(uiMsg.parts),
        response: undefined,
        isComplete: false,
        isIntentComplete: false,
        workspaceId: options.workspaceId,
        revisionNumber: options.revisionNumber,
        createdAt: new Date(),
      };

      // Check if next message is the assistant response
      const nextMsg = uiMessages[i + 1];
      if (nextMsg && nextMsg.role === 'assistant') {
        message.response = extractTextFromParts(nextMsg.parts);
        message.responsePlanId = extractPlanIdFromToolResults(nextMsg);

        // Determine completion status
        const isLastMessage = i + 1 === uiMessages.length - 1;
        message.isComplete = !options.isStreaming || !isLastMessage;
        message.isIntentComplete = message.isComplete;

        i++; // Skip the assistant message in next iteration
      }

      messages.push(message);
    }
  }

  return messages;
}

/**
 * Merges streaming messages with historical messages
 * Historical messages take precedence for shared IDs
 */
export function mergeMessages(
  historicalMessages: Message[],
  streamingMessages: Message[]
): Message[] {
  const historicalIds = new Set(historicalMessages.map(m => m.id));

  // Filter out streaming messages that already exist in history
  const newStreamingMessages = streamingMessages.filter(m => !historicalIds.has(m.id));

  return [...historicalMessages, ...newStreamingMessages];
}
```

### 1.2 Create Unit Tests for Message Mapper

**File**: `chartsmith-app/lib/chat/messageMapper.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import {
  extractTextFromParts,
  extractPromptFromUIMessage,
  extractResponseFromUIMessage,
  hasToolInvocations,
  mapUIMessageToMessage,
  mapUIMessagesToMessages,
  mergeMessages,
} from './messageMapper';
import { type UIMessage } from 'ai';

describe('messageMapper', () => {
  describe('extractTextFromParts', () => {
    it('should extract text from text parts', () => {
      const parts = [
        { type: 'text' as const, text: 'Hello' },
        { type: 'text' as const, text: 'World' },
      ];
      expect(extractTextFromParts(parts)).toBe('Hello\nWorld');
    });

    it('should filter out non-text parts', () => {
      const parts = [
        { type: 'text' as const, text: 'Hello' },
        { type: 'tool-invocation' as const, toolName: 'test', args: {} },
        { type: 'text' as const, text: 'World' },
      ];
      expect(extractTextFromParts(parts)).toBe('Hello\nWorld');
    });

    it('should return empty string for empty parts', () => {
      expect(extractTextFromParts([])).toBe('');
      expect(extractTextFromParts(undefined as any)).toBe('');
    });
  });

  describe('hasToolInvocations', () => {
    it('should detect tool invocations', () => {
      const message: UIMessage = {
        id: '1',
        role: 'assistant',
        parts: [
          { type: 'tool-invocation', toolName: 'textEditor', args: {} },
        ],
      };
      expect(hasToolInvocations(message)).toBe(true);
    });

    it('should return false for text-only messages', () => {
      const message: UIMessage = {
        id: '1',
        role: 'assistant',
        parts: [{ type: 'text', text: 'Hello' }],
      };
      expect(hasToolInvocations(message)).toBe(false);
    });
  });

  describe('mapUIMessagesToMessages', () => {
    it('should pair user and assistant messages', () => {
      const uiMessages: UIMessage[] = [
        { id: '1', role: 'user', parts: [{ type: 'text', text: 'Hello' }] },
        { id: '2', role: 'assistant', parts: [{ type: 'text', text: 'Hi there!' }] },
      ];

      const messages = mapUIMessagesToMessages(uiMessages, { workspaceId: 'ws-1' });

      expect(messages).toHaveLength(1);
      expect(messages[0].prompt).toBe('Hello');
      expect(messages[0].response).toBe('Hi there!');
      expect(messages[0].workspaceId).toBe('ws-1');
    });

    it('should handle streaming (incomplete) messages', () => {
      const uiMessages: UIMessage[] = [
        { id: '1', role: 'user', parts: [{ type: 'text', text: 'Hello' }] },
        { id: '2', role: 'assistant', parts: [{ type: 'text', text: 'Thinking...' }] },
      ];

      const messages = mapUIMessagesToMessages(uiMessages, { isStreaming: true });

      expect(messages).toHaveLength(1);
      expect(messages[0].isComplete).toBe(false);
      expect(messages[0].isIntentComplete).toBe(false);
    });
  });

  describe('mergeMessages', () => {
    it('should not duplicate messages by ID', () => {
      const historical: Message[] = [
        { id: '1', prompt: 'Old', isComplete: true },
      ];
      const streaming: Message[] = [
        { id: '1', prompt: 'New', isComplete: true },
        { id: '2', prompt: 'Also new', isComplete: true },
      ];

      const merged = mergeMessages(historical, streaming);

      expect(merged).toHaveLength(2);
      expect(merged[0].prompt).toBe('Old'); // Historical takes precedence
      expect(merged[1].id).toBe('2');
    });
  });
});
```

### 1.3 Create Adapter Hook

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

```typescript
"use client";

import { useCallback, useEffect, useMemo, useState, useRef } from "react";
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";
import { type Session } from "next-auth";
import { useAtom } from "jotai";

import { type Message } from "@/components/types";
import { messagesAtom } from "@/atoms/workspace";
import { mapUIMessagesToMessages, mergeMessages } from "@/lib/chat/messageMapper";
import { createAISDKChatMessageAction } from "@/lib/workspace/actions/create-ai-sdk-chat-message";
import { updateChatMessageResponseAction } from "@/lib/workspace/actions/update-chat-message-response";
import { STREAMING_THROTTLE_MS, DEFAULT_PROVIDER, DEFAULT_MODEL } from "@/lib/ai/config";

export interface AdaptedChatState {
  /** Messages in existing Message format (historical + streaming) */
  messages: Message[];

  /** Send a message with optional persona - MUST pass persona to API */
  sendMessage: (content: string, persona?: string) => Promise<void>;

  /** Whether currently waiting for response to start (status === 'submitted') */
  isThinking: boolean;

  /** Whether currently streaming a response (status === 'streaming') */
  isStreaming: boolean;

  /** Whether the current message was canceled */
  isCanceled: boolean;

  /** Cancel the current streaming response - sets isCanceled and calls stop() */
  cancel: () => void;

  /** Current error, if any */
  error: Error | null;
}

export function useAISDKChatAdapter(
  workspaceId: string,
  revisionNumber: number,
  session: Session,
  initialMessages: Message[] = []
): AdaptedChatState {
  // Track the current message being persisted
  const [currentChatMessageId, setCurrentChatMessageId] = useState<string | null>(null);
  const hasPersistedRef = useRef<Set<string>>(new Set());

  // Track cancel state (AI SDK doesn't expose this)
  const [isCanceled, setIsCanceled] = useState(false);

  // Jotai messages for real-time updates from Centrifugo
  const [jotaiMessages, setJotaiMessages] = useAtom(messagesAtom);

  // AI SDK chat hook
  const {
    messages: aiMessages,
    sendMessage: aiSendMessage,
    status,
    stop,
    error,
  } = useChat({
    transport: new DefaultChatTransport({ api: "/api/chat" }),
    experimental_throttle: STREAMING_THROTTLE_MS,
    onError: (err) => {
      console.error('[useAISDKChatAdapter] Error:', err);
    },
  });

  // Helper to get fresh body params WITH PERSONA (avoid stale closures)
  // Note: persona is passed per-call via sendMessage, stored in ref
  const currentPersonaRef = useRef<string>('auto');

  const getChatBody = useCallback(() => ({
    provider: DEFAULT_PROVIDER,
    model: DEFAULT_MODEL,
    workspaceId,
    revisionNumber,
    persona: currentPersonaRef.current,  // CRITICAL: Include persona
  }), [workspaceId, revisionNumber]);

  // Convert AI SDK messages to our format
  const streamingMessages = useMemo(() => {
    return mapUIMessagesToMessages(aiMessages, {
      workspaceId,
      revisionNumber,
      isStreaming: status === 'streaming',
    });
  }, [aiMessages, workspaceId, revisionNumber, status]);

  // Merge historical (from Jotai/DB) with streaming messages
  const mergedMessages = useMemo(() => {
    // Jotai messages are authoritative for historical data
    // Streaming messages are for current conversation
    return mergeMessages(jotaiMessages, streamingMessages);
  }, [jotaiMessages, streamingMessages]);

  // Persist response when streaming completes
  useEffect(() => {
    if (status === 'ready' && currentChatMessageId && aiMessages.length > 0) {
      // Avoid double-persistence
      if (hasPersistedRef.current.has(currentChatMessageId)) {
        return;
      }

      const lastAssistant = aiMessages.filter(m => m.role === 'assistant').pop();
      if (lastAssistant) {
        const textContent = lastAssistant.parts
          ?.filter((p): p is { type: 'text'; text: string } => p.type === 'text')
          .map(p => p.text)
          .join('\n') || '';

        hasPersistedRef.current.add(currentChatMessageId);

        updateChatMessageResponseAction(
          session,
          currentChatMessageId,
          textContent,
          true // isIntentComplete
        ).then(() => {
          setCurrentChatMessageId(null);
        }).catch(err => {
          console.error('[useAISDKChatAdapter] Failed to persist response:', err);
          hasPersistedRef.current.delete(currentChatMessageId);
        });
      }
    }
  }, [status, currentChatMessageId, aiMessages, session]);

  // Send message handler - MUST include persona
  const sendMessage = useCallback(async (content: string, persona?: string) => {
    if (!content.trim()) return;

    // Reset cancel state for new message
    setIsCanceled(false);

    // Store persona for getChatBody
    currentPersonaRef.current = persona ?? 'auto';

    try {
      // 1. Persist user message to database (for history)
      const chatMessage = await createAISDKChatMessageAction(
        session,
        workspaceId,
        content.trim()
      );
      setCurrentChatMessageId(chatMessage.id);

      // 2. Send to AI SDK with fresh body params INCLUDING PERSONA
      await aiSendMessage(
        { text: content.trim() },
        { body: getChatBody() }  // getChatBody now includes persona
      );
    } catch (err) {
      console.error('[useAISDKChatAdapter] Send failed:', err);
      throw err;
    }
  }, [session, workspaceId, aiSendMessage, getChatBody]);

  // Cancel handler - sets isCanceled AND calls stop()
  const cancel = useCallback(() => {
    setIsCanceled(true);
    stop();

    // Persist canceled state to database if we have a current message
    if (currentChatMessageId) {
      const lastAssistant = aiMessages.filter(m => m.role === 'assistant').pop();
      const partialResponse = lastAssistant?.parts
        ?.filter((p): p is { type: 'text'; text: string } => p.type === 'text')
        .map(p => p.text)
        .join('\n') || '';

      updateChatMessageResponseAction(
        session,
        currentChatMessageId,
        partialResponse,
        true  // isIntentComplete (message is done, even if canceled)
      ).catch(err => {
        console.error('[useAISDKChatAdapter] Failed to persist canceled message:', err);
      });
    }
  }, [stop, currentChatMessageId, aiMessages, session]);

  return {
    messages: mergedMessages,
    sendMessage,
    isThinking: status === 'submitted',
    isStreaming: status === 'streaming',
    isCanceled,
    cancel,  // Use our wrapped cancel, not raw stop()
    error: error ?? null,
  };
}
```

### 1.4 Additional Unit Tests (Addressing Feedback Gaps)

**Add to**: `chartsmith-app/lib/chat/messageMapper.test.ts`

```typescript
describe('persona propagation', () => {
  it('should include persona in getChatBody', () => {
    // Test that getChatBody includes persona field
    const body = getChatBody();
    expect(body).toHaveProperty('persona');
  });

  it('should update persona ref when sendMessage called', async () => {
    // Mock sendMessage with persona='developer'
    // Verify getChatBody returns persona='developer'
  });
});

describe('cancel state handling', () => {
  it('should set isCanceled when cancel() called', () => {
    // Call cancel()
    // Verify isCanceled is true
  });

  it('should persist partial response on cancel', async () => {
    // Start streaming
    // Call cancel()
    // Verify updateChatMessageResponseAction called with partial content
  });

  it('should reset isCanceled on new message', async () => {
    // Set isCanceled = true
    // Call sendMessage
    // Verify isCanceled is false
  });
});

describe('status flag mapping', () => {
  it('should map submitted to isThinking=true', () => {
    // Mock status='submitted'
    // Verify isThinking=true, isStreaming=false
  });

  it('should map streaming to isStreaming=true', () => {
    // Mock status='streaming'
    // Verify isThinking=false, isStreaming=true
  });

  it('should map ready to both false', () => {
    // Mock status='ready'
    // Verify isThinking=false, isStreaming=false
  });
});
```

### Success Criteria - Phase 1

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd chartsmith-app && npm run typecheck`
- [ ] Unit tests pass: `cd chartsmith-app && npm run test -- messageMapper`
- [ ] Linting passes: `cd chartsmith-app && npm run lint`
- [ ] **NEW**: Persona propagation test passes
- [ ] **NEW**: Cancel state tests pass
- [ ] **NEW**: Status flag mapping tests pass

#### Manual Verification:
- [ ] Message mapper correctly handles edge cases (empty parts, tool results)
- [ ] Adapter hook types are compatible with existing ChatContainer
- [ ] **NEW**: Verify persona is visible in network tab when sending message

**Implementation Note**: After completing Phase 1 and all automated verification passes, pause here for review before proceeding to Phase 2.

---

## Phase 2: ChatContainer Integration

**Duration**: 2-3 days
**Goal**: Integrate adapter into ChatContainer with feature flag

### 2.1 Add Feature Flag Environment Variable

**File**: `chartsmith-app/.env.local` (example)

```bash
# AI SDK Chat Feature Flag
# Set to 'true' to use AI SDK transport instead of Go worker
NEXT_PUBLIC_USE_AI_SDK_CHAT=false
```

**File**: `chartsmith-app/.env.example` (update)

```bash
# AI SDK Chat Feature Flag (optional)
# NEXT_PUBLIC_USE_AI_SDK_CHAT=true
```

### 2.2 Create Legacy Chat Hook Wrapper

**File**: `chartsmith-app/hooks/useLegacyChat.ts`

```typescript
"use client";

import { useCallback } from "react";
import { useAtom } from "jotai";
import { type Session } from "next-auth";

import { type Message } from "@/components/types";
import { messagesAtom, workspaceAtom, isRenderingAtom } from "@/atoms/workspace";
import { createChatMessageAction } from "@/lib/workspace/actions/create-chat-message";
import { type AdaptedChatState } from "./useAISDKChatAdapter";

/**
 * Wraps the legacy Go worker chat path to match AdaptedChatState interface
 */
export function useLegacyChat(session: Session): AdaptedChatState {
  const [messages, setMessages] = useAtom(messagesAtom);
  const [workspace] = useAtom(workspaceAtom);
  const [isRendering] = useAtom(isRenderingAtom);

  const sendMessage = useCallback(async (content: string, persona?: string) => {
    if (!content.trim() || !workspace) return;

    const chatMessage = await createChatMessageAction(
      session,
      workspace.id,
      content.trim(),
      persona ?? 'auto'
    );

    setMessages(prev => [...prev, chatMessage]);
  }, [session, workspace, setMessages]);

  // Legacy path doesn't have explicit cancel - rendering status is managed by Go worker
  const cancel = useCallback(() => {
    // Could implement cancelMessageAction here if needed
    console.warn('[useLegacyChat] Cancel not implemented for legacy path');
  }, []);

  return {
    messages,
    sendMessage,
    isThinking: false, // Legacy path shows thinking via isRendering
    isStreaming: isRendering,
    cancel,
    error: null,
  };
}
```

### 2.3 Modify ChatContainer

**File**: `chartsmith-app/components/ChatContainer.tsx`

**Changes Required:**
1. Import adapter hooks
2. Add feature flag check
3. Use conditional chat state
4. Keep all existing UI unchanged

```diff
// ChatContainer.tsx - Key changes only (pseudocode diff)

+ import { useAISDKChatAdapter } from "@/hooks/useAISDKChatAdapter";
+ import { useLegacyChat } from "@/hooks/useLegacyChat";

  export function ChatContainer({ session }: ChatContainerProps) {
+   // Feature flag check
+   const useAISDK = process.env.NEXT_PUBLIC_USE_AI_SDK_CHAT === 'true';
+
    const [workspace] = useAtom(workspaceAtom);
-   const [messages, setMessages] = useAtom(messagesAtom);
-   const [isRendering] = useAtom(isRenderingAtom);
+   const [initialMessages] = useAtom(messagesAtom);

+   // Conditional chat transport
+   const legacyChat = useLegacyChat(session);
+   const aiSDKChat = useAISDKChatAdapter(
+     workspace?.id ?? '',
+     workspace?.currentRevisionNumber ?? 0,
+     session,
+     initialMessages
+   );
+
+   const chatState = useAISDK ? aiSDKChat : legacyChat;
+
    const [chatInput, setChatInput] = useState("");
    const [selectedRole, setSelectedRole] = useState<"auto" | "developer" | "operator">("auto");

    const handleSubmitChat = async (e: React.FormEvent) => {
      e.preventDefault();
-     if (!chatInput.trim() || isRendering) return;
+     if (!chatInput.trim() || chatState.isStreaming) return;
-     if (!session || !workspace) return;
+     if (!session || !workspace) return;

-     const chatMessage = await createChatMessageAction(session, workspace.id, chatInput.trim(), selectedRole);
-     setMessages(prev => [...prev, chatMessage]);
+     await chatState.sendMessage(chatInput.trim(), selectedRole);
      setChatInput("");
    };

    // ... rest of component remains unchanged
    // ChatMessage components receive messages from chatState.messages
    // isRendering checks become chatState.isStreaming
  }
```

### 2.4 Update ChatContainer Message Rendering

Ensure messages from `chatState.messages` are rendered correctly:

```typescript
// In ChatContainer.tsx render section
{chatState.messages.map((message, index) => (
  <ChatMessage
    key={message.id}
    messageId={message.id}
    session={session}
    showChatInput={index === chatState.messages.length - 1}
  />
))}

{/* Thinking indicator */}
{chatState.isThinking && (
  <div className="flex items-center gap-2 text-muted-foreground">
    <Spinner className="h-4 w-4 animate-spin" />
    <span>Thinking...</span>
  </div>
)}

{/* Streaming indicator */}
{chatState.isStreaming && !chatState.isThinking && (
  <div className="flex items-center gap-2">
    <span className="text-muted-foreground">Generating response...</span>
    <Button variant="ghost" size="sm" onClick={chatState.cancel}>
      Cancel
    </Button>
  </div>
)}
```

### Success Criteria - Phase 2

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith-app && npm run typecheck`
- [ ] Build succeeds: `cd chartsmith-app && npm run build`
- [ ] Lint passes: `cd chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] With `USE_AI_SDK_CHAT=false`: Chat works exactly as before (Go worker)
- [ ] With `USE_AI_SDK_CHAT=true`: Chat sends via AI SDK, messages display
- [ ] Streaming indicator appears during AI response
- [ ] Cancel button stops generation
- [ ] Historical messages load correctly on page refresh

**Implementation Note**: Test both paths thoroughly before proceeding. The feature flag should allow instant rollback.

---

## Phase 3: Plan Workflow & Centrifugo Event Parity

**Duration**: 2-3 days
**Goal**: Document plan workflow differences and ensure Centrifugo events work correctly

### 3.1 DECISION: AI SDK Uses Commit/Discard Workflow

**This is a firm decision for PR2.0, not a recommendation.**

| Aspect | Main Path (Go Worker) | AI SDK Path (PR2.0) |
|--------|----------------------|---------------------|
| Change Review | PlanChatMessage UI | Editor pending indicators |
| Approval Action | "Proceed" button | "Commit" button |
| Rejection Action | "Ignore" button | "Discard" button |
| Plan Records | Created in `workspace_plan` | **NOT created** |
| `responsePlanId` | Set on message | **NOT set** |

**Rationale:**
- Commit/Discard already works (PR1.7)
- Both achieve the same outcome: user reviews before finalizing
- Avoids scope creep and Go endpoint changes
- Can add full Plan UI in future PR if needed

### 3.2 What This Means for UI Components

**PlanChatMessage.tsx Behavior:**
- Will **NOT show** for AI SDK messages (no `responsePlanId`)
- Still works correctly for legacy path messages
- No code changes needed - conditional rendering handles this

**ChatMessage.tsx Behavior:**
- AI SDK messages will show text response only (no plan section)
- Legacy messages continue to show plan section when `responsePlanId` exists

### 3.3 Explicit Non-Goals for PR2.0

The following are **OUT OF SCOPE** for PR2.0:

- [ ] ~~Creating `workspace_plan` records from AI SDK tool calls~~
- [ ] ~~Showing PlanChatMessage UI for AI SDK messages~~
- [ ] ~~Proceed/Ignore buttons for AI SDK messages~~
- [ ] ~~`followupActions` in AI SDK messages~~
- [ ] ~~`responseRollbackToRevisionNumber` in AI SDK messages~~

### 3.4 Centrifugo Event Parity (Critical)

All Centrifugo events must continue working in AI SDK mode:

| Event | Handler | Status | Notes |
|-------|---------|--------|-------|
| `artifact-updated` | `handleArtifactUpdated` | ✅ Works | File updates from tools |
| `plan-updated` | `handlePlanUpdatedAtom` | ✅ Works | No plans created, but handler exists |
| `render-stream` | `handleRenderStreamEvent` | ✅ Works | Render output display |
| `render-file` | Render file handler | ✅ Works | Rendered files list |
| `revision-created` | Refetch workspace | ✅ Works | After Commit action |
| `conversion-file` | `handleConversionFileUpdatedMessage` | ✅ Works | Conversion progress |
| `conversion-status` | `handleConversationUpdatedMessage` | ✅ Works | Conversion status |
| `chatmessage-updated` | `handleChatMessageUpdated` | ⚠️ Needs handling | See below |

**Chat Message Event Handling:**

The `chatmessage-updated` event may conflict with AI SDK streaming. The adapter must coordinate:

```typescript
// In useAISDKChatAdapter.ts - export current message ID for Centrifugo
export const currentStreamingMessageIdAtom = atom<string | null>(null);

// In useCentrifugo.ts - check before updating
const handleChatMessageUpdated = (data: RawChatMessage) => {
  const streamingId = get(currentStreamingMessageIdAtom);

  // Skip Centrifugo update for message currently streaming via AI SDK
  if (streamingId === data.id) {
    console.log('[Centrifugo] Skipping update for streaming message:', data.id);
    return;
  }

  // Normal update for historical messages
  setMessages(prev => mergeOrAppend(prev, data));
};
```

### 3.5 Conversion Event Verification

Ensure K8s-to-Helm conversion events still work:

```typescript
// In useCentrifugo.ts - these handlers must remain active
case 'conversion-file':
  handleConversionFileUpdatedMessage(message.data);
  break;
case 'conversion-status':
  handleConversationUpdatedMessage(message.data);
  break;
```

**Test:** Trigger a K8s conversion in AI SDK mode and verify progress UI updates.

### Success Criteria - Phase 3

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith-app && npm run typecheck`
- [ ] No Centrifugo handler errors in console

#### Manual Verification:
- [ ] AI SDK path: Pending changes visible in editor (yellow dots)
- [ ] AI SDK path: Commit creates new revision
- [ ] AI SDK path: Discard clears pending
- [ ] AI SDK path: Render results display in Terminal component
- [ ] AI SDK path: Conversion progress displays (if applicable)
- [ ] Legacy path: Full plan workflow still works (Proceed/Ignore)
- [ ] No duplicate message updates when switching paths

**Implementation Note**: Document all parity differences in ARCHITECTURE.md.

---

## Phase 4: Testing & Validation

**Duration**: 2-3 days
**Goal**: Comprehensive testing of both chat paths

### 4.1 Integration Test Scenarios

**Create test file**: `chartsmith-app/__tests__/chat-integration.test.ts`

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('Chat Integration', () => {
  describe('Feature Flag', () => {
    it('should use legacy chat when flag is false', () => {
      // Mock env var
      // Verify createChatMessageAction is called
    });

    it('should use AI SDK when flag is true', () => {
      // Mock env var
      // Verify useChat is used
    });
  });

  describe('Message Display', () => {
    it('should display historical messages', () => {
      // Load page with existing messages
      // Verify all messages render
    });

    it('should display streaming messages', () => {
      // Send message
      // Verify streaming indicator
      // Verify final message displays
    });
  });

  describe('Persona Selection', () => {
    it('should pass persona to AI SDK', () => {
      // Select developer persona
      // Send message
      // Verify body params include persona
    });
  });
});
```

### 4.2 E2E Test Scenarios

**Manual Test Checklist:**

| Test Case | Legacy Path | AI SDK Path | Notes |
|-----------|-------------|-------------|-------|
| **Core Chat** | | | |
| Send basic message | [ ] | [ ] | Text appears in chat |
| Streaming display | N/A | [ ] | Response streams in |
| Cancel streaming | [ ] | [ ] | Generation stops, partial saved |
| Cancel → isCanceled UI | [ ] | [ ] | Shows "canceled" message |
| Historical messages | [ ] | [ ] | Page refresh preserves history |
| **Persona Selection** | | | |
| Role dropdown | [ ] | [ ] | Auto/Developer/Operator options |
| Persona in request | N/A | [ ] | Verify in network tab |
| Persona affects response | [ ] | [ ] | Different prompt context |
| **File Operations** | | | |
| File creation | [ ] | [ ] | `textEditor create` works |
| File editing | [ ] | [ ] | `textEditor str_replace` works |
| Pending indicators | [ ] | [ ] | Yellow dots appear |
| **Review Workflow** | | | |
| Commit changes | [ ] | [ ] | New revision created |
| Discard changes | [ ] | [ ] | Pending cleared |
| Plan display | [ ] | N/A* | PlanChatMessage shows |
| Proceed/Ignore | [ ] | N/A* | Plan actions work |
| **Real-time Events** | | | |
| File updates | [ ] | [ ] | Centrifugo artifact events |
| Render stream | [ ] | [ ] | Terminal output updates |
| Conversion progress | [ ] | [ ] | Progress indicator updates |
| **Navigation** | | | |
| Rollback | [ ] | [ ] | Returns to previous revision |
| Terminal output | [ ] | [ ] | Render results display |

*Note: AI SDK path uses Commit/Discard instead of Plan workflow (see Phase 3 decision)

### 4.3 Performance Testing

**Metrics to Measure:**

| Metric | Target | Legacy | AI SDK |
|--------|--------|--------|--------|
| Time to first token | < 2s | - | - |
| Streaming frame rate | > 20 fps | N/A | - |
| Total response time | < 30s | - | - |
| Memory usage | < 200MB | - | - |

### Success Criteria - Phase 4

#### Automated Verification:
- [ ] All integration tests pass
- [ ] No TypeScript errors
- [ ] Build succeeds

#### Manual Verification:
- [ ] All E2E scenarios pass for both paths
- [ ] Performance metrics meet targets
- [ ] No console errors during normal operation
- [ ] Error states handled gracefully (network errors, API errors)

---

## Phase 5: Feature Flag Rollout

**Duration**: 1 day
**Goal**: Enable AI SDK chat in staging, then production

### 5.1 Staging Deployment

**Steps:**
1. Set `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` in staging environment
2. Deploy to staging
3. Run full test suite
4. Monitor for 24 hours

**Monitoring:**
- Error rates
- Response times
- User feedback (if applicable)

### 5.2 Production Rollout

**Gradual Rollout Strategy:**

```typescript
// Optional: Percentage-based rollout
const shouldUseAISDK = () => {
  const flag = process.env.NEXT_PUBLIC_USE_AI_SDK_CHAT;
  if (flag === 'true') return true;
  if (flag === 'false') return false;

  // Percentage rollout (e.g., '50' = 50%)
  const percentage = parseInt(flag ?? '0', 10);
  return Math.random() * 100 < percentage;
};
```

**Production Steps:**
1. Set `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` (or percentage)
2. Deploy
3. Monitor metrics
4. If issues: Set to `false` and redeploy (instant rollback)

### 5.3 Rollback Procedure

**If issues occur:**

1. **Immediate**: Set `NEXT_PUBLIC_USE_AI_SDK_CHAT=false`
2. **Redeploy**: Changes take effect immediately (client-side check)
3. **Investigate**: Review logs, error reports
4. **Fix**: Address issues in development
5. **Retry**: Re-enable after fixes

### Success Criteria - Phase 5

- [ ] Staging runs stable for 24+ hours
- [ ] Production rollout completes without rollback
- [ ] Error rates remain at or below baseline
- [ ] No user complaints about chat functionality

---

## Phase 6: Cleanup

**Duration**: 1-2 days
**Goal**: Remove test path and feature flag after stable period

### 6.1 Prerequisites

Before cleanup:
- [ ] AI SDK path stable in production for 2+ weeks
- [ ] No rollbacks required
- [ ] Team approval to remove legacy path

### 6.2 Files to Remove

```bash
# Remove test-ai-chat page and components
rm -rf chartsmith-app/app/test-ai-chat/
rm -f chartsmith-app/components/chat/AIChat.tsx
rm -f chartsmith-app/components/chat/AIMessageList.tsx

# Remove feature flag checks (make AI SDK the only path)
# This requires editing:
# - ChatContainer.tsx (remove conditional)
# - useLegacyChat.ts (can be removed)
# - Any env files mentioning the flag
```

### 6.3 Code Simplification

After removing legacy path:

```typescript
// ChatContainer.tsx - Simplified
export function ChatContainer({ session }: ChatContainerProps) {
  const [workspace] = useAtom(workspaceAtom);
  const [initialMessages] = useAtom(messagesAtom);

  const chatState = useAISDKChatAdapter(
    workspace?.id ?? '',
    workspace?.currentRevisionNumber ?? 0,
    session,
    initialMessages
  );

  // ... rest unchanged
}
```

### 6.4 Documentation Updates

**Files to Update:**
- `ARCHITECTURE.md` - Document new AI SDK flow
- `chartsmith-app/ARCHITECTURE.md` - Update chat section
- `README.md` - Update any setup instructions

### Success Criteria - Phase 6

- [ ] Test path fully removed
- [ ] Feature flag removed
- [ ] Documentation updated
- [ ] No dead code remaining
- [ ] Build still succeeds
- [ ] All tests pass

---

## Appendix A: File Reference Quick Links

| Purpose | File Path |
|---------|-----------|
| **New Files** | |
| Message mapper | `chartsmith-app/lib/chat/messageMapper.ts` |
| Mapper tests | `chartsmith-app/lib/chat/messageMapper.test.ts` |
| Adapter hook | `chartsmith-app/hooks/useAISDKChatAdapter.ts` |
| Legacy hook | `chartsmith-app/hooks/useLegacyChat.ts` |
| **Modified Files** | |
| Chat container | `chartsmith-app/components/ChatContainer.tsx` |
| **Reference Files** | |
| Message types | `chartsmith-app/components/types.ts` |
| Workspace atoms | `chartsmith-app/atoms/workspace.ts` |
| AI SDK chat endpoint | `chartsmith-app/app/api/chat/route.ts` |
| Tools | `chartsmith-app/lib/ai/tools/*.ts` |
| Centrifugo hook | `chartsmith-app/hooks/useCentrifugo.ts` |

---

## Appendix B: Risk Mitigation Checklist

| Risk | Check |
|------|-------|
| Message format mismatch | [ ] All Message fields mapped or handled |
| Streaming behavior | [ ] Cancel works, errors handled |
| Plan workflow | [ ] Document Commit/Discard as alternative |
| Database sync | [ ] Persistence on completion verified |
| Real-time updates | [ ] Centrifugo integration unchanged |
| Performance | [ ] First token < 2s, smooth streaming |
| Rollback | [ ] Feature flag works in production |

---

## Appendix C: Estimated LOC Changes

| File | Added | Modified | Removed |
|------|-------|----------|---------|
| `messageMapper.ts` | ~150 | - | - |
| `messageMapper.test.ts` | ~100 | - | - |
| `useAISDKChatAdapter.ts` | ~150 | - | - |
| `useLegacyChat.ts` | ~50 | - | - |
| `ChatContainer.tsx` | ~20 | ~30 | ~15 |
| **Phase 1-5 Total** | ~470 | ~30 | ~15 |
| **Phase 6 Cleanup** | - | - | ~800 (test path) |

---

*This implementation plan provides a structured approach to integrating AI SDK into the main workspace path while minimizing risk through feature flags and incremental changes.*
