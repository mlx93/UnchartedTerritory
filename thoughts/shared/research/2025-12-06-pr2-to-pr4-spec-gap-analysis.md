---
date: 2025-12-06T12:00:00-08:00
researcher: mlx93
git_commit: 88277edc2aaf918b606c9bcac60424b502dba51b
branch: main
repository: UnchartedTerritory
topic: "PR2 Spec Gap Analysis: What's Valid, What Needs to Change for PR4"
tags: [research, codebase, pr2, pr4, validation-agent, provider-switching, migration]
status: complete
last_updated: 2025-12-06
last_updated_by: mlx93
---

# Research: PR2 Spec Gap Analysis for PR4 Implementation

**Date**: 2025-12-06T12:00:00-08:00
**Researcher**: mlx93
**Git Commit**: 88277edc2aaf918b606c9bcac60424b502dba51b
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

Assuming PR3.2 feature parity fixes are implemented, what parts of the PR2 specs (Validation Agent & Live Provider Switching) are still valid and what needs to change to become PR4?

## Summary

The PR2 specs were written assuming a different codebase state. Since then, significant infrastructure has been built through PR3.0/3.1/3.2 that makes much of PR2 easier to implement. **Approximately 60% of the PR2 technical specification remains valid**, with the main changes being:

1. **Architecture is already decided and implemented** - Option A (HTTP endpoints) is in use
2. **Tool patterns exist** - Can follow established patterns rather than create new ones
3. **Provider switching needs rework** - Original spec assumed locking at conversation start, but current implementation needs live switching capability added
4. **Validation pipeline is entirely new** - This remains the bulk of the work

## Detailed Findings

### What Already Exists (No Implementation Needed)

#### 1. Go HTTP Server Infrastructure
- **Server**: `pkg/api/server.go` - HTTP server on port 8080
- **Pattern**: `mux.HandleFunc("POST /api/path", handlers.Handler)`
- **Handlers**: `pkg/api/handlers/*.go` - 8 existing handler files
- **Utilities**: `pkg/api/handlers/response.go` - Error response helpers

**PR2 spec assumed this didn't exist.** Now we just add a new handler.

#### 2. Tool Factory Pattern
- **Location**: `chartsmith-app/lib/ai/tools/*.ts`
- **Pattern**: Factory functions with context closure
- **Utilities**: `callGoEndpoint()` in `tools/utils.ts`
- **Integration**: `createTools()` and `createBufferedTools()` in `tools/index.ts`

**PR2 spec defined this pattern.** It's now implemented and can be followed.

#### 3. Tool Registration in Chat Route
- **Location**: `chartsmith-app/app/api/chat/route.ts`
- **Lines 132-138**: Tool creation based on context
- **Line 208**: Tools passed to `streamText()`

**PR2 spec described adding tools here.** Infrastructure exists.

#### 4. Provider Configuration System
- **Models**: `lib/ai/models.ts` - AVAILABLE_PROVIDERS, AVAILABLE_MODELS
- **Config**: `lib/ai/config.ts` - DEFAULT_PROVIDER, DEFAULT_MODEL
- **Factory**: `lib/ai/provider.ts` - `getModel(provider, model)`

**PR2 spec assumed this existed from PR1.** It does.

#### 5. ProviderSelector Component (Partial)
- **Location**: `components/chat/ProviderSelector.tsx`
- **Status**: Component exists but is NOT integrated into ChatContainer

**PR2 spec wanted to replace with LiveProviderSwitcher.** Needs integration work.

---

### What's Still Valid (Implementation Matches Spec)

#### 1. Validation Pipeline Logic (Go)

The spec's validation pipeline design remains valid:

| Stage | Tool | Description | Status |
|-------|------|-------------|--------|
| 1 | helm lint | Syntax/structure checks | Still valid |
| 2 | helm template | Template rendering | Still valid |
| 3 | kube-score | K8s best practices | Still valid |

**File locations per spec**:
```
pkg/validation/           # NEW - Validation logic (still valid)
├── pipeline.go          # RunValidation orchestration
├── helm.go              # runHelmLint, runHelmTemplate
├── kubescore.go         # runKubeScore
├── parser.go            # Output parsing utilities
└── types.go             # All type definitions
```

#### 2. Validation Types

The spec's type definitions remain valid:

- `ValidationRequest` - Input to pipeline
- `ValidationResult` - Pipeline output
- `ValidationIssue` - Individual finding
- `LintResult`, `TemplateResult`, `ScoreResult`

#### 3. validateChart Tool Schema

The Zod schema from the spec is still valid:

```typescript
inputSchema: z.object({
  chartPath: z.string().describe("Path to chart directory"),
  values: z.record(z.any()).optional().describe("Values overrides"),
  strictMode: z.boolean().optional().describe("Fail on warnings"),
  kubeVersion: z.string().optional().describe("Target K8s version")
})
```

#### 4. API Contract

The POST `/api/validate` contract remains valid:

**Request**:
```json
{
  "chartPath": "/path/to/chart",
  "values": { "key": "value" },
  "strictMode": false,
  "kubeVersion": "1.28"
}
```

