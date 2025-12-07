# PR4: Chartsmith Validation Agent & Live Provider Switching - Implementation Summary

**Date**: 2025-12-07
**Status**: Implementation Complete - Ready for Manual Testing

---

## Overview

This PR implements two major features for Chartsmith:

1. **Chart Validation Agent** - AI-powered validation using helm lint, helm template, and kube-score
2. **Live Provider Switching** - Dynamic model switching mid-conversation without losing context

---

## Files Created

### Go Backend (Chart Validation)

| File | Purpose |
|------|---------|
| `pkg/validation/types.go` | Type definitions for ValidationRequest, ValidationResult, ValidationIssue, LintResult, TemplateResult, ScoreResult |
| `pkg/validation/helm.go` | helm lint and helm template execution with output parsing |
| `pkg/validation/kubescore.go` | kube-score execution with JSON parsing and fix suggestions |
| `pkg/validation/pipeline.go` | Three-stage validation orchestration (lint → template → kube-score) |
| `pkg/api/handlers/validate.go` | HTTP handler for POST /api/validate endpoint |

### Frontend (Validation Tool & UI)

| File | Purpose |
|------|---------|
| `chartsmith-app/lib/ai/tools/validateChart.ts` | AI SDK tool factory for chart validation |
| `chartsmith-app/atoms/validationAtoms.ts` | Jotai atoms for validation state management |
| `chartsmith-app/components/chat/ValidationResults.tsx` | UI component displaying validation results with expandable issues |

### Frontend (Live Provider Switching)

| File | Purpose |
|------|---------|
| `chartsmith-app/components/chat/LiveProviderSwitcher.tsx` | Dropdown component for model selection |

---

## Files Modified

### Go Backend

| File | Changes |
|------|---------|
| `pkg/api/server.go` | Added route registration: `POST /api/validate` → `handlers.ValidateChart` |

### Frontend

| File | Changes |
|------|---------|
| `chartsmith-app/lib/ai/tools/index.ts` | Added validateChart import/export, registered in createTools() |
| `chartsmith-app/lib/ai/tools/bufferedTools.ts` | Added validateChart to createBufferedTools() (read-only, no buffering) |
| `chartsmith-app/components/types.ts` | Added `responseValidationId?: string` to Message and RawChatMessage interfaces |
| `chartsmith-app/hooks/useAISDKChatAdapter.ts` | Added `selectedProvider`, `selectedModel` state and `switchProvider()` callback; updated `getChatBody()` to use dynamic provider/model |
| `chartsmith-app/hooks/useLegacyChat.ts` | Added provider state for interface compatibility (defaults, no-op switch) |
| `chartsmith-app/components/ChatContainer.tsx` | Integrated LiveProviderSwitcher component in input area |
| `chartsmith-app/components/ChatMessage.tsx` | Added ValidationResults rendering in SortedContent component |

---

## Architecture

### Validation Pipeline Flow

```
User: "validate my chart"
         ↓
   AI SDK detects intent
         ↓
   Invokes validateChart tool
         ↓
   callGoEndpoint('/api/validate')
         ↓
   Go handler: ValidateChart()
         ↓
   validation.RunValidation()
         ↓
   ┌─────────────────────────────────────┐
   │ Stage 1: helm lint                  │
   │   - Parse output for [ERROR/WARNING]│
   │   - Generate suggestions            │
   └─────────────────────────────────────┘
         ↓ (if passed or not strict)
   ┌─────────────────────────────────────┐
   │ Stage 2: helm template              │
   │   - Render YAML                     │
   │   - Count resources                 │
   │   - Capture for Stage 3             │
   └─────────────────────────────────────┘
         ↓ (if succeeded)
   ┌─────────────────────────────────────┐
   │ Stage 3: kube-score (non-fatal)     │
   │   - Parse JSON output               │
   │   - Map grades to severity          │
   │   - Add fix suggestions             │
   └─────────────────────────────────────┘
         ↓
   Return aggregated ValidationResult
         ↓
   AI interprets and explains results
```

### Live Provider Switching Flow

```
User clicks LiveProviderSwitcher dropdown
         ↓
   Selects different model (e.g., GPT-4o)
         ↓
   chatState.switchProvider(provider, model)
         ↓
   Updates selectedProvider/selectedModel state
         ↓
   Next message uses new model in getChatBody()
         ↓
   Conversation history preserved (client-side)
```

---

## Key Implementation Details

### Validation Tool

- **Tool Name**: `validateChart`
- **Description**: Validates Helm charts for syntax, rendering, and best practices
- **Input Schema**:
  - `values?: Record<string, unknown>` - Value overrides
  - `strictMode?: boolean` - Treat warnings as failures
  - `kubeVersion?: string` - Target K8s version
- **Behavior**: Read-only (not buffered), executes immediately

### Kube-Score Grade Mapping

| Grade | Severity |
|-------|----------|
| 1 (CRITICAL) | `critical` |
| 2-5 (WARNING) | `warning` |
| 7-10 (OK) | Not reported (passed) |

### Fix Suggestions (partial list)

| Check ID | Suggestion |
|----------|------------|
| `container-resources` | Add resources.limits.memory and resources.limits.cpu |
| `container-security-context` | Set securityContext.runAsNonRoot: true |
| `container-image-tag` | Use specific image tag instead of :latest |
| `pod-probes` | Add readinessProbe and livenessProbe |

---

## Verification Results

### Automated Checks

| Check | Status |
|-------|--------|
| Go build (`go build ./...`) | ✅ Pass |
| TypeScript (`npx tsc --noEmit`) | ✅ Pass (production files) |

### Pre-existing Issues (Not from PR4)

- Test file type errors in `app/api/chat/__tests__/route.test.ts` (mock typing)
- Test file error in `tests/upload-chart.spec.ts`

---

## Manual Testing Checklist

Per PRD requirements:

- [ ] "Validate my chart" triggers validation tool
- [ ] helm lint, helm template, kube-score all execute
- [ ] Results displayed with severity indicators (critical/warning/info)
- [ ] AI explains issues in natural language
- [ ] Fix suggestions provided for each issue
- [ ] Live provider switching works without losing history
- [ ] All PR3.0/3.1 functionality still works

---

## API Contract

### POST /api/validate

**Request:**
```json
{
  "workspaceId": "ws_123",
  "revisionNumber": 5,
  "values": { "replicaCount": 3 },
  "strictMode": false,
  "kubeVersion": "1.28"
}
```

**Success Response (200):**
```json
{
  "validation": {
    "overall_status": "warning",
    "timestamp": "2024-01-15T10:30:00Z",
    "duration_ms": 2500,
    "results": {
      "helm_lint": { "status": "pass", "issues": [] },
      "helm_template": {
        "status": "pass",
        "rendered_resources": 5,
        "output_size_bytes": 4096
      },
      "kube_score": {
        "status": "warning",
        "score": 6,
        "total_checks": 15,
        "passed_checks": 12,
        "issues": [...]
      }
    }
  }
}
```

**Error Responses:**
- `400` - Missing workspaceId/revisionNumber, workspace not found
- `500` - Internal error (pipeline failure)
- `504` - Timeout (exceeded 30 seconds)

---

## Dependencies

### External CLIs (Required)

- `helm` v3.x - For lint and template commands
- `kube-score` v1.16+ - For best practices checking (graceful fallback if missing)

### Internal Dependencies (Already Exist)

- `pkg/workspace.ListCharts()` - For workspace/chart access
- `callGoEndpoint()` - For tool-to-backend communication
- `AVAILABLE_PROVIDERS` / `AVAILABLE_MODELS` - For provider configuration
