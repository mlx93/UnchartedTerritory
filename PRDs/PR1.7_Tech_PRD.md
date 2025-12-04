# PR1.7: Deep System Integration - Technical Specification

**Version**: 1.0
**PR**: PR1.7 (System Integration)
**Prerequisite**: PR1.6 merged
**Status**: Preliminary (Architectural Focus)
**Last Updated**: December 4, 2025

---

## Technical Overview

### Objective

PR1.7 addresses the complex architectural work deferred from PR1.6 to integrate the AI SDK test path deeply into Chartsmith's existing systems:

1. **Revision Tracking** - Create revisions from AI SDK file modifications
2. **Centrifugo Real-Time** - Connect test path to existing WebSocket infrastructure
3. **Plan Workflow** - Enable multi-file Plans from AI SDK path

These features require significant architectural decisions and trade-off analysis.

---

## Architecture Analysis

### Current System Understanding

```
┌─────────────────────────────────────────────────────────────────┐
│                      MAIN PATH FLOW                              │
│                                                                  │
│  User types prompt → createWorkspaceFromPromptAction()           │
│                    → workspace created with revision 0           │
│                    → enqueueWork("new_intent")                   │
│                                                                  │
│  Go Worker picks up → determines intent                          │
│                     → if PLAN: createPlan() → enqueueWork()      │
│                     → Plan streams to Centrifugo                 │
│                     → User reviews in PlanChatMessage            │
│                     → User clicks "Proceed"                      │
│                     → createRevisionAction() called              │
│                     → New revision created, files copied         │
│                     → Plan executed, files modified              │
│                                                                  │
│  Revision Model:                                                 │
│  - Revisions are EXPLICIT, user-triggered via "Proceed"         │
│  - Each revision is a complete snapshot of all files            │
│  - Files are COPIED to new revision number                      │
│  - rollbackToRevision() deletes newer revisions                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      TEST PATH FLOW (PR1.6)                      │
│                                                                  │
│  User types message → sendMessage() via useChat                  │
│                     → streamText() with tools                    │
│                     → AI calls textEditor tool                   │
│                     → Tool calls Go endpoint                     │
│                     → Go modifies file DIRECTLY                  │
│                     → File updated in current revision           │
│                     → NO NEW REVISION CREATED                    │
│                                                                  │
│  Problem:                                                        │
│  - AI makes incremental changes via textEditor                   │
│  - Each change modifies files in-place                          │
│  - No revision history of AI changes                            │
│  - No rollback capability                                       │
│  - No Plan review step                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Feature 1: Revision Tracking

### Problem Statement

The AI SDK `textEditor` tool modifies files directly via the Go HTTP endpoint. The endpoint calls `workspace.SetFileContentPending()` which updates the file content without creating a new revision. This means:

- No history of AI changes
- No ability to rollback AI modifications
- No visibility into what changed

### Existing Pattern Study

**File**: `lib/workspace/workspace.ts:946-1068` - `createRevision()`

The existing revision system:
1. Creates new revision row with incremented number
2. Copies ALL charts from previous revision to new revision
3. Copies ALL files from previous revision to new revision
4. Updates `workspace.current_revision_number`
5. Triggered by user clicking "Proceed" on a Plan

**Key SQL Pattern**:
```sql
-- Atomically get next revision number
WITH latest_revision AS (
  SELECT * FROM workspace_revision
  WHERE workspace_id = $1
  ORDER BY revision_number DESC
  LIMIT 1
),
next_revision AS (
  SELECT COALESCE(MAX(revision_number), 0) + 1 as next_num
  FROM workspace_revision
  WHERE workspace_id = $1
)
INSERT INTO workspace_revision (...)
```

### Architectural Options

#### Option A: Revision Per Tool Call (NOT RECOMMENDED)

Create a new revision each time `textEditor` modifies a file.

**Pros**:
- Complete history of every change
- Perfect rollback granularity

**Cons**:
- Revision explosion (AI might make 10-20 edits per conversation)
- Performance impact from copying all files repeatedly
- Confusing UI with many micro-revisions
- Doesn't match user mental model

#### Option B: Revision Per Conversation (PARTIAL)

Create one revision when a chat conversation ends or user explicitly saves.

**Pros**:
- Batches related changes together
- More manageable revision count
- Clearer relationship between conversation and changes

**Cons**:
- When is "conversation end"? Hard to detect
- User might leave mid-conversation
- Could lose changes if not explicitly saved

#### Option C: Explicit User Action (RECOMMENDED)

Add a "Create Checkpoint" button that creates a revision on demand.

**Pros**:
- User controls when to snapshot
- No automatic revision explosion
- Matches existing "Proceed" mental model
- Simple to implement

**Cons**:
- User might forget to checkpoint
- Less automatic protection

#### Option D: Hybrid - Content Pending + Explicit Commit (RECOMMENDED)

Leverage existing `content_pending` column:
1. AI changes go to `content_pending` (already happens)
2. `content` remains unchanged until user "commits"
3. "Commit Changes" button creates revision and moves `content_pending` → `content`

**Pros**:
- Non-destructive changes
- User can discard pending changes
- Creates revision at commit time
- Uses existing database pattern

**Cons**:
- Requires UI to show pending vs committed state
- Need to handle what happens if user navigates away

### Recommended Architecture (Option D)

```
┌─────────────────────────────────────────────────────────────────┐
│                    REVISED AI SDK FLOW                           │
│                                                                  │
│  1. AI calls textEditor tool                                     │
│     └── Go endpoint updates `content_pending` column             │
│     └── `content` column unchanged                               │
│     └── UI shows diff between content and content_pending        │
│                                                                  │
│  2. User sees "Pending Changes" indicator                        │
│     └── Can preview changes in file explorer                     │
│     └── Can continue chatting for more changes                   │
│                                                                  │
│  3. User clicks "Commit Changes" button                          │
│     └── createRevisionFromPending(workspaceId)                   │
│     └── New revision created                                     │
│     └── All files copied with content_pending → content          │
│     └── content_pending cleared                                  │
│                                                                  │
│  4. User can also "Discard Changes"                              │
│     └── content_pending cleared                                  │
│     └── No revision created                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Sketch