**Response structure** as specified is still valid.

#### 5. ValidationResults Component Design

The component design in PR2_SUPPLEMENTAL_DETAILS.md remains valid:
- Status badge with color coding
- Issues grouped by severity
- Expandable issue details
- kube-score summary

---

### What Needs to Change

#### 1. Architecture Decision Section - REMOVE

**Current Spec**:
```markdown
## Architecture Decision: RESOLVED
**Decision**: Option A - Add HTTP Endpoint to Go Backend
```

**Change**: This section should be removed from PR4 spec. Option A is already the established pattern. No decision needed.

#### 2. Provider Switching - SIGNIFICANT REWORK

**Original PR2 Spec Assumption** (FR-4.2):
> "Conversation history preserved completely"
> "Only future responses use new model"

**Current State**:
- `useAISDKChatAdapter.ts:140-141` **hardcodes** provider/model:
  ```typescript
  const getChatBody = useCallback(
    (messageId?: string) => ({
      provider: DEFAULT_PROVIDER,  // Hardcoded!
      model: DEFAULT_MODEL,        // Hardcoded!
      // ...
    }),
    [workspaceId, revisionNumber]
  );
  ```

**Required Changes**:

1. **Add state to useAISDKChatAdapter**:
   ```typescript
   const [selectedProvider, setSelectedProvider] = useState(DEFAULT_PROVIDER);
   const [selectedModel, setSelectedModel] = useState(DEFAULT_MODEL);
   ```

2. **Update getChatBody to use state**:
   ```typescript
   provider: selectedProvider,
   model: selectedModel,
   ```

3. **Expose switch function**:
   ```typescript
   const switchProvider = useCallback((provider, model) => {
     setSelectedProvider(provider);
     setSelectedModel(model);
   }, []);
   ```

4. **Integrate LiveProviderSwitcher into ChatContainer**:
   - Currently uses persona selector (role)
   - Need to add provider selector alongside or replace

#### 3. Tool Registration Location - SIMPLIFIED

**Original Spec**:
```typescript
// In /api/chat/route.ts (CREATED BY PR1)
import { validateChart } from '@/lib/ai/tools/validateChart'
```

**Current State**:
- Tools are created via factories in `tools/index.ts`
- Need to add `createValidateChartTool()` factory
- Need to add to both `createTools()` and `createBufferedTools()`

**Updated Pattern**:
```typescript
// In tools/index.ts
import { createValidateChartTool } from './validateChart';

export function createTools(...) {
  return {
    ...existingTools,
    validateChart: createValidateChartTool(authHeader, workspaceId, revisionNumber),
  };
}
```

#### 4. Handler Registration - SIMPLIFIED

**Original Spec**:
```go
// Add new route registration after line 38
mux.HandleFunc("POST /api/validate", handlers.ValidateChart)
```

**Current State**:
- `pkg/api/server.go` has pattern established
- Line ~41-44 shows recent additions (render/trigger)

**Updated Location**:
```go
// In pkg/api/server.go, around line 45
mux.HandleFunc("POST /api/validate", handlers.ValidateChart)
```

#### 5. Environment Variable - ALREADY EXISTS

**Original Spec**:
> Environment variable: `GO_BACKEND_URL=http://localhost:8080`

**Current State**:
- Already implemented in `tools/utils.ts:9`:
  ```typescript
  const GO_BACKEND_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';
  ```

**Change**: Remove from PR4 spec as environment setup. Document that it uses existing pattern.

#### 6. PR Numbering Reference - UPDATE

**Original Spec**:
- References "PR1" and "PR2"
- "Prerequisite: PR1.5 merged"

**Current State**:
- We're at PR3.2
- Should be "Prerequisite: PR3.2 merged"
- This becomes "PR4" (or could be PR3.3 depending on naming)

---

### Component-by-Component Gap Analysis

#### validateChart Tool (`lib/ai/tools/validateChart.ts`)

| Aspect | PR2 Spec | Current State | Status |
|--------|----------|---------------|--------|
| File location | `src/lib/ai/tools/validateChart.ts` | Should be `chartsmith-app/lib/ai/tools/validateChart.ts` | **Path update** |
| Tool pattern | AI SDK `tool()` | Factory function pattern | **Follow existing** |
| Execute function | Direct fetch | Use `callGoEndpoint()` | **Follow existing** |
| Registration | Add to streamText | Add to `createTools()` | **Follow existing** |

#### ValidationResults Component (`components/chat/ValidationResults.tsx`)

| Aspect | PR2 Spec | Current State | Status |
|--------|----------|---------------|--------|
| Component design | Card with severity badges | N/A - doesn't exist | **Implement as spec** |
| Location | `src/components/chat/` | `chartsmith-app/components/chat/` | **Path update** |
| Rendering trigger | Tool result part | Via message mapper | **Investigate pattern** |

#### LiveProviderSwitcher Component

| Aspect | PR2 Spec | Current State | Status |
|--------|----------|---------------|--------|
| Replace ProviderSelector | Yes | ProviderSelector exists unused | **Reuse or create new** |
| Location | Chat header | ChatContainer has role selector only | **Add to header** |
| State management | In Chat component | In useAISDKChatAdapter | **Rework needed** |

