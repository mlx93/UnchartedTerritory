# Manual UI Test Cases for Chartsmith

This document contains comprehensive manual test cases for validating Chartsmith's UI functionality before and after code changes.

---

## Prerequisites

Before running these tests, ensure:
- [ ] Docker containers running (`docker compose up -d` in `hack/chartsmith-dev/`)
- [ ] Database schema deployed (`make schema`)
- [ ] Bootstrap data loaded (`make bootstrap`)
- [ ] Artifact Hub cache populated (`./bin/chartsmith-worker artifacthub`)
- [ ] Helm CLI installed (`brew install helm`)
- [ ] Frontend running (`npm run dev` in `chartsmith-app/`)
- [ ] Backend worker running (`make run-worker`)

---

## 🔐 TC-1: Authentication Flow

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1.1 | Navigate to `http://localhost:3000` | Should see login page or home page (if already logged in) |
| 1.2 | Go to `http://localhost:3000/login?test-auth=true` | Should auto-login and redirect to home page |
| 1.3 | Verify avatar appears in top-right corner | User avatar/icon visible |
| 1.4 | Click avatar → Logout (if option exists) | Should log out and return to login |

**Notes:**
- First user to log in gets admin privileges
- Test auth bypasses Google OAuth for local development

---

## 🏠 TC-2: Home Page Elements

| Step | Action | Expected Result |
|------|--------|-----------------|
| 2.1 | View home page | "Welcome to ChartSmith" header visible |
| 2.2 | Verify text input | Placeholder: "Tell me about the application you want to create a Helm chart for" |
| 2.3 | Verify action buttons | 3 buttons: "Upload a Helm chart", "Upload Kubernetes manifests", "Start from a chart in Artifact Hub" |
| 2.4 | Verify footer | "Terms" and "Privacy" links visible |

**Notes:**
- Background image should load (dark themed with cubes)
- Input should auto-focus on page load

---

## ✏️ TC-3: Prompt-Based Chart Creation

| Step | Action | Expected Result |
|------|--------|-----------------|
| 3.1 | Type in prompt field: "Create a simple nginx deployment" | Text appears in field |
| 3.2 | Verify character limit warning | Warning appears when approaching 500 chars |
| 3.3 | Press Enter (or click arrow button) | Loading spinner appears |
| 3.4 | Wait for redirect | URL changes to `/workspace/{id}` |
| 3.5 | Verify workspace loads | Chat interface with AI response visible |

**Notes:**
- Maximum 512 characters
- Warning threshold at 500 characters
- Arrow button only appears when text is entered

---

## 📁 TC-4: Helm Chart Upload

| Step | Action | Expected Result |
|------|--------|-----------------|
| 4.1 | Click "Upload a Helm chart" button | File picker dialog opens |
| 4.2 | Select a `.tgz` or `.tar.gz` file | Upload begins, loading indicator shown |
| 4.3 | Wait for processing | Redirected to `/workspace/{id}` |
| 4.4 | Verify chart files appear | File tree shows chart structure |

**Notes:**
- Maximum file size: 100MB
- Accepted formats: `.tgz`, `.tar.gz`, `.tar`
- Chart must have valid `Chart.yaml`

---

## 📁 TC-5: Kubernetes Manifest Upload

| Step | Action | Expected Result |
|------|--------|-----------------|
| 5.1 | Click "Upload Kubernetes manifests" | File picker dialog opens |
| 5.2 | Select a manifest archive | Upload begins |
| 5.3 | Wait for processing | Redirected to workspace |
| 5.4 | Verify conversion message | AI mentions converting K8s to Helm |

**Notes:**
- Manifests are converted to Helm chart format
- Conversion progress shown in UI

---

## 🔍 TC-6: Artifact Hub Search

| Step | Action | Expected Result |
|------|--------|-----------------|
| 6.1 | Click "Start from a chart in Artifact Hub" | Modal/search dialog opens |
| 6.2 | Type "nginx" in search | Search results appear |
| 6.3 | Select a chart | Chart details or confirmation shown |
| 6.4 | Confirm selection | Workspace created with chart |

**Prerequisites:**
- Artifact Hub cache must be populated (`./bin/chartsmith-worker artifacthub`)

**Notes:**
- Search queries local database cache, not live API
- Minimum 3 characters for search
- Can also paste direct Artifact Hub URLs
- Format `org/chart-name` also supported

---

## 💬 TC-7: Workspace Chat Functionality