**New Server Action**: `createRevisionFromPendingAction()`

```typescript
// lib/workspace/actions/create-revision-from-pending.ts
export async function createRevisionFromPendingAction(
  session: Session,
  workspaceId: string
): Promise<Workspace> {
  const db = getDB(await getParam("DB_URI"));

  await db.query('BEGIN');

  try {
    // 1. Get current revision number
    const currentRev = await db.query(
      `SELECT current_revision_number FROM workspace WHERE id = $1`,
      [workspaceId]
    );
    const prevRevNum = currentRev.rows[0].current_revision_number;
    const newRevNum = prevRevNum + 1;

    // 2. Create new revision row
    await db.query(
      `INSERT INTO workspace_revision (workspace_id, revision_number, created_at, created_by_user_id, created_type, is_complete)
       VALUES ($1, $2, NOW(), $3, 'ai_sdk', true)`,
      [workspaceId, newRevNum, session.user.id]
    );

    // 3. Copy charts to new revision
    await db.query(
      `INSERT INTO workspace_chart (id, revision_number, workspace_id, name)
       SELECT id, $2, workspace_id, name
       FROM workspace_chart
       WHERE workspace_id = $1 AND revision_number = $3`,
      [workspaceId, newRevNum, prevRevNum]
    );

    // 4. Copy files to new revision WITH content_pending → content
    await db.query(
      `INSERT INTO workspace_file (id, revision_number, chart_id, workspace_id, file_path, content, embeddings)
       SELECT id, $2, chart_id, workspace_id, file_path,
              COALESCE(content_pending, content),  -- Use pending if exists
              embeddings
       FROM workspace_file
       WHERE workspace_id = $1 AND revision_number = $3`,
      [workspaceId, newRevNum, prevRevNum]
    );

    // 5. Clear content_pending on old revision
    await db.query(
      `UPDATE workspace_file SET content_pending = NULL
       WHERE workspace_id = $1 AND revision_number = $2`,
      [workspaceId, prevRevNum]
    );

    // 6. Update current revision
    await db.query(
      `UPDATE workspace SET current_revision_number = $1 WHERE id = $2`,
      [newRevNum, workspaceId]
    );

    await db.query('COMMIT');

    return await getWorkspace(workspaceId);
  } catch (err) {
    await db.query('ROLLBACK');
    throw err;
  }
}
```

**UI Changes**:
- Add "Pending Changes" badge when `content_pending` exists
- Add "Commit Changes" and "Discard Changes" buttons
- Show diff view for pending changes

---

## Feature 2: Centrifugo Real-Time Integration

### Problem Statement

The main path uses Centrifugo WebSocket for real-time updates:
- Chat message streaming
- Plan progress
- File changes
- Render status

The test path uses AI SDK streaming directly. When AI modifies files, the file explorer doesn't update in real-time.

### Existing Pattern Study

**File**: `hooks/useCentrifugo.ts`

Event types handled:
- `chatmessage-updated` - Chat streaming
- `revision-created` - Refresh workspace after revision
- `artifact-updated` - File changes
- `plan-updated` - Plan progress
- `render-stream` - Render results
- `conversion-file` - Conversion updates

**Channel Format**: `{workspaceId}#{userId}`

**Server-Side Publishing**: Go worker publishes via Centrifugo API after database changes.

### Architectural Options