#### Go Validation Handler (`pkg/api/handlers/validate.go`)

| Aspect | PR2 Spec | Current State | Status |
|--------|----------|---------------|--------|
| Handler structure | Standalone file | Follow existing pattern in handlers/ | **Match existing** |
| Response helpers | Define new | Use existing `writeJSON`, `writeBadRequest` | **Reuse** |
| Security (path traversal) | Implement | Pattern exists in editor.go | **Follow existing** |

---

### Validation Pipeline Changes

The validation pipeline spec is mostly valid, but should reference existing patterns:

#### 1. Error Handling Pattern
**PR2 Spec**: Defined new response helpers
**Current**: Use `handlers/response.go` utilities:
- `writeJSON(w, status, data)`
- `writeBadRequest(w, message)`
- `writeInternalError(w, message)`

#### 2. Workspace Access Pattern
**PR2 Spec**: Direct database queries
**Current**: Use `workspace.ListCharts()` like `editor.go:220`

#### 3. Realtime Updates Pattern
**PR2 Spec**: Not detailed
**Current**: Could add `ValidationCompletedEvent` following `ArtifactUpdatedEvent` pattern

---

### Migration Sequence Changes

The original 3-day timeline (Days 4-6) needs adjustment:

**Original Day 4**: Go Backend
- Still valid: Create `pkg/validation/` package
- Simplified: Use existing handler patterns

**Original Day 5**: API & Frontend
- Still valid: Create validateChart tool
- Changed: Provider switching needs more work than spec assumed
- Changed: Use existing tool patterns

**Original Day 6**: Integration & Polish
- Still valid: End-to-end testing
- Added: Provider switching integration testing

---

### Files to Create (Still Required)

#### Go Backend
```
pkg/validation/
├── pipeline.go         # NEW - RunValidation function
├── helm.go             # NEW - runHelmLint, runHelmTemplate
├── kubescore.go        # NEW - runKubeScore
├── parser.go           # NEW - Output parsing
└── types.go            # NEW - Type definitions

pkg/api/handlers/
└── validate.go         # NEW - HTTP handler
```

#### Frontend
```
chartsmith-app/lib/ai/tools/
└── validateChart.ts            # NEW - Tool factory

chartsmith-app/components/chat/
├── ValidationResults.tsx       # NEW - Results display
└── LiveProviderSwitcher.tsx    # NEW or reuse ProviderSelector
```

---

### Files to Modify

#### Go Backend
- `pkg/api/server.go` - Add route registration (line ~45)

#### Frontend
- `lib/ai/tools/index.ts` - Add validateChart to tool factories
- `lib/ai/tools/bufferedTools.ts` - Add validateChart (non-buffered, read-only)
- `hooks/useAISDKChatAdapter.ts` - Add provider/model state management
- `components/ChatContainer.tsx` - Add provider switcher UI

---

## Recommendations for PR4 Spec

### 1. Remove Architecture Decision Sections
The decision is made. Just reference the existing pattern.

### 2. Add "Current State" Section
Document what already exists before describing changes.

### 3. Update File Paths
Use `chartsmith-app/` prefix, not `src/`.

### 4. Simplify Tool Integration
Reference existing patterns by file:line instead of redefining.

### 5. Expand Provider Switching
This needs more detail than original spec since implementation requires reworking adapter state.

### 6. Add Validation Tool Result Rendering
The spec mentions ValidationResults component but doesn't detail how it integrates with message rendering. Need to add to `messageMapper.ts` and `ChatMessage.tsx`.

---

## Code References

### Existing Patterns to Follow

- Tool factory: `chartsmith-app/lib/ai/tools/textEditor.ts:34-89`
- Go handler: `chartsmith/pkg/api/handlers/editor.go:76-118`
- Handler registration: `chartsmith/pkg/api/server.go:20-44`
- Tool registration: `chartsmith-app/lib/ai/tools/index.ts:48-70`
- Provider state: `chartsmith-app/hooks/useAISDKChatAdapter.ts:138-148`

### Key Integration Points

- `app/api/chat/route.ts:208` - streamText with tools
- `components/ChatMessage.tsx:155-224` - Tool result rendering
- `lib/chat/messageMapper.ts:333-371` - Tool invocation detection

---

## Related Research

- Feature parity research: `thoughts/shared/research/2025-12-06-chartsmith-ai-sdk-feature-parity.md`
- PR3.2 implementation plan: `thoughts/shared/plans/2025-12-06-pr3.2-feature-parity-fixes.md`

---

## Open Questions

1. **Should validateChart be buffered?** Probably not - it's read-only like getChartContext. But need to confirm validation doesn't modify anything.

2. **How should validation results appear?** As a tool result in the message (like conversion progress)? Or as a followup action result?

3. **Should we reuse ProviderSelector or create LiveProviderSwitcher?** The existing component has the dropdown logic but assumes locking. May be easier to modify than create new.

4. **Does kube-score need to be pre-installed?** The spec handles missing binary gracefully, but should we document installation requirements?
