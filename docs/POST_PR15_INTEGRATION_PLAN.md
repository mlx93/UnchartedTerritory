# Post-PR1.5 Integration Plan: Collapsing AI SDK into Existing UI

**Date**: 2024-12-04  
**Status**: 📋 Planning  
**Prerequisite**: PR1.5 (Tools) must be complete first

---

## Overview

After PR1.5 adds tool support to the AI SDK chat, we will **integrate it into the existing UI** rather than maintaining separate pages. The goal is:

- ✅ **Same UI** - Users see no visual change
- ✅ **Better backend** - Faster streaming, better tool integration
- 🗑️ **Delete test pages** - `/test-ai-chat` is temporary

---

## 1. Components to Modify

### Phase 1: Landing Page Chat

| Component | Current Backend | New Backend |
|-----------|-----------------|-------------|
| `CreateChartOptions.tsx` | `createWorkspaceFromPromptAction()` | `useChat` + `sendMessage()` |

**Changes Required:**

```diff
// CreateChartOptions.tsx

+ import { useChat } from "@ai-sdk/react";
+ import { TextStreamChatTransport } from "ai";

  export function CreateChartOptions() {
-   const handlePromptSubmit = async () => {
-     const w = await createWorkspaceFromPromptAction(session, prompt);
-     router.replace(`/workspace/${w.id}`);
-   };

+   const transport = new TextStreamChatTransport({
+     api: "/api/chat",
+     body: { provider: 'anthropic', model: 'anthropic/claude-sonnet-4-20250514' },
+   });
+
+   const { sendMessage, messages, status } = useChat({ transport });
+
+   const handlePromptSubmit = async () => {
+     await sendMessage({ text: prompt });
+     // Tool will create workspace and redirect
+   };
```

### Phase 2: New Chart Flow

| Component | Current Backend | New Backend |
|-----------|-----------------|-------------|
| `NewChartContent.tsx` | Props from `ChatContainer` | Direct `useChat` integration |
| `NewChartChatMessage.tsx` | `createChatMessageAction()` | `sendMessage()` |

**Changes Required:**

```diff
// NewChartContent.tsx

- interface NewChartContentProps {
-   handleSubmitChat: (e: React.FormEvent) => void;
- }
+ // Remove prop - use internal useChat hook instead

- export function NewChartContent({ handleSubmitChat }: NewChartContentProps) {
+ export function NewChartContent({ session }: { session: Session }) {
+   const { sendMessage, messages, status } = useChat({ ... });
```

### Phase 3: Workspace Chat (Sidebar)

| Component | Current Backend | New Backend |
|-----------|-----------------|-------------|
| `ChatContainer.tsx` | `createChatMessageAction()` → PostgreSQL → Go Worker | `useChat` → `/api/chat` → AI SDK |
| `ChatMessage.tsx` | Jotai atoms for messages | AI SDK message format |

**Changes Required:**

```diff
// ChatContainer.tsx

- import { createChatMessageAction } from "@/lib/workspace/actions/create-chat-message";
+ import { useChat } from "@ai-sdk/react";

  export function ChatContainer({ session }: ChatContainerProps) {
-   const [messages, setMessages] = useAtom(messagesAtom);
+   const { messages, sendMessage, status } = useChat({
+     transport: new TextStreamChatTransport({
+       api: "/api/chat",
+       body: { 
+         provider: 'anthropic', 
+         model: 'anthropic/claude-sonnet-4-20250514',
+         workspaceId: workspace?.id,  // Pass workspace context
+       },
+     }),
+   });

    const handleSubmitChat = async (e: React.FormEvent) => {
-     const chatMessage = await createChatMessageAction(session, workspace.id, chatInput.trim(), selectedRole);
-     setMessages(prev => [...prev, chatMessage]);
+     await sendMessage({ text: chatInput.trim() });
    };
```

---

## 2. Message Format Conversion

The existing system uses a custom `Message` type. The AI SDK uses `UIMessage`.

### Mapping Table

| Existing Field | AI SDK Equivalent | Notes |
|----------------|-------------------|-------|
| `message.prompt` | `message.parts[0].text` (role=user) | User input |
| `message.response` | `message.parts[0].text` (role=assistant) | Assistant text |
| `message.responsePlanId` | Tool invocation result | Via `createPlan` tool |
| `message.responseRenderId` | Tool invocation result | Via `renderChart` tool |
| `message.isIntentComplete` | `status === 'ready'` | Stream complete |
| `message.isCanceled` | N/A (use `stop()`) | Call `stop()` from useChat |

### Message Component Adaptation

