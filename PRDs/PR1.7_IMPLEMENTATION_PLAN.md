# PR1.7 Implementation Plan

**Version**: 1.1
**Status**: Ready for Implementation
**Prerequisite**: ✅ All prerequisites completed (Bug 1 ✅, Bug 2 ✅)
**Last Updated**: December 4, 2025

---

## Overview

This document provides step-by-step implementation details for PR1.7 features. Read `PR1.7_Tech_PRD.md` for architectural context and `PR1.7_Product_PRD.md` for requirements.

**Implementation Order**:
1. Prerequisite: Fix Bug 2 (inconsistent content handling)
2. Feature 1: Centrifugo Integration (enables real-time updates)
3. Feature 2: Pending Changes UI (shows pending state)
4. Feature 3: Revision Tracking (commit/discard workflow)
5. Feature 4: Plan Workflow (optional, lower priority)

---

## Prerequisite: Bug 2 - Inconsistent Content Handling ✅ COMPLETED

### Problem (Resolved)
`AddFileToChart()` writes to `content` directly, but `SetFileContentPending()` writes to `content_pending`. All AI changes should go to `content_pending` for consistent pending/commit workflow.

### Solution Implemented
Created new `AddFileToChartPending()` function in `pkg/workspace/chart.go:56-74` that writes to `content_pending` with empty `content`. The textEditor handler (`pkg/api/handlers/editor.go:156`) now calls this function for the `create` command.

**Why new function instead of modifying existing**: `AddFileToChart()` is also used by the K8s-to-Helm conversion workflow which needs direct `content` writes (not pending). Creating a separate function preserves that behavior.

### Verification (Already Passing)
```bash
# 1. Build succeeds
cd chartsmith && go build ./...

# 2. editor.go:156 calls AddFileToChartPending
grep -n "AddFileToChartPending" chartsmith/pkg/api/handlers/editor.go
# Output: 156:	err = workspace.AddFileToChartPending(...)

# 3. New files via AI go to content_pending
# Ask AI: "Create a new file called test.yaml"
# Check DB: content = '', content_pending = '<file content>'
```

---

## Feature 1: Centrifugo Integration

### Goal
Publish `artifact-updated` events from Go textEditor handler so file explorer updates in real-time.

### Phase 1.0: Modify AddFileToChartPending to Return fileID

**File**: `chartsmith/pkg/workspace/chart.go`

**Current signature** (line 58):
```go
func AddFileToChartPending(ctx context.Context, chartID string, workspaceID string, revisionNumber int, path string, contentPending string) error
```

**Change to**:
```go
func AddFileToChartPending(ctx context.Context, chartID string, workspaceID string, revisionNumber int, path string, contentPending string) (string, error)
```

**Full updated function**:
```go
// AddFileToChartPending creates a new file with content in content_pending column.
// Use this for AI SDK paths where content should be staged for review before committing.
// Returns the generated fileID.
func AddFileToChartPending(ctx context.Context, chartID string, workspaceID string, revisionNumber int, path string, contentPending string) (string, error) {
	conn := persistence.MustGetPooledPostgresSession()
	defer conn.Release()

	fileID, err := securerandom.Hex(12)
	if err != nil {
		return "", fmt.Errorf("failed to generate random ID: %w", err)
	}

	query := `INSERT INTO workspace_file (id, revision_number, chart_id, workspace_id, file_path, content, content_pending) VALUES ($1, $2, $3, $4, $5, '', $6)`
	_, err = conn.Exec(ctx, query, fileID, revisionNumber, chartID, workspaceID, path, contentPending)
	if err != nil {
		return "", fmt.Errorf("failed to insert file: %w", err)
	}

	return fileID, nil
}
```

**Then update caller in** `chartsmith/pkg/api/handlers/editor.go` (line 156):

**Current**:
```go
err = workspace.AddFileToChartPending(ctx, chartID, req.WorkspaceID, req.RevisionNumber, req.Path, req.Content)
```

