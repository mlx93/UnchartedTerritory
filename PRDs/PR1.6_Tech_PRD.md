# PR1.6: Test Path Feature Parity - Technical Specification

**Version**: 1.0
**PR**: PR1.6 (Feature Parity)
**Prerequisite**: PR1.5 merged
**Status**: Ready for Implementation
**Last Updated**: December 4, 2025

---

## Technical Overview

### Objective

Achieve feature parity between the `/test-ai-chat` path and the main `/workspace/[id]` path. This requires fixing tool execution, adding workspace creation, integrating the file explorer, implementing chat persistence, and fixing CSS issues.

### Key Architecture Decision

| Question | Decision | Rationale |
|----------|----------|-----------|
| Tool execution fix | Simplify system prompt | Remove verbose tool documentation; AI SDK provides tool descriptions automatically |
| Workspace creation | Reuse existing action | `createWorkspaceFromPromptAction` already exists |
| File explorer | Add WorkspaceContainer + atom hydration | Requires server/client split and Jotai state |
| Chat persistence | Hybrid with existing tables | Use `workspace_chat` table via server actions |
| CSS fix | Remove conflicting prose classes | Tailwind Typography conflicts with dark mode |

### Root Cause Analysis: Tool Execution Issue

**Observed Symptom**: AI outputs tool invocations as text (e.g., `<latestSubchartVersion>...`) instead of using structured tool calls.

**Investigation Findings**:
1. The test page **correctly passes** `workspaceId` and `revisionNumber` to the API (lines 51-55 of `page.tsx`)
2. The route handler **correctly creates tools** via `createTools()` (line 113-114 of `route.ts`)
3. The route handler **correctly uses** `toUIMessageStreamResponse()` (line 133)
4. The route handler **correctly sets** `maxSteps: 5` (line 128)

**Root Cause**: The `CHARTSMITH_TOOL_SYSTEM_PROMPT` in `lib/ai/prompts.ts` includes extensive tool documentation (lines 23-68) that describes tool names, parameters, and usage patterns. When the model sees this verbose documentation AND receives the same tools via AI SDK's automatic tool passing, it can become confused and output descriptions instead of making actual tool calls.

**Solution**: Simplify the system prompt to remove detailed tool parameter documentation. The AI SDK automatically provides tool schemas and descriptions to the model - the system prompt should only provide behavioral guidelines, not duplicate the tool documentation.

---

## System Architecture

### After PR1.6