#### Option A: Publish from Go Endpoint (RECOMMENDED)

When `textEditor` Go endpoint modifies a file, publish to Centrifugo.

**Implementation**:
```go
// In pkg/api/handlers/editor.go
func TextEditor(w http.ResponseWriter, r *http.Request) {
    // ... existing file modification ...

    // After successful modification, publish event
    realtime.PublishArtifactUpdated(r.Context(), workspaceID, userID, file)
}
```

**Pros**:
- Reuses existing Centrifugo infrastructure
- File explorer updates immediately
- Consistent event model

**Cons**:
- Need to pass userID to Go endpoint
- Additional network call to Centrifugo

#### Option B: Polling from Client

Client periodically fetches workspace data.

**Pros**:
- Simple to implement
- No Centrifugo changes needed

**Cons**:
- Laggy updates
- Unnecessary network traffic
- Poor user experience

#### Option C: AI SDK Event + Centrifugo Bridge

Use AI SDK's tool result to trigger client-side update.

**Pros**:
- No server changes needed

**Cons**:
- Doesn't update other clients/tabs
- Inconsistent with main path

### Recommended Architecture (Option A)

```
┌─────────────────────────────────────────────────────────────────┐
│                    CENTRIFUGO INTEGRATION                        │
│                                                                  │
│  1. AI SDK calls textEditor tool                                 │
│     └── Go endpoint modifies file                                │
│     └── Go endpoint publishes to Centrifugo:                     │
│         {                                                        │
│           eventType: "artifact-updated",                         │
│           workspaceId: "xxx",                                    │
│           file: { id, filePath, content, content_pending }       │
│         }                                                        │
│                                                                  │
│  2. useCentrifugo hook receives event                            │
│     └── handleArtifactUpdated() updates workspace atom           │
│     └── FileBrowser re-renders with new file                     │
│                                                                  │
│  3. Test path already imports useCentrifugo (PR1.7 adds it)      │
│     └── Subscribes to workspace channel                          │
│     └── Receives all events like main path                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Sketch

**Go Changes**:
```go
// pkg/api/handlers/editor.go
import "chartsmith/pkg/realtime"

func TextEditor(w http.ResponseWriter, r *http.Request) {
    // ... existing implementation ...

    // After successful file operation
    if success {
        // Extract userID from auth header (already validated)
        userID := r.Context().Value("userID").(string)

        // Publish artifact update
        realtime.PublishArtifactUpdated(ctx, req.WorkspaceID, userID, &realtime.ArtifactUpdate{
            ID:              file.ID,
            RevisionNumber:  file.RevisionNumber,
            FilePath:        file.FilePath,
            Content:         file.Content,
            ContentPending:  file.ContentPending,
            ChartID:         file.ChartID,
        })
    }
}
```

**TypeScript Changes**:
```typescript
// In test-ai-chat client component
import { useCentrifugo } from "@/hooks/useCentrifugo";

export function TestAIChatClient({ workspace, session }) {
  // ... existing code ...

  // Add Centrifugo integration
  const { isReconnecting } = useCentrifugo({ session });

  // File explorer now updates automatically via atom changes
}
```

---

## Feature 3: Plan Workflow Integration

### Problem Statement

The main path proposes multi-file Plans that users can review before applying:
- Plan shows description and affected files
- User can ask clarifying questions
- User clicks "Proceed" to apply all changes
- Creates a new revision atomically

The test path makes changes directly via `textEditor` without Plan review.

### Existing Pattern Study

**Database Schema**: `workspace_plan` table
```yaml
columns:
  - id: text (primary key)
  - workspace_id: text
  - chat_message_ids: text[]
  - status: text (planning, pending, review, applying, applied, ignored)
  - description: text
  - charts_affected: text[]
  - files_affected: text[]
  - proceed_at: timestamp