**Change to**:
```go
fileID, err := workspace.AddFileToChartPending(ctx, chartID, req.WorkspaceID, req.RevisionNumber, req.Path, req.Content)
```

### Phase 1.1: Add Centrifugo Publishing to textEditor Handler

**File**: `chartsmith/pkg/api/handlers/editor.go`

**Step 1**: Add imports at top of file:
```go
import (
    // ... existing imports ...
    "chartsmith/pkg/realtime"
    realtimetypes "chartsmith/pkg/realtime/types"
    workspacetypes "chartsmith/pkg/workspace/types"
)
```

**Step 2**: Create helper function (add after imports, before `TextEditor` function):
```go
func publishArtifactUpdate(ctx context.Context, workspaceID string, revisionNumber int, chartID, fileID, filePath, content string, contentPending *string) {
    userIDs, err := workspace.ListUserIDsForWorkspace(ctx, workspaceID)
    if err != nil {
        slog.Error("Failed to get user IDs for realtime", "error", err, "workspaceID", workspaceID)
        return
    }

    recipient := realtimetypes.Recipient{UserIDs: userIDs}
    event := &realtimetypes.ArtifactUpdatedEvent{
        WorkspaceID: workspaceID,
        WorkspaceFile: &workspacetypes.File{
            ID:             fileID,
            RevisionNumber: revisionNumber,
            ChartID:        chartID,
            WorkspaceID:    workspaceID,
            FilePath:       filePath,
            Content:        content,
            ContentPending: contentPending,
        },
    }

    if err := realtime.SendEvent(ctx, recipient, event); err != nil {
        slog.Error("Failed to send artifact update", "error", err, "workspaceID", workspaceID)
    }
}
```

**Step 3**: Call helper after successful `create` command (in `handleCreate` function, after `AddFileToChartPending` call):
```go
// After successful AddFileToChartPending (around line 156-160)
// Note: Phase 1.0 updated AddFileToChartPending to return (string, error)
fileID, err := workspace.AddFileToChartPending(ctx, chartID, req.WorkspaceID, req.RevisionNumber, req.Path, req.Content)
if err != nil {
    logger.Debug("Failed to create file", zap.Error(err))
    writeInternalError(w, "Failed to create file")
    return
}

// ADD THIS: Publish realtime update
publishArtifactUpdate(ctx, req.WorkspaceID, req.RevisionNumber, chartID, fileID, req.Path, "", &req.Content)

// Continue with success response...
writeJSON(w, http.StatusOK, TextEditorResponse{
    Success: true,
    Content: req.Content,
    Message: "File created successfully",
})
```

**Step 4**: Call helper after successful `str_replace` command (in `handleStrReplace` function, after `SetFileContentPending` call):
```go
// After successful SetFileContentPending (around line 245-250)
err = workspace.SetFileContentPending(ctx, req.Path, req.RevisionNumber, foundFile.chartID, req.WorkspaceID, newContent)
if err != nil {
    // ... error handling ...
}

// ADD THIS: Publish realtime update
// Need to get file ID - query for it
fileID, err := workspace.GetFileIDByPath(ctx, req.WorkspaceID, req.RevisionNumber, req.Path)
if err != nil {
    slog.Error("Failed to get file ID for realtime", "error", err)
} else {
    publishArtifactUpdate(ctx, req.WorkspaceID, req.RevisionNumber, foundFile.chartID, fileID, req.Path, foundFile.content, &newContent)
}

// Continue with success response...
```

**Step 5**: Add `GetFileIDByPath` helper function to `chartsmith/pkg/workspace/file.go`:
```go
func GetFileIDByPath(ctx context.Context, workspaceID string, revisionNumber int, filePath string) (string, error) {
    db := persistence.GetDB()
    var fileID string
    err := db.QueryRowContext(ctx,
        `SELECT id FROM workspace_file WHERE workspace_id = $1 AND revision_number = $2 AND file_path = $3`,
        workspaceID, revisionNumber, filePath,
    ).Scan(&fileID)
    if err != nil {
        return "", fmt.Errorf("failed to get file ID: %w", err)
    }
    return fileID, nil
}
```