```
┌─────────────────────────────────────────────────────────────────┐
│                    TEST PATH (/test-ai-chat)                     │
│                                                                  │
│  Phase 1: Tool Fix                                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ lib/ai/prompts.ts - Simplified system prompt               │ │
│  │ (Remove tool parameter documentation, keep behavior rules) │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Phase 2: Workspace Creation                                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ app/test-ai-chat/page.tsx - Landing page with prompt input │ │
│  │ → calls createWorkspaceFromPromptAction()                  │ │
│  │ → redirects to /test-ai-chat/{workspaceId}                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Phase 3: File Explorer                                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ app/test-ai-chat/[workspaceId]/page.tsx                    │ │
│  │ → Server component fetches workspace                       │ │
│  │ → Client component hydrates Jotai atoms                    │ │
│  │ → Renders FileBrowser + Chat in split view                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Phase 4: Chat Persistence                                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ createChatMessageAction() - On user message send           │ │
│  │ updateChatMessageResponseAction() - On AI response done    │ │
│  │ (NEW server action)                                        │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## File Structure Changes

### Modified Files

| File | Changes |
|------|---------|
| `lib/ai/prompts.ts` | Simplify system prompt to remove tool documentation |
| `app/test-ai-chat/page.tsx` | Add workspace creation flow (may need to create) |
| `app/test-ai-chat/[workspaceId]/page.tsx` | Convert to server/client pattern, add file explorer |

### New Files

| File | Purpose |
|------|---------|
| `lib/workspace/actions/update-chat-message-response.ts` | Server action to update AI response |
| `lib/workspace/actions/create-ai-sdk-chat-message.ts` | Server action to persist AI SDK chat messages (skips Go intent processing) |

---

## Phase 1: Fix Tool Execution (P0)

### Problem Statement

The AI outputs tool invocations as text instead of executing them. While the technical configuration is correct (tools registered, maxSteps set, correct response format), the system prompt provides excessive tool documentation that competes with AI SDK's automatic tool descriptions.

### Solution

Simplify `CHARTSMITH_TOOL_SYSTEM_PROMPT` to remove tool parameter details:

**Current (Problematic)**:
```typescript
export const CHARTSMITH_TOOL_SYSTEM_PROMPT = `...
### getChartContext
Load the current workspace including all charts, files, and metadata.
- Use this first to understand the current state of the chart
- Returns the complete file structure and contents
- No parameters required (uses current workspace)

### textEditor
View, edit, or create files in the chart.
Commands:
- **view**: Read a file's contents. Use before making changes.
- **create**: Create a new file. Fails if file already exists.
- **str_replace**: Replace text in a file. Supports fuzzy matching.

Parameters:
- command: "view" | "create" | "str_replace"
- path: File path relative to chart root (e.g., "templates/deployment.yaml")
- content: (create only) Full content of the new file
- oldStr: (str_replace only) Text to find
- newStr: (str_replace only) Text to replace with
...`;
```

**Fixed (Simplified)**:
```typescript
export const CHARTSMITH_TOOL_SYSTEM_PROMPT = `You are ChartSmith, an expert AI assistant specializing in Helm charts.

You have access to tools that can:
- Get chart context (view all files and metadata)
- Edit files (view, create, and modify content)
- Look up version information (subchart and Kubernetes versions)

When the user asks about their chart, use the available tools to gather context and make changes. Do not describe how you would use tools - just use them directly.

## Behavior Guidelines
- Use tools to view files before making changes
- Make precise, targeted edits using str_replace
- Always verify changes by viewing the updated file
- Focus on Helm charts and Kubernetes configuration

## System Constraints
- Focus exclusively on Helm charts and Kubernetes manifests
- Use 2-space indentation for YAML
- Ensure valid Helm templating syntax

## Response Format
- Use Markdown for responses
- Be concise and precise
- Provide code examples when helpful`;
```

### Implementation Steps

1. **Read** current `lib/ai/prompts.ts`
2. **Replace** `CHARTSMITH_TOOL_SYSTEM_PROMPT` with simplified version
3. **Test** by asking "What files are in my chart?" - should trigger `getChartContext` tool call
4. **Verify** tool results appear in UI with proper formatting

### Acceptance Criteria

- [ ] AI calls `getChartContext` when asked about chart contents
- [ ] AI calls `textEditor` with `view` command when asked to show a file
- [ ] AI calls `latestSubchartVersion` when asked about chart versions
- [ ] Tool invocations display in UI (rendering code already exists - verify it works)

**Note**: The test page already has tool result rendering at lines 186-200. Phase 1 is primarily a system prompt fix and verification step, not new UI implementation.

---

## Phase 2: Add Workspace Creation (P1)

### Problem Statement

Users cannot create new workspaces from the test path. They must manually navigate to an existing workspace ID.

### Solution

Create a landing page at `/test-ai-chat` that:
1. Shows a prompt input UI
2. Calls `createWorkspaceFromPromptAction` on submit
3. Redirects to `/test-ai-chat/{newWorkspaceId}`

### Implementation

**File**: `app/test-ai-chat/page.tsx` (modify or create)

```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { useSession } from "@/app/hooks/useSession";
import { createWorkspaceFromPromptAction } from "@/lib/workspace/actions/create-workspace-from-prompt";
import { EditorLayout } from "@/components/layout/EditorLayout";
import { useTheme } from "@/contexts/ThemeContext";
import { Send, Loader2 } from "lucide-react";

