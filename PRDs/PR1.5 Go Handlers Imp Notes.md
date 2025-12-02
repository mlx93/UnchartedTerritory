PR1.5 Go Handlers – Implementation Notes (Draft)
This document provides tiny, precise, implementation‑ready notes for each Go HTTP handler introduced in PR1.5. The goal is to guide Cursor/Claude Code when modifying the Go backend so it reuses existing workspace logic rather than re‑inventing it.
Overview
All handlers live under:
pkg/api/
    server.go
    handlers/
        charts.go
        editor.go
Handlers expose JSON‑based APIs called by AI SDK tools in /api/chat.
1. CreateChart Handler
File: pkg/api/handlers/charts.go
Function: func (h *ChartHandler) CreateChart(w http.ResponseWriter, r *http.Request)
Inputs (JSON)
{
  "name": "nginx-deployment",
  "description": "A basic nginx deployment",
  "type": "deployment",
  "includeIngress": false,
  "includeService": true
}
Implementation Notes (How to wire into current architecture)
1.	Parse the request into a CreateChartRequest struct.
2.	Reuse existing workspace‑creation logic, which currently lives in code paths invoked by the “New Chart” button.
3.	Functions you should call:
o	CreateWorkspace(ctx, userID, name)
o	InitializeWorkspaceFiles(workspaceID, type, opts)
o	CreateInitialRevision(workspaceID, files)
4.	Ensure YAML files follow the same structure as when created through the current UI.
5.	Trigger PostgreSQL NOTIFY events so the UI updates:
6.	Use the same function call worker code currently uses: workspace.Notify(ctx, workspaceID, "chart_created").
7.	Respond with JSON:
{
  "success": true,
  "workspaceID": "...",
  "revisionID": "...",
  "message": "Chart created successfully."
}
2. GetChartContext Handler
File: pkg/api/handlers/charts.go
Function: func (h *ChartHandler) GetChartContext(w http.ResponseWriter, r *http.Request)
Inputs
{
  "workspaceID": "..."
}
Implementation Steps
1.	Validate workspace ownership using existing auth utilities.
2.	Load workspace metadata, using existing functions: workspace.LoadByID(workspaceID).
3.	Load the latest revision: revision, err := revisions.GetLatest(workspaceID).
4.	Load rendered chart YAML:
5.	Use existing file handling code: files, _ := workspaceFiles.List(revision.ID).
6.	Construct a JSON payload summarizing:
7.	Top‑level resources
8.	Deployment/Service/Ingress presence
9.	Optional: number of templates, workload type
10.	Return JSON for the AI model to consume:
{
  "success": true,
  "overview": "... human readable summary ...",
  "files": [
    { "path": "templates/deployment.yaml", "content": "..." }
  ]
}
3. UpdateChart Handler
File: pkg/api/handlers/charts.go
Function: func (h *ChartHandler) UpdateChart(w http.ResponseWriter, r *http.Request)
Inputs
{
  "workspaceID": "...",
  "patch": {
      "file": "templates/deployment.yaml",
      "oldText": "replicas: 1",
      "newText": "replicas: 3"
  }
}
Implementation Steps
1.	Load the latest revision as in GetChartContext.
2.	Use internal patch / file update utilities:
3.	Those used by Go’s text_editor tool in pkg/agent/tools.
4.	Specifically call functions such as files.ApplyPatch(...) or reuse the exact logic used in the existing text‑edit tool so behavior is identical.
5.	Create a new revision with updated files using the existing revision creation workflow.
6.	Trigger a NOTIFY event:
7.	workspace.Notify(ctx, workspaceID, "revision_created").
8.	Return JSON:
{
  "success": true,
  "revisionID": "...",
  "message": "Chart updated."
}
4. TextEditor Handler
File: pkg/api/handlers/editor.go
Function: func (h *EditorHandler) Edit(w http.ResponseWriter, r *http.Request)
Inputs
{
  "workspaceID": "...",
  "filePath": "templates/deployment.yaml",
  "instructions": "Add resource limits to the container"
}
Implementation Steps
1.	Load file content, using existing FS helpers.
2.	Use the existing text editor logic, currently inside pkg/agent/tools/text_editor.go.
3.	Instead of letting Go call the LLM, call the tool logic alone:
4.	Extract the non‑LLM editing utilities (string manipulation, AST pattern replacement, patch generation).
5.	Apply the patch and create a new revision as above.
6.	Respond with JSON:
{
  "success": true,
  "newContent": "..."
}
5. Common Patterns Across All Handlers
•	Reuse existing LoadWorkspace, LoadLatestRevision, and ListFiles logic.
•	Reuse ApplyPatch, CreateRevision, and DB change tracking.
•	Use common response helpers, e.g. WriteJSON(w, statusCode, payload).
•	All handlers must:
•	Validate user access
•	Return structured errors
•	Trigger NOTIFY events