### Phase 1.2: Integrate useCentrifugo in Test-AI-Chat

**File**: `chartsmith/chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Step 1**: Add import:
```typescript
import { useCentrifugo } from "@/hooks/useCentrifugo";
```

**Step 2**: Add hook call inside `TestAIChatClient` component (after other hooks, around line 58):
```typescript
// Add Centrifugo for real-time file updates
const { isReconnecting } = useCentrifugo({ session });
```

**Step 3**: Remove the refetch-after-tool-completion useEffect (lines 153-169). Delete this entire block:
```typescript
// DELETE THIS BLOCK:
// Refetch workspace when AI tool calls complete to update file explorer
// This is a simple solution for PR1.6; PR1.7 will use Centrifugo for real-time
useEffect(() => {
  const lastMessage = messages[messages.length - 1];
  // ... etc
}, [messages, status, session, workspace.id, setWorkspace]);
```

### Verification
```bash
# 1. Start app with Go backend and Centrifugo running
cd chartsmith && make dev

# 2. Open test-ai-chat in browser
# 3. Open Network tab, filter for WebSocket

# 4. Ask AI: "Create a new file called realtime-test.yaml"

# 5. Verify:
# - WebSocket shows artifact-updated event
# - File appears in explorer WITHOUT page refresh
# - Console shows no refetch calls
```

---

## Feature 2: Pending Changes UI

### Goal
Show visual indicator when files have pending changes, before commit/discard buttons exist.

### Phase 2.1: Add Pending Changes Indicator

**File**: `chartsmith/chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Step 1**: Import the pending files atom:
```typescript
import {
  workspaceAtom,
  selectedFileAtom,
  editorViewAtom,
  renderedFilesAtom,
  allFilesWithContentPendingAtom  // ADD THIS
} from "@/atoms/workspace";
```

**Step 2**: Use the atom in component:
```typescript
const [filesWithPending] = useAtom(allFilesWithContentPendingAtom);
const hasPendingChanges = filesWithPending.length > 0;
```

**Step 3**: Add indicator in the UI (in the left sidebar area, after the nav icons, around line 305):
```typescript
{/* Pending changes indicator */}
{hasPendingChanges && (
  <div className={`mx-2 px-2 py-1 rounded text-xs ${
    theme === "dark" ? "bg-yellow-900/30 text-yellow-400" : "bg-yellow-100 text-yellow-700"
  }`}>
    {filesWithPending.length} pending
  </div>
)}
```

### Phase 2.2: Show Pending Badge on Files in FileBrowser

**File**: `chartsmith/chartsmith-app/components/FileBrowser.tsx`

This may already show pending state. Check if `contentPending` is used in the rendering. If not:

**Step 1**: Find where file items are rendered (look for `file.filePath` mapping)

**Step 2**: Add pending indicator:
```typescript
{file.contentPending && (
  <span className="ml-1 text-yellow-500 text-xs">●</span>
)}
```

### Verification
```bash
# 1. Create a file or modify via str_replace
# 2. Verify yellow indicator appears in sidebar
# 3. Verify file in explorer shows pending dot
# 4. Refresh page - pending state should persist (from DB)
```

---

## Feature 3: Revision Tracking (Commit/Discard)

### Goal
Add "Commit Changes" and "Discard Changes" buttons that create revisions or clear pending state.

### Phase 3.1: Create Commit Action

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/commit-pending-changes.ts` (NEW FILE)

```typescript
"use server";

import { Session } from "@/lib/types/session";
import { getDB, getParam } from "@/lib/db";
import { getWorkspace } from "../workspace";
import { Workspace } from "@/lib/types/workspace";