| Step | Action | Expected Result |
|------|--------|-----------------|
| 7.1 | In workspace, view chat panel | Messages visible, input at bottom |
| 7.2 | Click role selector (sparkle icon) | Dropdown: "Auto-detect", "Chart Developer", "End User" |
| 7.3 | Type: "What files are in this chart?" | Text appears in input |
| 7.4 | Press Enter | Message sent, loading indicator, AI response streams |
| 7.5 | Verify AI response | Response shows chart file list |

**Notes:**
- Role selector only visible after first revision created
- Messages persist across page refreshes
- Streaming responses update in real-time via WebSocket

---

## 📝 TC-8: File Editor Functionality

| Step | Action | Expected Result |
|------|--------|-----------------|
| 8.1 | Click a file in file tree (e.g., `values.yaml`) | File opens in editor panel |
| 8.2 | Make an edit to the file | Changes visible in editor |
| 8.3 | Ask AI: "Update the replica count to 3" | AI modifies file, diff shown |
| 8.4 | Verify pending changes indicator | Visual indicator of unsaved changes |

**Notes:**
- Monaco editor used for syntax highlighting
- Diff view shows before/after for AI changes
- Accept/Reject buttons for pending changes

---

## 🔄 TC-9: AI Tool Usage (Streaming)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 9.1 | Ask: "Add a ConfigMap for environment variables" | AI response begins streaming |
| 9.2 | Observe tool indicators | Should see "editing file" or similar tool indicators |
| 9.3 | Wait for completion | New file appears in tree OR existing file modified |
| 9.4 | View the change | Diff or new content visible |

**Notes:**
- Tools include: text_editor, get_latest_subchart_version, get_latest_kubernetes_version
- Tool execution shown in real-time

---

## 📋 TC-10: Multiple Messages

| Step | Action | Expected Result |
|------|--------|-----------------|
| 10.1 | Send 3-4 messages in sequence | Each message/response appears in order |
| 10.2 | Scroll up in chat | Previous messages visible |
| 10.3 | Send new message | Auto-scrolls to bottom |
| 10.4 | Verify message persistence | Refresh page → messages still there |

**Notes:**
- Messages stored in database
- Auto-scroll behavior on new messages
- WebSocket handles real-time updates

---

## 🎨 TC-11: Rendered Output View

| Step | Action | Expected Result |
|------|--------|-----------------|
| 11.1 | Create or open a workspace with a chart | Chart files visible in Explorer |
| 11.2 | Click "Rendered" tab | View switches to rendered output |
| 11.3 | Click a file in the tree | Shows rendered K8s manifest OR "not included" message |
| 11.4 | Click "Source" tab | View switches back to template source |

**Prerequisites:**
- Helm CLI must be installed
- Render must have completed successfully

**Notes:**
- Rendered view shows output of `helm template`
- Files may be excluded based on values.yaml conditionals
- "Why was this file not included?" button available

---

## ✅ TC-12: Accept/Reject Changes

| Step | Action | Expected Result |
|------|--------|-----------------|
| 12.1 | Ask AI to modify a file | Diff view appears with pending changes |
| 12.2 | Click "Accept" button | Changes applied, diff view closes |
| 12.3 | Ask AI for another change | New diff appears |
| 12.4 | Click "Reject" button | Changes discarded, original content restored |

**Notes:**
- Accept/Reject dropdown allows "This file" or "All files"
- Navigation arrows to move between files with changes

---

## 🚀 Quick Smoke Test (5 minutes)

Run these for a fast baseline validation:

| # | Test | Pass? |
|---|------|-------|
| 1 | Login: `http://localhost:3000/login?test-auth=true` → redirects to home | ☐ |
| 2 | Create workspace: Type "simple nginx pod" → Enter → workspace loads | ☐ |
| 3 | AI responds: Initial AI message appears with chart content | ☐ |
| 4 | Chat works: Ask "list all files" → AI responds | ☐ |
| 5 | File tree works: Click a file → opens in editor | ☐ |
| 6 | Render works: Type "render" → Terminal shows helm output | ☐ |

---

## Known Issues / Limitations

1. **WebSocket disconnects**: Frontend may show "generating..." even after backend completes. Refresh to resolve.
2. **AI context limitations**: AI may not have access to all file contents in current implementation.
3. **Artifact Hub search**: Requires cache population; won't work with empty database.
4. **Render requires Helm**: `helm` CLI must be installed and in PATH.

---

*Last Updated: December 2024*

