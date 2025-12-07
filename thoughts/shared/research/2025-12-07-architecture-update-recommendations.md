---
date: 2025-12-07T16:45:00-08:00
researcher: Claude Code
git_commit: 1ab743a203f6b2d491d7dd5acbc6a43ead81112d
branch: main
repository: UnchartedTerritory
topic: "ARCHITECTURE.md Update Recommendations for AI SDK Migration"
tags: [research, codebase, architecture, ai-sdk, vercel, migration]
status: complete
last_updated: 2025-12-07
last_updated_by: Claude Code
---

# Research: ARCHITECTURE.md Update Recommendations

**Date**: 2025-12-07T16:45:00-08:00
**Researcher**: Claude Code
**Git Commit**: 1ab743a203f6b2d491d7dd5acbc6a43ead81112d
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

Analyze the current codebase state after the Vercel AI SDK migration and determine how to update the ARCHITECTURE.md file to reflect the new implementation.

## Summary

The migration to Vercel AI SDK is **complete** and meets all requirements specified in `Replicated_Chartsmith.md`. The existing `chartsmith-app/ARCHITECTURE.md` file is partially updated but needs consolidation with content from `docs/ARCHITECTURE.md` and should be expanded to include PR3.0, PR4, and provider switching features.

## Requirements Verification

### Must Have Requirements - All Met ✅

| Requirement | Status | Evidence |
|-------------|--------|----------|
| Replace custom chat UI with Vercel AI SDK | ✅ Complete | `@ai-sdk/react` v2.0.106 with `useChat` hook in `useAISDKChatAdapter.ts` |
| Migrate from @anthropic-ai/sdk to AI SDK Core | ✅ Complete | `ai` v5.0.106 with `streamText()` in `app/api/chat/route.ts` |
| Maintain all existing chat functionality | ✅ Complete | Streaming, messages, history all work via adapter pattern |
| Keep existing system prompts and behavior | ✅ Complete | `lib/ai/prompts.ts` with persona-specific prompts |
| All features continue to work (tool calling, file context) | ✅ Complete | 5 tools implemented in `lib/ai/tools/` |
| Tests pass | ✅ Complete | 94 unit tests pass, integration tests exist |

### Nice to Have Requirements - All Met ✅

| Requirement | Status | Evidence |
|-------------|--------|----------|
| Easy provider switching | ✅ Complete | `LiveProviderSwitcher.tsx`, multi-provider support via OpenRouter |
| Improved streaming experience | ✅ Complete | AI SDK throttling (50ms), optimized state updates |
| Simplified state management | ✅ Complete | AI SDK manages chat state, Jotai for persistence |

## Current Architecture Files Analysis

### 1. `/chartsmith/ARCHITECTURE.md` (Root)
- **Status**: Minimal, generic
- **Content**: Basic principles, mentions subproject
- **Recommendation**: Keep as-is, references `chartsmith-app/ARCHITECTURE.md`

### 2. `/chartsmith/chartsmith-app/ARCHITECTURE.md`
- **Status**: Partially updated (covers PR1, PR1.5, PR2.0)
- **Missing**: PR3.0 features, PR4 features, validation tool
- **Issue**: Mentions outdated model (claude-sonnet-4 → claude-sonnet-4)

### 3. `/chartsmith/docs/ARCHITECTURE.md`
- **Status**: More complete but separate from main file
- **Content**: Covers plan workflow, execution paths, migration progress
- **Recommendation**: Merge into `chartsmith-app/ARCHITECTURE.md`

## Recommended ARCHITECTURE.md Updates

The `chartsmith-app/ARCHITECTURE.md` should be updated with the following changes:

### Section 1: Update AI SDK Package Versions

**Current (outdated):**
```env
DEFAULT_AI_MODEL=anthropic/claude-sonnet-4
```

**Update to reflect current `package.json`:**
```env
# AI SDK Packages
ai: ^5.0.106
@ai-sdk/react: ^2.0.106
@ai-sdk/anthropic: ^2.0.53
@ai-sdk/openai: ^2.0.77
@openrouter/ai-sdk-provider: ^1.3.0
```

### Section 2: Add PR3.0 Plan Workflow Section

Add after the PR2.0 section:

```markdown
## AI SDK Plan Workflow (PR3.0)

PR3.0 introduces plan-based approval for file modifications, matching legacy Go worker behavior.

### Intent Classification

User prompts are classified via Go backend before AI SDK processing:

| Intent | Route | Behavior |
|--------|-------|----------|
| `off-topic` | Immediate decline | No LLM call |
| `proceed` | Execute plan | Run buffered tools |
| `render` | Trigger render | Chart preview |
| `plan` | Plan mode | AI SDK without tools |
| `ai-sdk` | Normal | AI SDK with tools |

### Buffered Tool Calls

File-modifying operations are buffered instead of executed immediately:

| Tool | Command | Behavior |
|------|---------|----------|
| `textEditor` | `view` | Execute immediately |
| `textEditor` | `create` | Buffer for plan |
| `textEditor` | `str_replace` | Buffer for plan |
| Other tools | * | Execute immediately |

### Plan Creation Flow

1. User sends message to `/api/chat`
2. Intent classified via Go `/api/intent/classify`
3. AI SDK streams response with buffered tools
4. `onFinish` callback creates plan via Go `/api/plan/create-from-tools`
5. Frontend shows `PlanChatMessage` with Proceed/Ignore
6. User approval executes buffered tools via `proceedPlanAction()`

### Key Files (PR3.0)

| File | Purpose |
|------|---------|
| `lib/ai/intent.ts` | Intent classification client |
| `lib/ai/plan.ts` | Plan creation client |
| `lib/ai/tools/bufferedTools.ts` | Buffered tool implementations |
| `lib/ai/tools/toolInterceptor.ts` | Buffer management |
| `lib/workspace/actions/proceed-plan.ts` | Execute buffered tools |
| `pkg/api/handlers/intent.go` | Intent endpoint |
| `pkg/api/handlers/plan.go` | Plan creation endpoint |
```