export async function commitPendingChangesAction(
  session: Session,
  workspaceId: string
): Promise<Workspace> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const db = getDB(await getParam("DB_URI"));
  const client = await db.connect();

  try {
    await client.query("BEGIN");

    // 1. Get current revision number
    const currentRevResult = await client.query(
      `SELECT current_revision_number FROM workspace WHERE id = $1`,
      [workspaceId]
    );
    if (currentRevResult.rows.length === 0) {
      throw new Error("Workspace not found");
    }
    const prevRevNum = currentRevResult.rows[0].current_revision_number;
    const newRevNum = prevRevNum + 1;

    // 2. Create new revision row
    await client.query(
      `INSERT INTO workspace_revision (workspace_id, revision_number, created_at, created_by_user_id, created_type, is_complete, is_rendered)
       VALUES ($1, $2, NOW(), $3, 'ai_sdk_commit', true, false)`,
      [workspaceId, newRevNum, session.user.id]
    );

    // 3. Copy charts to new revision
    await client.query(
      `INSERT INTO workspace_chart (id, revision_number, workspace_id, name)
       SELECT id, $2, workspace_id, name
       FROM workspace_chart
       WHERE workspace_id = $1 AND revision_number = $3`,
      [workspaceId, newRevNum, prevRevNum]
    );

    // 4. Copy files to new revision WITH content_pending → content
    await client.query(
      `INSERT INTO workspace_file (id, revision_number, chart_id, workspace_id, file_path, content, embeddings)
       SELECT id, $2, chart_id, workspace_id, file_path,
              COALESCE(NULLIF(content_pending, ''), content),
              embeddings
       FROM workspace_file
       WHERE workspace_id = $1 AND revision_number = $3`,
      [workspaceId, newRevNum, prevRevNum]
    );

    // 5. Clear content_pending on old revision (cleanup)
    await client.query(
      `UPDATE workspace_file SET content_pending = NULL
       WHERE workspace_id = $1 AND revision_number = $2`,
      [workspaceId, prevRevNum]
    );

    // 6. Update workspace current revision
    await client.query(
      `UPDATE workspace SET current_revision_number = $1, last_updated_at = NOW() WHERE id = $2`,
      [newRevNum, workspaceId]
    );

    await client.query("COMMIT");

    // Return updated workspace
    const workspace = await getWorkspace(workspaceId);
    if (!workspace) {
      throw new Error("Failed to fetch updated workspace");
    }
    return workspace;

  } catch (err) {
    await client.query("ROLLBACK");
    throw err;
  } finally {
    client.release();
  }
}
```

### Phase 3.2: Create Discard Action

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/discard-pending-changes.ts` (NEW FILE)

```typescript
"use server";

import { Session } from "@/lib/types/session";
import { getDB, getParam } from "@/lib/db";
import { getWorkspace } from "../workspace";
import { Workspace } from "@/lib/types/workspace";

export async function discardPendingChangesAction(
  session: Session,
  workspaceId: string
): Promise<Workspace> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const db = getDB(await getParam("DB_URI"));

  // Clear all content_pending for current revision
  const revResult = await db.query(
    `SELECT current_revision_number FROM workspace WHERE id = $1`,
    [workspaceId]
  );
  if (revResult.rows.length === 0) {
    throw new Error("Workspace not found");
  }
  const currentRevNum = revResult.rows[0].current_revision_number;

  await db.query(
    `UPDATE workspace_file SET content_pending = NULL
     WHERE workspace_id = $1 AND revision_number = $2 AND content_pending IS NOT NULL`,
    [workspaceId, currentRevNum]
  );

  // Delete files that only have content_pending (created but not committed)
  await db.query(
    `DELETE FROM workspace_file
     WHERE workspace_id = $1 AND revision_number = $2 AND (content IS NULL OR content = '')`,
    [workspaceId, currentRevNum]
  );

  // Return updated workspace
  const workspace = await getWorkspace(workspaceId);
  if (!workspace) {
    throw new Error("Failed to fetch updated workspace");
  }
  return workspace;
}
```