export default function TestAIChatLandingPage() {
  const router = useRouter();
  const { session } = useSession();
  const { theme } = useTheme();
  const [prompt, setPrompt] = useState("");
  const [isCreating, setIsCreating] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!prompt.trim() || !session || isCreating) return;

    setIsCreating(true);
    setError(null);

    try {
      const workspace = await createWorkspaceFromPromptAction(session, prompt.trim());
      router.push(`/test-ai-chat/${workspace.id}`);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to create workspace");
      setIsCreating(false);
    }
  };

  return (
    <EditorLayout>
      <div className="flex flex-col items-center justify-center min-h-[calc(100vh-3.5rem)] p-8">
        <div className="max-w-2xl w-full">
          <h1 className="text-3xl font-bold mb-2 text-center">
            AI SDK Test Chat
          </h1>
          <p className={`text-center mb-8 ${theme === "dark" ? "text-gray-400" : "text-gray-600"}`}>
            Describe your Helm chart and I'll help you build it
          </p>

          <form onSubmit={handleSubmit} className="space-y-4">
            <textarea
              value={prompt}
              onChange={(e) => setPrompt(e.target.value)}
              placeholder="Describe your Helm chart... (e.g., 'Create a PostgreSQL deployment with persistent storage')"
              rows={4}
              disabled={isCreating}
              className={`w-full px-4 py-3 rounded-lg border ${
                theme === "dark"
                  ? "bg-dark-surface border-dark-border text-white placeholder-gray-500"
                  : "bg-white border-gray-200 text-gray-900 placeholder-gray-400"
              } focus:outline-none focus:ring-2 focus:ring-primary/50 resize-none`}
            />

            {error && (
              <p className="text-red-500 text-sm">{error}</p>
            )}

            <button
              type="submit"
              disabled={!prompt.trim() || isCreating || !session}
              className={`w-full py-3 rounded-lg font-medium transition-colors flex items-center justify-center gap-2
                ${prompt.trim() && session && !isCreating
                  ? "bg-primary text-white hover:bg-primary/90"
                  : "bg-gray-300 text-gray-500 cursor-not-allowed"
                }`}
            >
              {isCreating ? (
                <>
                  <Loader2 className="w-5 h-5 animate-spin" />
                  Creating workspace...
                </>
              ) : (
                <>
                  <Send className="w-5 h-5" />
                  Start Building
                </>
              )}
            </button>
          </form>
        </div>
      </div>
    </EditorLayout>
  );
}
```

### Acceptance Criteria

- [ ] Landing page renders at `/test-ai-chat`
- [ ] Prompt input field accepts text
- [ ] Submit creates workspace and redirects
- [ ] Loading state shown during creation
- [ ] Error handling for failed creation

---

## Phase 3: Add File Explorer (P1)

### Problem Statement

The test path shows only chat; users cannot see or navigate the file tree.

### Solution

Restructure the test page to:
1. **Server component**: Fetch workspace data
2. **Client component**: Hydrate Jotai atoms, render FileBrowser + Chat

### Architecture Pattern

```
app/test-ai-chat/[workspaceId]/page.tsx (Server)
  └── getWorkspaceAction() - Fetch workspace
  └── <TestAIChatClient workspace={workspace} />

TestAIChatClient (Client)
  └── useEffect to hydrate atoms:
      - setWorkspace(workspace)
      - setCharts(workspace.charts)
      - setMessages([])  // Start fresh for AI SDK chat
  └── Layout:
      - Left: FileBrowser
      - Right: Chat UI (existing)
```

### Implementation

**File**: `app/test-ai-chat/[workspaceId]/page.tsx`

This requires significant restructuring. The current file is entirely a client component. It needs to be split:

```typescript
// Server Component (page.tsx)
import { getWorkspaceAction } from "@/lib/workspace/actions/get-workspace";
import { getSession } from "@/lib/auth";
import { TestAIChatClient } from "./client";
import { redirect } from "next/navigation";

export default async function TestAIChatPage({
  params
}: {
  params: Promise<{ workspaceId: string }>
}) {
  const { workspaceId } = await params;
  const session = await getSession();

  if (!session) {
    redirect("/login");
  }

  const workspace = await getWorkspaceAction(session, workspaceId);

  if (!workspace) {
    redirect("/test-ai-chat");
  }

  return <TestAIChatClient workspace={workspace} session={session} />;
}
```

```typescript
// Client Component (client.tsx - new file)
"use client";

import React, { useState, useEffect, useRef } from "react";
import { useSearchParams } from "next/navigation";
import { useSetAtom } from "jotai";
import { Loader2, Send } from "lucide-react";
import { useChat } from "@ai-sdk/react";

// Atoms - NOTE: chartsAtom and looseFilesAtom are DERIVED atoms (read-only)
// They automatically derive from workspaceAtom - only set workspaceAtom
import { workspaceAtom } from "@/atoms/workspace";

// Actions
import { getWorkspaceAction } from "@/lib/workspace/actions/get-workspace";

// Components
import { ScrollingContent } from "@/components/ScrollingContent";
import { EditorLayout } from "@/components/layout/EditorLayout";
import { FileBrowser } from "@/components/FileBrowser";
import { useTheme } from "@/contexts/ThemeContext";
import ReactMarkdown from "react-markdown";
import Image from "next/image";

