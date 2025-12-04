# Manual Test Cases: `/test-ai-chat` vs `/` Parity

**Purpose**: Verify the new AI SDK chat system (`/test-ai-chat`) has feature parity with the existing production chat.  
**Created**: December 4, 2025  
**PR**: PR1.5 Validation

---

## Prerequisites

- [ ] Go worker running (`make run-worker`) - HTTP server on port 8080
- [ ] Next.js dev server running (`npm run dev`) - port 3000
- [ ] Database accessible
- [ ] Have a valid workspace created (via homepage flow first)

---

## 1. Basic Chat Functionality

| Test | Steps | Expected Result | Pass? |
|------|-------|-----------------|-------|
| **Send message** | Type "Hello" and send | AI responds with streaming text | ☐ |
| **Streaming display** | Watch response appear | Text streams in progressively (not all at once) | ☐ |
| **Multiple messages** | Send 3+ messages in a row | All messages display in order | ☐ |
| **Long response** | Ask "Explain Kubernetes in detail" | Full response renders without truncation | ☐ |
| **Auto-scroll** | Send message when scrolled up | Chat scrolls to newest message | ☐ |

---

## 2. Provider/Model Selection

| Test | Steps | Expected Result | Pass? |
|------|-------|-----------------|-------|
| **Default model** | Load page | Claude Sonnet 4 selected | ☐ |
| **Switch provider** | Select different model from dropdown | Model switches without error | ☐ |
| **Model persists** | Send message after switching | Response uses selected model | ☐ |

---

## 3. Tool: getChartContext

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **Load context** | "What files are in my chart?" | AI lists files from workspace | ☐ |
| **Chart structure** | "Describe my chart structure" | AI explains deployment, service, etc. | ☐ |
| **Empty workspace** | Ask with new empty workspace | AI indicates no files or minimal structure | ☐ |

---

## 4. Tool: textEditor

### View Command

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **View existing file** | "Show me values.yaml" | AI displays file contents | ☐ |
| **View deployment** | "Show me the deployment.yaml" | AI displays deployment config | ☐ |
| **Non-existent file** | "Show me nonexistent.yaml" | AI reports file doesn't exist | ☐ |

### Create Command

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **Create new file** | "Create a configmap.yaml with key: value" | AI creates the file | ☐ |
| **Create with content** | "Create a secret.yaml for database credentials" | AI creates with appropriate content | ☐ |
| **File already exists** | "Create values.yaml" (when exists) | AI reports file exists, suggests str_replace | ☐ |

### str_replace Command

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **Simple edit** | "Change replicas from 1 to 3" | AI modifies the file | ☐ |
| **Add content** | "Add a new environment variable DEBUG=true" | AI uses str_replace to add | ☐ |
| **String not found** | "Replace NONEXISTENT with something" | AI reports string not found | ☐ |

---

## 5. Tool: latestSubchartVersion

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **PostgreSQL** | "What's the latest PostgreSQL chart version?" | Returns version like `18.1.13` | ☐ |
| **Redis** | "Latest Redis helm chart version?" | Returns valid semver version | ☐ |
| **Nginx** | "What version is the nginx-ingress chart?" | Returns version | ☐ |
| **Unknown chart** | "Latest version of fake-nonexistent-chart?" | Returns "?" or graceful error | ☐ |
| **Multiple lookups** | "Get versions for postgresql and redis" | Both versions returned | ☐ |

---

## 6. Tool: latestKubernetesVersion

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **Default (patch)** | "What's the latest Kubernetes version?" | Returns `1.32.1` | ☐ |
| **Major version** | "What's the major Kubernetes version?" | Returns `1` | ☐ |
| **Minor version** | "What's the current K8s minor version?" | Returns `1.32` | ☐ |
| **Compatibility** | "Is my chart compatible with latest K8s?" | AI uses version info in response | ☐ |

---

## 7. Multi-Tool Workflows

| Test | Prompt | Expected Result | Pass? |
|------|--------|-----------------|-------|
| **Context + Edit** | "Look at my chart and add resource limits" | AI reads context, then edits | ☐ |
| **Version + Create** | "Add postgresql as a dependency with latest version" | AI looks up version, creates/modifies Chart.yaml | ☐ |
| **Full workflow** | "Review my chart and suggest improvements" | AI uses getChartContext, explains issues | ☐ |

---

## 8. Error Handling

| Test | Steps | Expected Result | Pass? |
|------|-------|-----------------|-------|
| **No workspace ID** | Load page without workspace context | Graceful error or no tools available | ☐ |
| **Go server down** | Stop Go worker, send tool request | Error message (not crash) | ☐ |
| **Network timeout** | Slow network simulation | Timeout error displayed | ☐ |

---

## 9. UI/UX Parity with `/`

| Test | Check | Expected | Pass? |
|------|-------|----------|-------|
| **Layout** | Compare visual layout | Similar structure | ☐ |
| **Message bubbles** | User vs AI styling | Distinct visual difference | ☐ |
| **Loading state** | While AI responds | Loading indicator visible | ☐ |
| **Error display** | Force an error | Error message styled properly | ☐ |
| **Input disabled** | While streaming | Input disabled during response | ☐ |

---

## 10. Feature Comparison Matrix

| Feature | `/` (Homepage) | `/test-ai-chat` | Parity? |
|---------|----------------|-----------------|---------|
| Streaming responses | ✅ | ✅ | ☐ |
| Model selection | ❓ | ✅ | ☐ |
| View chart files | ✅ (via Jotai) | ✅ (via tool) | ☐ |
| Edit chart files | ✅ (via queue) | ✅ (via tool) | ☐ |
| Version lookups | ✅ (via Go) | ✅ (via tool) | ☐ |
| Real-time updates | ✅ (Centrifugo) | ❌ | Expected diff |

---

## Quick Smoke Test (5 minutes)

Run these 6 tests for a quick validation:

1. [ ] **Load page** → Check no console errors
2. [ ] **Send "Hello"** → Verify streaming response
3. [ ] **"What files are in my chart?"** → Verify getChartContext works
4. [ ] **"What's the latest PostgreSQL version?"** → Verify version lookup
5. [ ] **"Show me values.yaml"** → Verify textEditor view
6. [ ] **"Change replicas to 5"** → Verify textEditor str_replace

---

## Console Monitoring Checklist

Keep browser DevTools Console open during testing:

- [ ] No JavaScript errors
- [ ] No network 4xx/5xx errors (except expected 404s)
- [ ] Tool calls logged (if debug enabled)
- [ ] No memory leaks on long sessions

---

## Test Results Summary

| Section | Tests | Passed | Failed | Notes |
|---------|-------|--------|--------|-------|
| Basic Chat | 5 | | | |
| Provider/Model | 3 | | | |
| getChartContext | 3 | | | |
| textEditor View | 3 | | | |
| textEditor Create | 3 | | | |
| textEditor str_replace | 3 | | | |
| latestSubchartVersion | 5 | | | |
| latestKubernetesVersion | 4 | | | |
| Multi-Tool Workflows | 3 | | | |
| Error Handling | 3 | | | |
| UI/UX Parity | 5 | | | |
| **TOTAL** | **40** | | | |

---

## Sign-Off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Tester | | | |
| Reviewer | | | |

---

*Document created for PR1.5 validation. Update as needed for PR2.*