### Phase 3.3: Add UI Buttons

**File**: `chartsmith/chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Step 1**: Import actions:
```typescript
import { commitPendingChangesAction } from "@/lib/workspace/actions/commit-pending-changes";
import { discardPendingChangesAction } from "@/lib/workspace/actions/discard-pending-changes";
```

**Step 2**: Add state for loading:
```typescript
const [isCommitting, setIsCommitting] = useState(false);
const [isDiscarding, setIsDiscarding] = useState(false);
```

**Step 3**: Add handlers:
```typescript
const handleCommit = async () => {
  if (isCommitting || !hasPendingChanges) return;
  setIsCommitting(true);
  try {
    const updated = await commitPendingChangesAction(session, workspace.id);
    setWorkspace(updated);
  } catch (err) {
    console.error("Failed to commit changes:", err);
  } finally {
    setIsCommitting(false);
  }
};

const handleDiscard = async () => {
  if (isDiscarding || !hasPendingChanges) return;
  if (!confirm("Discard all pending changes? This cannot be undone.")) return;
  setIsDiscarding(true);
  try {
    const updated = await discardPendingChangesAction(session, workspace.id);
    setWorkspace(updated);
  } catch (err) {
    console.error("Failed to discard changes:", err);
  } finally {
    setIsDiscarding(false);
  }
};
```

**Step 4**: Add buttons in UI (replace the simple indicator from Phase 2.1):
```typescript
{/* Pending changes controls */}
{hasPendingChanges && (
  <div className={`mx-2 p-2 rounded ${
    theme === "dark" ? "bg-dark-border/40" : "bg-gray-100"
  }`}>
    <div className={`text-xs mb-2 ${
      theme === "dark" ? "text-gray-400" : "text-gray-600"
    }`}>
      {filesWithPending.length} pending change{filesWithPending.length !== 1 ? 's' : ''}
    </div>
    <div className="flex gap-1">
      <button
        onClick={handleCommit}
        disabled={isCommitting}
        className={`flex-1 px-2 py-1 text-xs rounded ${
          theme === "dark"
            ? "bg-green-900/50 text-green-400 hover:bg-green-900/70"
            : "bg-green-100 text-green-700 hover:bg-green-200"
        } disabled:opacity-50`}
      >
        {isCommitting ? "..." : "Commit"}
      </button>
      <button
        onClick={handleDiscard}
        disabled={isDiscarding}
        className={`flex-1 px-2 py-1 text-xs rounded ${
          theme === "dark"
            ? "bg-red-900/50 text-red-400 hover:bg-red-900/70"
            : "bg-red-100 text-red-700 hover:bg-red-200"
        } disabled:opacity-50`}
      >
        {isDiscarding ? "..." : "Discard"}
      </button>
    </div>
  </div>
)}
```

### Verification
```bash
# 1. Create files via AI, verify pending indicator shows
# 2. Click "Commit" - verify:
#    - Revision number increments (shown in assistant messages)
#    - Pending indicator disappears
#    - Files still visible in explorer
# 3. Create more files, click "Discard" - verify:
#    - Pending indicator disappears
#    - New files removed from explorer
#    - Revision number unchanged
# 4. Check database:
psql -c "SELECT revision_number, created_type FROM workspace_revision WHERE workspace_id = 'xxx' ORDER BY revision_number DESC LIMIT 3;"
# Should show 'ai_sdk_commit' for committed revisions
```

---

## Feature 4: Plan Workflow (Lower Priority)

### Goal
Add optional "Plan Mode" where AI proposes changes for review before execution.

**Note**: This is a larger feature. Implement after Features 1-3 are stable.

### Phase 4.1: Create planProposal Tool

**File**: `chartsmith/chartsmith-app/lib/ai/tools/planProposal.ts` (NEW FILE)

```typescript
import { tool } from "ai";
import { z } from "zod";