// Types
import { Workspace } from "@/lib/types/workspace";
import { Session } from "@/lib/types/session";

// Config
import {
  getDefaultProvider,
  getDefaultModelForProvider,
  STREAMING_THROTTLE_MS,
} from "@/lib/ai";

interface TestAIChatClientProps {
  workspace: Workspace;
  session: Session;
}

export function TestAIChatClient({ workspace, session }: TestAIChatClientProps) {
  const { theme } = useTheme();
  const searchParams = useSearchParams();

  // Atom setter for hydration - only set workspaceAtom
  // chartsAtom and looseFilesAtom are derived and update automatically
  const setWorkspace = useSetAtom(workspaceAtom);

  // Hydrate workspace atom on mount
  // Charts and files derive automatically from workspaceAtom
  useEffect(() => {
    setWorkspace(workspace);
  }, [workspace, setWorkspace]);

  const revisionNumber = workspace.currentRevisionNumber || 1;

  const [chatInput, setChatInput] = useState("");
  const [hasAutoSent, setHasAutoSent] = useState(false);
  const chatInputRef = useRef<HTMLTextAreaElement>(null);

  const selectedProvider = getDefaultProvider();
  const selectedModel = getDefaultModelForProvider(selectedProvider);

  const {
    messages,
    sendMessage,
    status,
  } = useChat({
    api: "/api/chat",
    body: {
      provider: selectedProvider,
      model: selectedModel,
      workspaceId: workspace.id,
      revisionNumber,
    },
    experimental_throttle: STREAMING_THROTTLE_MS,
  });

  const isLoading = status === "submitted" || status === "streaming";

  // Refetch workspace when AI tool calls complete to update file explorer
  // This is a simple solution for PR1.6; PR1.7 will use Centrifugo for real-time
  useEffect(() => {
    const lastMessage = messages[messages.length - 1];
    const hasToolResult = lastMessage?.parts?.some(
      (p: any) => p.type === 'tool-invocation' && p.toolInvocation?.state === 'result'
    );

    if (hasToolResult && status === 'ready') {
      // Refetch workspace to get updated files after tool execution
      getWorkspaceAction(session, workspace.id).then((updated) => {
        if (updated) setWorkspace(updated);
      });
    }
  }, [messages, status, session, workspace.id, setWorkspace]);

  // ... rest of existing chat UI implementation
  // (Keep the existing message rendering, input handling, etc.)

  return (
    <EditorLayout>
      <div className="flex w-full h-[calc(100vh-3.5rem)]">
        {/* File Explorer - Left Panel */}
        <div className={`w-64 flex-shrink-0 border-r ${
          theme === "dark" ? "border-dark-border bg-dark-surface" : "border-gray-200 bg-gray-50"
        } overflow-y-auto`}>
          <div className="p-2">
            <h2 className={`text-xs font-semibold uppercase tracking-wider mb-2 px-2 ${
              theme === "dark" ? "text-gray-500" : "text-gray-400"
            }`}>
              Files
            </h2>
            <FileBrowser />
          </div>
        </div>

        {/* Chat Panel - Right */}
        <div className="flex-1 flex flex-col min-w-0">
          {/* ... existing chat implementation ... */}
        </div>
      </div>
    </EditorLayout>
  );
}
```

### Key Implementation Notes

1. **Atom Hydration**: Only set `workspaceAtom` - `chartsAtom` and `looseFilesAtom` are **derived atoms** (read-only) that automatically update when `workspaceAtom` changes. See `atoms/workspace.ts:95-105`.

2. **Session Migration**: The current test page uses `useSession()` hook (client-side). The new pattern requires:
   - Server component calls `getSession()`
   - Pass session as prop to client component
   - Client uses session directly (no hook needed)

3. **File Explorer Updates**: When AI modifies files via `textEditor`, the database updates but the UI doesn't know. The solution for PR1.6 is to refetch workspace after tool results. PR1.7 will use Centrifugo for real-time updates.

4. **FileBrowser Component**: Import from existing `components/FileBrowser.tsx`. It uses `selectedFileAtom` for tracking selection - file clicks will highlight but no code viewer panel in PR1.6.

5. **Provider Wrap**: Ensure the app has Jotai Provider (likely already in layout)

### Acceptance Criteria

- [ ] File explorer shows chart files in left panel
- [ ] Files update when AI makes changes via textEditor tool (via refetch after tool completion)
- [ ] Selected file highlights in tree
- [ ] Chat panel functions normally on right side

---

## Phase 4: Add Chat Persistence (P2)

### Problem Statement

Chat messages are lost on page refresh. The main path persists to `workspace_chat` table.

### Solution

Create a hybrid persistence approach:
1. **On user message**: Call `createChatMessageAction()` before sending to AI SDK
2. **On AI response complete**: Call new `updateChatMessageResponseAction()` to save response
3. **On page load**: Fetch existing messages and display

### New Server Action

**File**: `lib/workspace/actions/update-chat-message-response.ts`

```typescript
"use server";