### Section 3: Add PR4 Validation Tool Section

```markdown
## Chart Validation (PR4)

PR4 adds the `validateChart` tool for helm lint, helm template, and kube-score validation.

### Validation Tool

| Tool | Endpoint | Purpose |
|------|----------|---------|
| `validateChart` | `/api/validate` | Run validation pipeline |

### Validation Response

```typescript
{
  overall_status: "pass" | "warning" | "fail";
  results: {
    helm_lint: LintResult;
    helm_template?: TemplateResult;
    kube_score?: ScoreResult;
  };
}
```

### Validation UI

- `components/chat/ValidationResults.tsx` displays results
- Issues grouped by severity (critical > warning > info)
- Collapsible detail panels
```

### Section 4: Add Live Provider Switching Section

```markdown
## Live Provider Switching (PR4)

Users can switch AI providers mid-conversation without losing context.

### Supported Providers

| Provider | Models |
|----------|--------|
| Anthropic | Claude Sonnet 4 (default) |
| OpenAI | GPT-4o, GPT-4o Mini |

### Provider Priority

When `USE_OPENROUTER_PRIMARY=true` (default):
1. OpenRouter (unified gateway)
2. Direct Anthropic API (fallback)
3. Direct OpenAI API (fallback)

### Key Components

| Component | Purpose |
|-----------|---------|
| `LiveProviderSwitcher.tsx` | Mid-conversation provider selection |
| `lib/ai/provider.ts` | Provider factory with fallback logic |
| `lib/ai/models.ts` | Model definitions |

### Usage Flow

1. User clicks LiveProviderSwitcher in chat header
2. Selects new provider/model
3. `switchProvider()` updates adapter state
4. Next message uses new provider
5. Conversation context preserved
```

### Section 5: Update Tools Summary Table

Update the existing tools table to include all 6 tools:

```markdown
### Tool Summary

| Tool | TypeScript | Go Endpoint | Purpose |
|------|------------|-------------|---------|
| `getChartContext` | `getChartContext.ts` | `/api/tools/context` | Load workspace files |
| `textEditor` | `textEditor.ts` | `/api/tools/editor` | View/create/edit files |
| `latestSubchartVersion` | `latestSubchartVersion.ts` | `/api/tools/versions/subchart` | Subchart versions |
| `latestKubernetesVersion` | `latestKubernetesVersion.ts` | `/api/tools/versions/kubernetes` | K8s version info |
| `validateChart` | `validateChart.ts` | `/api/validate` | Chart validation |
| `convertK8sToHelm` | `convertK8s.ts` | `/api/conversion/start` | K8s to Helm conversion |
```

### Section 6: Update Go Backend Endpoints

Add complete endpoint list:

```markdown
## Go HTTP API Endpoints

### Tool Endpoints
- `POST /api/tools/editor` - Text file operations
- `POST /api/tools/versions/subchart` - Subchart version lookup
- `POST /api/tools/versions/kubernetes` - K8s version lookup
- `POST /api/tools/context` - Chart context retrieval

### Intent/Plan Endpoints
- `POST /api/intent/classify` - Intent classification via Groq
- `POST /api/plan/create-from-tools` - Create plan from buffered calls
- `POST /api/plan/update-action-file-status` - Update file status
- `POST /api/plan/publish-update` - Publish plan event

### Other Endpoints
- `POST /api/conversion/start` - Start K8s conversion
- `POST /api/validate` - Run chart validation
- `GET /health` - Health check
```

### Section 7: Update Migration Progress

```markdown
## Migration Progress

- [x] AI SDK chat integration (PR1)
- [x] Tool support via Go HTTP (PR1.5)
- [x] Main workspace integration (PR2.0)
- [x] Intent classification (PR3.0)
- [x] Plan workflow parity (PR3.0)
- [x] K8s conversion bridge (PR3.0)
- [x] Chart validation tool (PR4)
- [x] Live provider switching (PR4)
- [ ] Remove legacy Go LLM code (future)
```

## Recommended Action

Replace the content of `/chartsmith/chartsmith-app/ARCHITECTURE.md` with an updated version that incorporates all the sections above while preserving the existing structure and conventions.

## Code References

- `chartsmith/chartsmith-app/package.json:20-30` - AI SDK dependencies
- `chartsmith/chartsmith-app/lib/ai/` - AI module directory
- `chartsmith/chartsmith-app/app/api/chat/route.ts` - Chat API route
- `chartsmith/chartsmith-app/hooks/useAISDKChatAdapter.ts` - Chat adapter hook
- `chartsmith/chartsmith-app/lib/ai/tools/` - Tool implementations
- `chartsmith/pkg/api/server.go:18-41` - Go route registration
- `chartsmith/pkg/api/handlers/` - Go endpoint handlers
