PR1.5 Refinements Document
This document captures light refinements recommended to tighten the PR1.5 deliverable and improve clarity and reviewability.
Refinement 1 — Explicitly Declare UI Deprecation of Legacy Chat
Add to PR1.5 Product PRD:
“After PR1.5, no user‑facing chat UI paths route through the legacy Go LLM stack (pkg/llm/*).
The only chat interface available in the product is the AI SDK–powered /api/chat route.”
This makes “one chat path” explicit.
Refinement 2 — Add a Security & Auth Note for Go Tool Endpoints
Add:
“All PR1.5 Go HTTP tool endpoints (/api/tools/...) will enforce existing workspace ownership and authentication middleware. Only the owner or authorized collaborators may modify or read workspace resources.”
This keeps reviewers confident that new entrypoints don’t bypass security.
Refinement 3 — Specify Error Response Structure
Add:
“Every Go tool handler returns structured errors in the shape:
{ "success": false, "message": "error text" }
The AI SDK route will surface these errors to users in a readable assistant message.”
Ensures consistent error UX.
Refinement 4 — Clarify Reuse of Existing Workspace Logic
To reassure reviewers:
“Chart creation, revision creation, file writes, and event notifications all reuse existing production‑stable Go functions (e.g., CreateWorkspace, CreateInitialRevision, ApplyPatch).
We do not fork or duplicate workspace logic.”
Good for maintainability.
Refinement 5 — Add One‑Sentence Rationale for Dropping Go→LLM Calls
Add:
“Go no longer performs any LLM reasoning after PR1.5; all intelligence (planning, tool invocation) is handled in the AI SDK layer.
Go becomes a clean service layer responsible only for deterministic CRUD and file operations.”
This explains why gateway removal is a simplification, not a deficiency.
Refinement 6 — Add Success Metric About Tool Invocation
To the Success Criteria section, add:
“AI SDK chat must invoke createChart for a new‑chart conversation, and Go must create a valid workspace.
This is validated via integration test.”
Echoes the spec requirement: “Demonstrate creating a new chart via chat.”
Refinement 7 — Add Optional Future Cleanup Section
Not required in PR1.5, but nice to state:
“In a later PR (post‑PR2), the deprecated Go LLM stack (pkg/llm/*) may be removed entirely once all call sites are eliminated.”
Signals a long‑term path to a cleaner codebase.