import { Session } from "@/lib/types/session";
import { getDB, getParam } from "@/lib/db";

export async function updateChatMessageResponseAction(
  session: Session,
  chatMessageId: string,
  response: string,
  isComplete: boolean = true
): Promise<void> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const db = getDB(await getParam("DB_URI"));

  // NOTE: workspace_chat table does NOT have updated_at column
  // Only update response and is_intent_complete
  await db.query(
    `UPDATE workspace_chat
     SET response = $1,
         is_intent_complete = $2
     WHERE id = $3`,
    [response, isComplete, chatMessageId]
  );
}
```

**Schema Note**: The `workspace_chat` table (see `testdata/init-scripts-merged.sql:25`) does not have an `updated_at` column. Use `is_intent_complete` instead of `is_complete`.

### New Server Action for AI SDK Messages

**File**: `lib/workspace/actions/create-ai-sdk-chat-message.ts`

**Background**: The existing `createChatMessageAction` at `lib/workspace/actions/create-chat-message.ts` only accepts 4 parameters (`session`, `workspaceId`, `message`, `messageFromPersona`) and internally calls `createChatMessage()` which triggers Go backend intent processing via `enqueueWork("new_intent")`. We need a new action that skips this processing for AI SDK messages.

```typescript
"use server";

import { Session } from "@/lib/types/session";
import { getDB, getParam } from "@/lib/db";
import { ChatMessage, ChatMessageFromPersona, ChatMessageIntent } from "@/lib/workspace/chat";
import { v4 as uuidv4 } from "uuid";