```

**Plan Flow**:
1. Go worker creates Plan with status `planning`
2. LLM generates description and file changes
3. Status changes to `review`
4. User sees `PlanChatMessage` component
5. User clicks "Proceed"
6. `createRevisionAction()` called
7. Status changes to `applying`, then `applied`

**Component**: `components/PlanChatMessage.tsx`
- Shows Plan description (ReactMarkdown)
- Shows affected files with status icons
- Shows "Proceed" and feedback buttons
- Handles chat input for clarifications

### Architectural Options

#### Option A: Hybrid Plan Mode (RECOMMENDED)

Add an optional "Plan Mode" to AI SDK path where:
1. User can request "Plan this change" prefix
2. AI proposes changes but doesn't execute immediately
3. Changes displayed in Plan-like UI
4. User can approve to apply all changes

**Pros**:
- Gives users review capability
- Doesn't require full Go integration
- Can reuse Plan UI components

**Cons**:
- Different mechanism than main path
- Two ways to propose plans

#### Option B: Route to Go Worker

When AI SDK determines a Plan is needed, enqueue to Go worker.

**Pros**:
- Single planning mechanism
- Uses proven Plan system

**Cons**:
- Breaks AI SDK flow
- Requires intent detection
- Complex handoff

#### Option C: Client-Side Plan Building

AI SDK collects changes client-side, displays for review, then applies.

**Pros**:
- Stays in AI SDK flow
- No server changes

**Cons**:
- Doesn't persist Plan to database
- Can't share/review across sessions

### Recommended Architecture (Option A)

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAN MODE FOR AI SDK                          │
│                                                                  │
│  1. User message indicates planning:                             │
│     - "Plan the changes to add PostgreSQL"                       │
│     - "Propose a refactor of the deployment"                     │
│                                                                  │
│  2. System prompt instructs AI to use "planProposal" tool:       │
│     - Tool collects: description, file changes                   │
│     - Does NOT execute changes immediately                       │
│     - Returns structured proposal                                │
│                                                                  │
│  3. Client renders PlanProposalCard:                             │
│     - Shows description                                          │
│     - Lists files to create/modify                               │
│     - "Apply Changes" / "Discard" buttons                        │
│                                                                  │
│  4. On Apply:                                                    │
│     - Creates Plan record in database (optional)                 │
│     - Executes each file change via textEditor                   │
│     - Creates revision after all changes                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Sketch

**New Tool**: `planProposal`

```typescript
// lib/ai/tools/planProposal.ts
export function createPlanProposalTool(workspaceId: string) {
  return tool({
    description: 'Propose a multi-file change plan for user review before execution',
    parameters: z.object({
      description: z.string().describe('Explanation of the proposed changes'),
      changes: z.array(z.object({
        action: z.enum(['create', 'modify', 'delete']),
        path: z.string(),
        content: z.string().optional(),
        oldStr: z.string().optional(),
        newStr: z.string().optional(),
      })),
    }),
    execute: async ({ description, changes }) => {
      // Return proposal for client rendering, don't execute
      return {
        type: 'plan_proposal',
        workspaceId,
        description,
        changes,
        status: 'pending_review',
      };
    },
  });
}
```

**Client Component**: `PlanProposalCard`

```typescript
// components/chat/PlanProposalCard.tsx
export function PlanProposalCard({ proposal, onApply, onDiscard }) {
  return (
    <div className="border rounded-lg p-4">
      <h3 className="font-semibold">Proposed Changes</h3>
      <ReactMarkdown>{proposal.description}</ReactMarkdown>

      <div className="mt-4 space-y-2">
        {proposal.changes.map((change, i) => (
          <div key={i} className="flex items-center gap-2">
            <Icon action={change.action} />
            <span className="font-mono text-sm">{change.path}</span>
          </div>
        ))}
      </div>

      <div className="mt-4 flex gap-2">
        <Button onClick={onApply}>Apply Changes</Button>
        <Button variant="ghost" onClick={onDiscard}>Discard</Button>
      </div>
    </div>
  );
}
```

---

## Implementation Priority

| Priority | Feature | Complexity | Dependencies |
|----------|---------|------------|--------------|
| 1 | Revision Tracking (Option D) | Medium | PR1.6 complete |
| 2 | Centrifugo Integration | Medium | Revision tracking |
| 3 | Plan Workflow | High | Both above |

**Note**: These features can be implemented incrementally. Start with revision tracking to provide change history, then add real-time updates, finally add plan workflow.

---

## Open Questions for Implementation

### Revision Tracking
1. Should "Commit Changes" auto-trigger on conversation end?
2. How to handle uncommitted changes when user navigates away?
3. Should there be a maximum pending changes limit?

### Centrifugo Integration
1. Should Go endpoints require userID in request body or extract from auth?
2. Rate limiting for high-frequency tool calls?
3. Batch file updates into single Centrifugo event?

### Plan Workflow
1. Should Plans persist to database even from AI SDK?
2. How to handle mixed mode (some direct, some planned)?
3. Should AI auto-detect when planning is appropriate?

---

## Dependencies

### External Dependencies
- PR1.6 merged and deployed
- Go Centrifugo client (`pkg/realtime/`)
- Existing Plan infrastructure

### Internal Dependencies
- `content_pending` column in `workspace_file`
- `useCentrifugo` hook
- `PlanChatMessage` component patterns

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PR1.7_Product_PRD.md` | Product requirements |
| `PR1.6_Tech_PRD.md` | PR1.6 specification (prerequisite) |
| `lib/workspace/workspace.ts` | Revision creation pattern |
| `hooks/useCentrifugo.ts` | Real-time event handling |
| `components/PlanChatMessage.tsx` | Plan UI patterns |

---

*Document End - PR1.7: Deep System Integration Technical Specification*