```tsx
// Existing ChatMessage pattern
{message.response && <ReactMarkdown>{message.response}</ReactMarkdown>}

// AI SDK pattern  
{message.parts?.map((part, i) => {
  if (part.type === 'text') return <ReactMarkdown key={i}>{part.text}</ReactMarkdown>;
  if (part.type === 'tool-invocation') return <ToolResult key={i} {...part} />;
  return null;
})}
```

---

## 3. Jotai Atom Migration

### Atoms to Keep (Read-Only from AI SDK)

| Atom | Purpose | Integration |
|------|---------|-------------|
| `workspaceAtom` | Current workspace | Pass to tool calls |
| `plansAtom` | Plans list | Updated via tool results |
| `rendersAtom` | Renders list | Updated via tool results |
| `conversionsAtom` | Conversions | Updated via tool results |

### Atoms to Deprecate

| Atom | Reason |
|------|--------|
| `messagesAtom` | Replaced by `useChat` state |
| `isRenderingAtom` | Replaced by `status === 'streaming'` |

### Migration Pattern

```tsx
// Before: Dual state management
const [messages, setMessages] = useAtom(messagesAtom);
const [isRendering] = useAtom(isRenderingAtom);

// After: AI SDK owns message state
const { messages, status } = useChat({ ... });
const isRendering = status === 'submitted' || status === 'streaming';

// Keep workspace atoms - tools update these
const [workspace] = useAtom(workspaceAtom);
const [plans, setPlans] = useAtom(plansAtom);
```

---

## 4. Tool Requirements (from PR1.5)

For the integration to work, PR1.5 must provide these tools:

| Tool | Purpose | Used By |
|------|---------|---------|
| `createWorkspace` | Create new workspace from prompt | Landing page |
| `getChartContext` | Get current chart files | All chat flows |
| `textEditor` | View/edit/create files | Workspace chat |
| `createPlan` | Generate modification plan | Workspace chat |
| `executePlan` | Apply plan to chart | Workspace chat |
| `latestSubchartVersion` | Query ArtifactHub | Workspace chat |

---

## 5. Migration Steps

### Step 1: Verify PR1.5 Tools Work
- [ ] All tools execute successfully via `/api/chat`
- [ ] Tools can interact with Go backend via HTTP
- [ ] Workspace context passed correctly

### Step 2: Landing Page Integration
- [ ] Modify `CreateChartOptions.tsx` to use `useChat`
- [ ] Add `createWorkspace` tool call on submit
- [ ] Test full landing → workspace flow

### Step 3: New Chart Flow Integration  
- [ ] Modify `NewChartContent.tsx` to use `useChat`
- [ ] Adapt `NewChartChatMessage.tsx` for AI SDK messages
- [ ] Test conversation + plan creation

### Step 4: Workspace Chat Integration
- [ ] Modify `ChatContainer.tsx` to use `useChat`
- [ ] Adapt `ChatMessage.tsx` for AI SDK messages
- [ ] Test modifications + renders + rollbacks

### Step 5: Cleanup
- [ ] Delete `/test-ai-chat` page
- [ ] Remove deprecated Jotai atoms
- [ ] Remove old server actions (if fully replaced)
- [ ] Update tests

---

## 6. Rollback Plan

If integration causes issues:

1. **Keep old code paths** - Don't delete until verified
2. **Feature flag** - `USE_AI_SDK_CHAT=true` in env
3. **Gradual rollout** - Test with specific workspaces first

```tsx
const useAISDKChat = process.env.NEXT_PUBLIC_USE_AI_SDK_CHAT === 'true';

if (useAISDKChat) {
  // New AI SDK path
} else {
  // Original Go backend path
}
```

---

## 7. Files to Delete After Integration

| File | Reason |
|------|--------|
| `app/test-ai-chat/page.tsx` | Test page no longer needed |
| `components/chat/AIChat.tsx` | Integrated into existing components |
| `components/chat/AIMessageList.tsx` | Integrated into ChatMessage |
| `components/chat/ProviderSelector.tsx` | Provider hardcoded or simplified |

---

## Timeline Estimate

| Phase | Effort | Dependencies |
|-------|--------|--------------|
| PR1.5 Tools | 2-3 days | PR1 complete ✅ |
| Landing Page | 0.5 days | PR1.5 complete |
| New Chart Flow | 1 day | PR1.5 complete |
| Workspace Chat | 1-2 days | PR1.5 complete |
| Cleanup & Testing | 1 day | All above |

**Total Post-PR1.5**: ~4-5 days

---

*This document should be updated as PR1.5 progresses and requirements become clearer.*