export async function createAISDKChatMessageAction(
  session: Session,
  workspaceId: string,
  message: string
): Promise<ChatMessage> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const db = getDB(await getParam("DB_URI"));
  const id = uuidv4();
  const now = new Date();

  // Insert directly into workspace_chat, skipping the enqueueWork("new_intent") call
  // that exists in the standard createChatMessage() function
  await db.query(
    `INSERT INTO workspace_chat (id, workspace_id, prompt, response, created_at, message_from_persona, intent, is_intent_complete)
     VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
    [
      id,
      workspaceId,
      message,
      null,  // response will be updated later
      now,
      ChatMessageFromPersona.AUTO,
      ChatMessageIntent.NON_PLAN,  // Skip intent processing
      true  // Mark as complete since AI SDK handles it
    ]
  );

  return {
    id,
    workspaceId,
    prompt: message,
    response: null,
    createdAt: now,
    messageFromPersona: ChatMessageFromPersona.AUTO,
    intent: ChatMessageIntent.NON_PLAN,
    isIntentComplete: true
  } as ChatMessage;
}
```

**Why a New Action?**
1. Existing `createChatMessageAction` signature: `(session, workspaceId, message, messageFromPersona)` - only 4 params
2. Existing action calls `createChatMessage()` which triggers `enqueueWork("new_intent")` (see `lib/workspace/workspace.ts:229-233`)
3. We want AI SDK to handle the message, not Go backend intent classification
4. Creating a new action is cleaner than modifying the existing one and risking main path regressions

**⚠️ Schema Verification Required**: The INSERT query above uses columns `(id, workspace_id, prompt, response, created_at, message_from_persona, intent, is_intent_complete)`. The implementation agent MUST verify this against the actual `workspace_chat` table schema in `testdata/init-scripts-merged.sql` - additional columns like `revision_number`, `sent_by`, and various `is_intent_*` flags may have NOT NULL constraints that would cause the insert to fail. Adjust the INSERT to include all required columns with appropriate default values.

### Client Integration

In the test chat client, track the current message ID and update when AI completes:

```typescript
import { createAISDKChatMessageAction } from "@/lib/workspace/actions/create-ai-sdk-chat-message";
import { updateChatMessageResponseAction } from "@/lib/workspace/actions/update-chat-message-response";

// Track current user message ID for response persistence
const [currentChatMessageId, setCurrentChatMessageId] = useState<string | null>(null);

const handleChatSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  if (!chatInput.trim() || isLoading) return;

  // 1. Persist user message using AI SDK-specific action
  // NOTE: We use createAISDKChatMessageAction instead of createChatMessageAction
  // because the existing action only accepts 4 parameters and triggers Go backend
  // intent processing via enqueueWork("new_intent"). The new action skips this.
  const chatMessage = await createAISDKChatMessageAction(
    session,
    workspace.id,
    chatInput.trim()
  );
  setCurrentChatMessageId(chatMessage.id);

  // 2. Send to AI SDK
  await sendMessage({ text: chatInput.trim() });
  setChatInput("");
};

// Persist AI response when complete
useEffect(() => {
  if (status === "ready" && currentChatMessageId && messages.length > 0) {
    const lastAssistant = messages.filter(m => m.role === "assistant").pop();
    if (lastAssistant) {
      // Extract text content from parts
      const textContent = lastAssistant.parts
        ?.filter(p => p.type === "text")
        .map(p => p.text)
        .join("\n") || "";

      updateChatMessageResponseAction(
        session,
        currentChatMessageId,
        textContent,
        true
      ).then(() => {
        setCurrentChatMessageId(null);
      });
    }
  }
}, [status, currentChatMessageId, messages, session]);
```

### Load Existing Messages

On page mount, fetch persisted messages:

```typescript
// In server component
const existingMessages = await getWorkspaceMessagesAction(session, workspaceId);

// Pass to client
<TestAIChatClient
  workspace={workspace}
  session={session}
  initialMessages={existingMessages}
/>
```

### Message Format Incompatibility

**Important**: Database messages and AI SDK messages have different formats:

```typescript
// Database ChatMessage format (from lib/workspace/chat.ts)
{ id, prompt, response, createdAt, isComplete, ... }

// AI SDK UIMessage format (from @ai-sdk/react)
{ id, role, content, parts, createdAt, ... }
```

**Recommended approach for PR1.6**: Display persisted messages separately from AI SDK messages:

```typescript
// In client component
interface TestAIChatClientProps {
  workspace: Workspace;
  session: Session;
  initialMessages?: ChatMessage[];  // Database format
}

export function TestAIChatClient({ workspace, session, initialMessages = [] }: TestAIChatClientProps) {
  // AI SDK messages (current session)
  const { messages: aiMessages, ... } = useChat({ ... });

  return (
    <>
      {/* History section - persisted messages */}
      {initialMessages.length > 0 && (
        <div className="border-b pb-4 mb-4">
          <div className="text-xs text-gray-500 mb-2">Previous conversation</div>
          {initialMessages.map(msg => (
            <div key={msg.id}>
              <div className="user-message">{msg.prompt}</div>
              {msg.response && <div className="assistant-message">{msg.response}</div>}
            </div>
          ))}
        </div>
      )}

      {/* Current session - AI SDK messages */}
      {aiMessages.map(message => (
        // ... existing message rendering
      ))}
    </>
  );
}
```

This avoids complex format conversion while providing conversation history.

### Acceptance Criteria

- [ ] User messages persist to database before AI processes
- [ ] AI responses save to database when complete
- [ ] Refreshing page shows previous messages (in history section)
- [ ] Current session messages display via AI SDK format
- [ ] New `createAISDKChatMessageAction` created and used (skips Go intent processing)

---

## Phase 5: Fix CSS Issues (P3)

### Problem Statement

White highlighting covers text in chat bubbles due to Tailwind Typography `prose` class conflicts.

### Solution

Remove or override `prose` classes in the assistant message rendering.

**Current (Problematic)** - search for `prose prose-sm` in the test page:
```typescript
<div className="prose prose-sm dark:prose-invert max-w-none">
  <ReactMarkdown>{part.text || ""}</ReactMarkdown>
</div>
```

**Note**: Line numbers may shift during development. Search for the pattern `prose prose-sm dark:prose-invert` instead of relying on specific line numbers.

**Fixed**:
```typescript
<div className={`max-w-none ${
  theme === "dark"
    ? "[&_code]:bg-gray-800 [&_pre]:bg-gray-900"
    : "[&_code]:bg-gray-100 [&_pre]:bg-gray-50"
}`}>
  <ReactMarkdown
    components={{
      code: ({ className, children, ...props }) => (
        <code
          className={`${className || ''} ${
            theme === "dark" ? "bg-gray-800 text-gray-200" : "bg-gray-100 text-gray-800"
          } px-1 py-0.5 rounded text-sm`}
          {...props}
        >
          {children}
        </code>
      ),
      pre: ({ children }) => (
        <pre className={`${
          theme === "dark" ? "bg-gray-900" : "bg-gray-50"
        } p-3 rounded-lg overflow-x-auto my-2`}>
          {children}
        </pre>
      ),
    }}
  >
    {part.text || ""}
  </ReactMarkdown>
</div>
```

### Acceptance Criteria

- [ ] No white highlighting in dark mode
- [ ] Code blocks have appropriate background
- [ ] Text is readable in both themes

---

## Testing Strategy

### Phase 1 Testing (Tools)

```bash
# 1. Start the development server
npm run dev

# 2. Navigate to existing workspace
# http://localhost:3000/test-ai-chat/{workspaceId}

# 3. Test tool execution with prompts:
# - "What files are in this chart?" → should call getChartContext
# - "Show me the values.yaml file" → should call textEditor view
# - "What's the latest PostgreSQL subchart version?" → should call latestSubchartVersion
# - "Create a new file called test.yaml" → should call textEditor create
```

### Phase 2 Testing (Workspace Creation)

```bash
# 1. Navigate to landing page
# http://localhost:3000/test-ai-chat

# 2. Enter a prompt: "Create a simple nginx deployment"
# 3. Click submit
# 4. Verify redirect to /test-ai-chat/{newWorkspaceId}
# 5. Verify workspace exists in database
```

### Phase 3 Testing (File Explorer)

```bash
# 1. Navigate to workspace page
# 2. Verify file tree displays in left panel
# 3. Ask AI to create a new file
# 4. Verify file appears in explorer without refresh
# 5. Click file in explorer, verify selection
```

### Phase 4 Testing (Persistence)

```bash
# 1. Send a chat message
# 2. Wait for AI response
# 3. Refresh the page
# 4. Verify both user and AI messages persist
# 5. Verify messages display correctly
```

---

## Implementation Priority

| Priority | Phase | Estimated Effort | Dependencies |
|----------|-------|-----------------|--------------|
| P0 | Phase 1: Tool Fix | 30 min | None |
| P1 | Phase 2: Workspace Creation | 2-3 hours | None |
| P1 | Phase 3: File Explorer | 3-4 hours | Phase 2 |
| P2 | Phase 4: Chat Persistence | 4-6 hours | Phase 3 |
| P3 | Phase 5: CSS Fix | 30 min | None |

**Total Estimated Effort**: 10-14 hours

---

## Out of Scope (Deferred to PR1.7)

The following are explicitly deferred:

1. **Revision tracking** - Requires architectural design for batching AI SDK file modifications
2. **Centrifugo real-time integration** - Complex integration with existing WebSocket system
3. **Plan workflow integration** - Requires designing multi-file Plan proposals for AI SDK path
4. **Full WorkspaceContent reuse** - Would duplicate main path complexity

See PR1.7 PRD for architectural decisions on these items.

---

## Validation Checklist

### Must Pass

- [ ] Tool execution works (getChartContext, textEditor, version tools)
- [ ] Workspace creation from landing page works
- [ ] File explorer displays and updates
- [ ] Chat messages persist across refresh
- [ ] No CSS highlighting issues

### Quality Gates

- [ ] All TypeScript compilation passes
- [ ] No console errors in browser
- [ ] Responsive layout works
- [ ] Dark/light theme works correctly

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PR1.6_Product_PRD.md` | Functional requirements |
| `PR1.6_SUB_AGENT.md` | Execution prompt for implementation agent |
| `docs/research/2025-12-04-PR1.6-FEATURE-PARITY-ANALYSIS.md` | Research findings |
| `docs/research/2025-12-04-PR1.6-PRD-GAPS-ANALYSIS.md` | PRD gaps analysis (implementation gotchas) |
| `chartsmith/docs/PR1.5_KNOWN_ISSUES.md` | Issue documentation |

---

*Document End - PR1.6: Test Path Feature Parity Technical Specification*