export function createPlanProposalTool(workspaceId: string) {
  return tool({
    description: "Propose a set of file changes for user review before execution. Use this when the user asks to 'plan' changes or when making multiple related file modifications.",
    parameters: z.object({
      description: z.string().describe("A markdown description explaining what changes will be made and why"),
      changes: z.array(z.object({
        action: z.enum(["create", "modify", "delete"]).describe("The type of change"),
        path: z.string().describe("File path relative to chart root"),
        content: z.string().optional().describe("New file content for create, or replacement content for modify"),
        oldStr: z.string().optional().describe("For modify: the text to find and replace"),
        newStr: z.string().optional().describe("For modify: the replacement text"),
      })).describe("List of file changes to propose"),
    }),
    execute: async ({ description, changes }) => {
      // Return proposal for client-side rendering - don't execute changes yet
      return {
        type: "plan_proposal" as const,
        workspaceId,
        description,
        changes,
        status: "pending_review" as const,
        createdAt: new Date().toISOString(),
      };
    },
  });
}
```

### Phase 4.2: Register Tool

**File**: `chartsmith/chartsmith-app/lib/ai/tools/index.ts`

Add to `createTools()` function:
```typescript
import { createPlanProposalTool } from "./planProposal";

export function createTools(...) {
  return {
    // ... existing tools ...
    planProposal: createPlanProposalTool(workspaceId),
  };
}
```

### Phase 4.3: Create PlanProposalCard Component

**File**: `chartsmith/chartsmith-app/components/chat/PlanProposalCard.tsx` (NEW FILE)

```typescript
"use client";

import { useState } from "react";
import { useTheme } from "@/contexts/ThemeContext";
import ReactMarkdown from "react-markdown";
import { Plus, Pencil, Trash2, Loader2 } from "lucide-react";

interface PlanChange {
  action: "create" | "modify" | "delete";
  path: string;
  content?: string;
  oldStr?: string;
  newStr?: string;
}

interface PlanProposal {
  type: "plan_proposal";
  workspaceId: string;
  description: string;
  changes: PlanChange[];
  status: "pending_review" | "applying" | "applied" | "discarded";
}

interface PlanProposalCardProps {
  proposal: PlanProposal;
  onApply: (changes: PlanChange[]) => Promise<void>;
  onDiscard: () => void;
}

