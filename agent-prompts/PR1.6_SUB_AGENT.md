# PR1.6 Implementation Agent

## Your Mission

Implement PR1.6: Test Path Feature Parity for Chartsmith.

This PR achieves feature parity between the `/test-ai-chat` path and the main `/workspace/[id]` path. You will fix tool execution, add workspace creation, integrate the file explorer, implement chat persistence, and fix CSS issues.

---

## Primary Documents (READ FIRST - IN ORDER)

1. `PRDs/PR1.6_Tech_PRD.md` - Technical specification (implementation details)
2. `PRDs/PR1.6_Product_PRD.md` - Product requirements (acceptance criteria)
3. `docs/research/2025-12-04-PR1.6-FEATURE-PARITY-ANALYSIS.md` - Research findings and root cause analysis
4. **`docs/research/2025-12-04-PR1.6-PRD-GAPS-ANALYSIS.md`** - **CRITICAL: Implementation gotchas and fixes**
5. `chartsmith/docs/PR1.5_KNOWN_ISSUES.md` - Original issue documentation

**Note**: `PRDs/` contains planning docs at workspace root. `chartsmith/` is the forked repo. All TypeScript goes in `chartsmith/chartsmith-app/`.

**IMPORTANT**: The gaps analysis document (#4) contains critical fixes that override some code snippets in other documents. Read it carefully before implementing.

---

## Critical Context

### What Already Exists (PR1.5 Complete)

| File | Purpose |
|------|---------|
| `chartsmith-app/app/api/chat/route.ts` | API route with 4 registered tools |
| `chartsmith-app/lib/ai/tools/*.ts` | Tool implementations (working) |
| `chartsmith-app/lib/ai/prompts.ts` | System prompt (needs simplification) |
| `chartsmith-app/app/test-ai-chat/[workspaceId]/page.tsx` | Test chat page (needs enhancement) |
| `chartsmith-app/components/FileBrowser.tsx` | File explorer component (to be reused) |
| `chartsmith-app/atoms/workspace.ts` | Jotai atoms for workspace state |

### The #1 Fix: System Prompt Simplification

The current system prompt at `lib/ai/prompts.ts` includes verbose tool documentation. This competes with AI SDK's automatic tool schema passing, confusing the model into outputting text descriptions instead of making tool calls.

**Current Problem** (lines 23-68 of prompts.ts):
- Documents every tool with full parameters
- Shows example invocations
- Duplicates what AI SDK already provides

**Solution**:
- Remove detailed tool parameter documentation
- Keep only behavioral guidelines
- Let AI SDK handle tool schema passing

### Prerequisites

Before starting, verify the Go backend is running:
```bash
curl http://localhost:8080/health
# Should return healthy response
```

---

## Implementation Phases

**Important**: Complete and test each phase before moving to the next. Phase 1 is the critical foundation - if tools don't execute properly, later phases won't work correctly.

### Phase 1: Fix Tool Execution (P0) - 30 minutes

**Goal**: AI executes tools instead of outputting XML text.

**File to Modify**: `chartsmith-app/lib/ai/prompts.ts`

**Task**: Replace `CHARTSMITH_TOOL_SYSTEM_PROMPT` with simplified version:

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

**Testing**:
1. Navigate to `/test-ai-chat/{existingWorkspaceId}`
2. Ask "What files are in my chart?"
3. Verify `getChartContext` tool is called (not text output)
4. Ask "What's the latest PostgreSQL subchart version?"
5. Verify `latestSubchartVersion` tool is called with result

**Success Criteria**:
- [ ] `getChartContext` executes and returns file list
- [ ] `textEditor view` shows file contents
- [ ] `latestSubchartVersion` returns actual version
- [ ] No XML-formatted tool text appears in chat

**Note**: Tool result UI rendering already exists in the test page (lines 186-200). This phase is primarily a system prompt fix and verification step, not new UI implementation.

---

### Phase 2: Add Workspace Creation (P1) - 2-3 hours

**Goal**: Users can create workspaces from test path landing page.

**Files to Create/Modify**:
- `chartsmith-app/app/test-ai-chat/page.tsx` (may need creation or modification)

**Implementation**:

Check if `app/test-ai-chat/page.tsx` exists. If it redirects to the main path or shows a placeholder, replace with landing page:

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

**Testing**:
1. Navigate to `/test-ai-chat`
2. Enter a prompt like "Create a simple nginx deployment"
3. Click submit
4. Verify redirect to `/test-ai-chat/{newWorkspaceId}`
5. Verify workspace appears in database

**Success Criteria**:
- [ ] Landing page renders at `/test-ai-chat`
- [ ] Prompt input accepts text
- [ ] Submit creates workspace
- [ ] Redirects to workspace page
- [ ] Error handling works

---

### Phase 3: Add File Explorer (P1) - 3-4 hours

**Goal**: Show file tree alongside chat.

**Files to Modify/Create**:
- `chartsmith-app/app/test-ai-chat/[workspaceId]/page.tsx` - Convert to server component
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx` - New client component

**Architecture**:
1. Server component fetches workspace data
2. Client component hydrates Jotai atoms
3. Layout shows FileBrowser on left, Chat on right

**Implementation Steps**:

1. **Read the current `page.tsx`** to understand its structure
2. **Create `client.tsx`** with the client component code
3. **Convert `page.tsx`** to server component that:
   - Gets session
   - Fetches workspace via `getWorkspaceAction`
   - Passes workspace and session to client component

**Server Component** (`page.tsx`):
```typescript
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

**Client Component** (`client.tsx`):
- Move existing client code from `page.tsx`
- Add Jotai atom hydration in `useEffect`
- Add FileBrowser component to layout
- **IMPORTANT**: Only import `workspaceAtom` - the others are derived

**Key Code for Atom Hydration**:
```typescript
import { useSetAtom } from "jotai";
import { workspaceAtom } from "@/atoms/workspace";  // Only this one!
import { FileBrowser } from "@/components/FileBrowser";
import { getWorkspaceAction } from "@/lib/workspace/actions/get-workspace";

// In component:
// IMPORTANT: chartsAtom and looseFilesAtom are DERIVED atoms (read-only)
// They automatically derive from workspaceAtom - only set workspaceAtom
const setWorkspace = useSetAtom(workspaceAtom);

useEffect(() => {
  setWorkspace(workspace);
  // Charts and files derive automatically - no need to set them!
}, [workspace, setWorkspace]);

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
```

**Session Migration Note**: The current test page uses `useSession()` hook (client-side). The new pattern passes session from server component as a prop - no hook needed in client.

**Layout Structure**:
```typescript
<EditorLayout>
  <div className="flex w-full h-[calc(100vh-3.5rem)]">
    {/* File Explorer - Left Panel */}
    <div className="w-64 flex-shrink-0 border-r border-dark-border overflow-y-auto">
      <FileBrowser />
    </div>

    {/* Chat Panel - Right */}
    <div className="flex-1">
      {/* Existing chat UI */}
    </div>
  </div>
</EditorLayout>
```

**Testing**:
1. Navigate to workspace page
2. Verify file tree shows in left panel
3. Ask AI to create a new file
4. Verify file appears in explorer (updates via refetch after tool completion)

**Success Criteria**:
- [ ] File explorer displays chart files
- [ ] Files show proper folder structure
- [ ] Files update after AI makes changes (via refetch)
- [ ] Chat panel works normally
- [ ] Only `workspaceAtom` is set (derived atoms update automatically)

---

### Phase 4: Add Chat Persistence (P2) - 4-6 hours

**Goal**: Messages survive page refresh.

**Files to Create/Modify**:
- `chartsmith-app/lib/workspace/actions/update-chat-message-response.ts` (NEW)
- `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx` (modify)

**Implementation Steps**:

1. **Create new server action** for updating AI response:

```typescript
// lib/workspace/actions/update-chat-message-response.ts
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

  // IMPORTANT: workspace_chat table does NOT have updated_at column!
  // Use is_intent_complete instead of is_complete
  await db.query(
    `UPDATE workspace_chat
     SET response = $1,
         is_intent_complete = $2
     WHERE id = $3`,
    [response, isComplete, chatMessageId]
  );
}
```

**Schema Note**: Check `testdata/init-scripts-merged.sql:25` - the `workspace_chat` table does not have an `updated_at` column.

2. **Create new server action** `create-ai-sdk-chat-message.ts`:

**IMPORTANT**: The existing `createChatMessageAction` only accepts 4 parameters and triggers Go backend intent processing. Create a new action:

```typescript
// lib/workspace/actions/create-ai-sdk-chat-message.ts
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

  // Insert directly, skipping enqueueWork("new_intent") call
  await db.query(
    `INSERT INTO workspace_chat (id, workspace_id, prompt, response, created_at, message_from_persona, intent, is_intent_complete)
     VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
    [id, workspaceId, message, null, now, ChatMessageFromPersona.AUTO, ChatMessageIntent.NON_PLAN, true]
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

**⚠️ Schema Verification Required**: Before implementing, verify the `workspace_chat` table schema. Additional columns like `revision_number`, `sent_by`, and various `is_intent_*` flags may have NOT NULL constraints. Adjust the INSERT to include all required columns with appropriate default values.

```bash
# Run this to see the full schema:
grep -A 50 "CREATE TABLE workspace_chat" chartsmith/testdata/init-scripts-merged.sql
```

3. **Modify client component** to:
   - Track current message ID
   - Call `createAISDKChatMessageAction` before sending to AI SDK
   - Call `updateChatMessageResponseAction` when AI response completes

**Key Code**:
```typescript
import { createAISDKChatMessageAction } from "@/lib/workspace/actions/create-ai-sdk-chat-message";
import { updateChatMessageResponseAction } from "@/lib/workspace/actions/update-chat-message-response";

// State
const [currentChatMessageId, setCurrentChatMessageId] = useState<string | null>(null);

// On submit
const handleChatSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  if (!chatInput.trim() || isLoading) return;

  // 1. Persist user message using AI SDK-specific action (skips Go intent processing)
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

// On AI response complete
useEffect(() => {
  if (status === "ready" && currentChatMessageId && messages.length > 0) {
    const lastAssistant = messages.filter(m => m.role === "assistant").pop();
    if (lastAssistant) {
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

3. **Load existing messages** on page load:

In server component, fetch messages:
```typescript
const existingMessages = await getWorkspaceMessagesAction(session, workspaceId);
```

Pass to client and display before AI SDK messages.

**Message Format Incompatibility Note**:
Database messages (`ChatMessage`) and AI SDK messages (`UIMessage`) have different formats:
- Database: `{ id, prompt, response, createdAt, ... }`
- AI SDK: `{ id, role, content, parts, ... }`

**Recommended approach**: Display persisted messages separately from AI SDK messages (in a "Previous conversation" section above current chat). This avoids complex format conversion.

**Testing**:
1. Send a chat message
2. Wait for AI response
3. Refresh page
4. Verify message persists (in history section)

**Success Criteria**:
- [ ] User messages persist immediately
- [ ] AI responses persist when complete
- [ ] Messages survive refresh (shown in history section)
- [ ] New `createAISDKChatMessageAction` created (skips Go intent processing)
- [ ] No `updated_at` in SQL query (column doesn't exist)

---

### Phase 5: Fix CSS Issues (P3) - 30 minutes

**Goal**: Remove white highlighting from chat text.

**File to Modify**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx` (or page.tsx depending on Phase 3)

**Current Problem** - search for `prose prose-sm dark:prose-invert`:
```typescript
<div className="prose prose-sm dark:prose-invert max-w-none">
  <ReactMarkdown>{part.text || ""}</ReactMarkdown>
</div>
```

**Note**: Line numbers may shift during development. Search for the pattern instead of relying on specific line numbers.

**Solution**: Remove `prose` classes, add explicit styling:

```typescript
<div className="max-w-none">
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

**Testing**:
1. Switch to dark mode
2. Verify no white highlighting
3. Verify code blocks look correct
4. Test in light mode

**Success Criteria**:
- [ ] No white highlighting
- [ ] Code blocks readable
- [ ] Both themes work

---

## DO NOT

- Modify the API route (`route.ts`) - tools are already registered
- Change tool implementations - they work correctly
- Add Centrifugo integration - that's PR1.7
- Add revision tracking - that's PR1.7
- Modify main workspace path - only touch test-ai-chat

---

## COMPLETION REQUIREMENTS

When you have completed ALL phases, create:

**File: `docs/PR1.6_COMPLETION_REPORT.md`**

### Required Sections:

#### 1. Files Summary Table
```markdown
| Action | File Path | Description |
|--------|-----------|-------------|
| Modified | chartsmith-app/lib/ai/prompts.ts | Simplified system prompt |
| Created | chartsmith-app/app/test-ai-chat/page.tsx | Landing page |
| ... | ... | ... |
```

#### 2. Phase Completion Status
- [ ] Phase 1: Tool execution fixed
- [ ] Phase 2: Workspace creation working
- [ ] Phase 3: File explorer integrated
- [ ] Phase 4: Chat persistence implemented
- [ ] Phase 5: CSS fixed

#### 3. Test Results
Document manual testing results for each phase.

#### 4. Deviations from Spec
Any changes made with rationale.

#### 5. PR1.7 Readiness
Confirm what was deferred:
- [ ] Revision tracking (deferred)
- [ ] Centrifugo integration (deferred)
- [ ] Plan workflow (deferred)

---

*When your completion report is ready, the Master Orchestrator will review it and provide the PR1.7 sub-agent prompt for deeper system integration work.*