export function PlanProposalCard({ proposal, onApply, onDiscard }: PlanProposalCardProps) {
  const { theme } = useTheme();
  const [isApplying, setIsApplying] = useState(false);
  const [status, setStatus] = useState(proposal.status);

  const handleApply = async () => {
    setIsApplying(true);
    setStatus("applying");
    try {
      await onApply(proposal.changes);
      setStatus("applied");
    } catch (err) {
      console.error("Failed to apply changes:", err);
      setStatus("pending_review");
    } finally {
      setIsApplying(false);
    }
  };

  const handleDiscard = () => {
    setStatus("discarded");
    onDiscard();
  };

  const getActionIcon = (action: string) => {
    switch (action) {
      case "create": return <Plus className="w-3 h-3 text-green-500" />;
      case "modify": return <Pencil className="w-3 h-3 text-yellow-500" />;
      case "delete": return <Trash2 className="w-3 h-3 text-red-500" />;
      default: return null;
    }
  };

  return (
    <div className={`rounded-lg border p-4 ${
      theme === "dark" ? "bg-dark-surface border-dark-border" : "bg-white border-gray-200"
    }`}>
      <div className="flex items-center justify-between mb-3">
        <h3 className={`font-semibold text-sm ${
          theme === "dark" ? "text-gray-200" : "text-gray-800"
        }`}>
          Proposed Changes
        </h3>
        <span className={`text-xs px-2 py-0.5 rounded ${
          status === "applied" ? "bg-green-500/20 text-green-400" :
          status === "discarded" ? "bg-gray-500/20 text-gray-400" :
          status === "applying" ? "bg-yellow-500/20 text-yellow-400" :
          "bg-blue-500/20 text-blue-400"
        }`}>
          {status.replace("_", " ")}
        </span>
      </div>

      <div className={`text-xs mb-4 ${theme === "dark" ? "text-gray-300" : "text-gray-600"}`}>
        <ReactMarkdown>{proposal.description}</ReactMarkdown>
      </div>

      <div className="space-y-1 mb-4">
        {proposal.changes.map((change, i) => (
          <div key={i} className={`flex items-center gap-2 text-xs px-2 py-1 rounded ${
            theme === "dark" ? "bg-dark-border/40" : "bg-gray-50"
          }`}>
            {getActionIcon(change.action)}
            <span className="font-mono">{change.path}</span>
          </div>
        ))}
      </div>

      {status === "pending_review" && (
        <div className="flex gap-2">
          <button
            onClick={handleApply}
            disabled={isApplying}
            className={`flex-1 px-3 py-1.5 text-xs rounded font-medium ${
              theme === "dark"
                ? "bg-primary text-white hover:bg-primary/80"
                : "bg-primary text-white hover:bg-primary/90"
            } disabled:opacity-50`}
          >
            {isApplying ? <Loader2 className="w-3 h-3 animate-spin mx-auto" /> : "Apply Changes"}
          </button>
          <button
            onClick={handleDiscard}
            className={`px-3 py-1.5 text-xs rounded ${
              theme === "dark"
                ? "bg-dark-border text-gray-300 hover:bg-dark-border/80"
                : "bg-gray-100 text-gray-700 hover:bg-gray-200"
            }`}
          >
            Discard
          </button>
        </div>
      )}
    </div>
  );
}
```

### Phase 4.4: Create Apply Plan Helper

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/apply-plan-changes.ts` (NEW FILE)

```typescript
"use server";

import { Session } from "@/lib/types/session";

interface PlanChange {
  action: "create" | "modify" | "delete";
  path: string;
  content?: string;
  oldStr?: string;
  newStr?: string;
}

export async function applyPlanChangesAction(
  session: Session,
  workspaceId: string,
  revisionNumber: number,
  changes: PlanChange[]
): Promise<{ success: boolean; errors: string[] }> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const errors: string[] = [];
  const editorApiUrl = process.env.GO_API_URL || "http://localhost:3333";

  for (const change of changes) {
    try {
      let body: Record<string, unknown>;

      switch (change.action) {
        case "create":
          body = {
            command: "create",
            workspaceId,
            revisionNumber,
            path: change.path,
            content: change.content || "",
          };
          break;
        case "modify":
          body = {
            command: "str_replace",
            workspaceId,
            revisionNumber,
            path: change.path,
            oldStr: change.oldStr,
            newStr: change.newStr,
          };
          break;
        case "delete":
          // Delete not yet supported by textEditor - skip with warning
          errors.push(`Delete not supported: ${change.path}`);
          continue;
        default:
          errors.push(`Unknown action: ${change.action}`);
          continue;
      }

      const response = await fetch(`${editorApiUrl}/api/tools/editor`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(body),
      });

      const result = await response.json();
      if (!result.success) {
        errors.push(`${change.path}: ${result.message}`);
      }
    } catch (err) {
      errors.push(`${change.path}: ${err instanceof Error ? err.message : "Unknown error"}`);
    }
  }

  return { success: errors.length === 0, errors };
}
```

### Phase 4.5: Render Plan Proposals in Chat

**File**: `chartsmith/chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

**Step 1**: Import the apply action:
```typescript
import { applyPlanChangesAction } from "@/lib/workspace/actions/apply-plan-changes";
```

**Step 2**: In the message rendering section, detect and render plan proposals:
```typescript
// Inside the message.parts?.map loop, add handling for tool results
if (part.type.startsWith('tool-') && part.toolName === 'planProposal' && part.state === 'result') {
  const proposal = part.result as PlanProposal;
  return (
    <PlanProposalCard
      key={i}
      proposal={proposal}
      onApply={async (changes) => {
        // Direct API calls - deterministic and efficient
        const result = await applyPlanChangesAction(
          session,
          workspace.id,
          workspace.currentRevisionNumber,
          changes
        );
        if (!result.success) {
          console.error("Some changes failed:", result.errors);
          // Optionally show errors to user
        }
        // Refresh workspace to show new files
        const updated = await getWorkspaceAction(session, workspace.id);
        if (updated) setWorkspace(updated);
      }}
      onDiscard={() => {
        // No action needed - card will show "discarded" status
      }}
    />
  );
}
```

### Phase 4.6: Update System Prompt

**File**: `chartsmith/chartsmith-app/lib/ai/prompts.ts`

Add guidance for when to use planProposal:
```typescript
export const CHARTSMITH_SYSTEM_PROMPT = `...existing prompt...

## Plan Mode
When the user asks you to "plan" changes, or when making multiple related file modifications (3+ files), use the planProposal tool instead of making direct changes. This allows the user to review and approve changes before they're applied.

Examples of when to use planProposal:
- "Plan the changes to add PostgreSQL"
- "Propose a refactor of the deployment"
- Making changes that affect multiple related files
`;
```

### Verification
```bash
# 1. Ask: "Plan adding a Redis deployment to my chart"
# 2. Verify plan proposal card appears with:
#    - Description
#    - List of files to create/modify
#    - Apply/Discard buttons
# 3. Click Apply, verify files are created
# 4. Test Discard, verify no changes made
```

---

## Success Criteria Checklist

### Prerequisite ✅ COMPLETED
- [x] `AddFileToChartPending()` function created (writes to `content_pending`)
- [x] `editor.go` uses `AddFileToChartPending()` for create command
- [x] New files show as pending in database

### Feature 1: Centrifugo
- [ ] `artifact-updated` events published from Go textEditor handler
- [ ] Test-ai-chat subscribes via `useCentrifugo`
- [ ] File explorer updates without page refresh
- [ ] Refetch workaround removed

### Feature 2: Pending UI
- [ ] Pending changes count shown in sidebar
- [ ] Files with pending changes marked in explorer

### Feature 3: Revision Tracking
- [ ] "Commit" button creates new revision
- [ ] "Discard" button clears pending without revision
- [ ] Revision number increments correctly
- [ ] Database shows `ai_sdk_commit` created_type

### Feature 4: Plan Workflow (Optional)
- [ ] `planProposal` tool registered
- [ ] Plan proposals render in chat
- [ ] Apply executes changes
- [ ] Discard cancels proposal

---

## File Summary

### New Files
- `lib/workspace/actions/commit-pending-changes.ts`
- `lib/workspace/actions/discard-pending-changes.ts`
- `lib/ai/tools/planProposal.ts` (optional - Feature 4)
- `lib/workspace/actions/apply-plan-changes.ts` (optional - Feature 4)
- `components/chat/PlanProposalCard.tsx` (optional - Feature 4)

### Modified Files
- `pkg/workspace/chart.go` - ✅ AddFileToChartPending exists; TODO: Update to return `(string, error)` (Feature 1, Phase 1.0)
- `pkg/api/handlers/editor.go` - ✅ Uses AddFileToChartPending; TODO: Capture fileID, add Centrifugo publishing (Feature 1)
- `pkg/workspace/file.go` - Add GetFileIDByPath helper (Feature 1)
- `app/test-ai-chat/[workspaceId]/client.tsx` - UI changes (Features 1-4)
- `lib/ai/tools/index.ts` - Register planProposal tool (optional - Feature 4)
- `lib/ai/prompts.ts` - Update system prompt (optional - Feature 4)
