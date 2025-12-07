# PR4 Supplemental Implementation Details

This document contains component pseudocode and rendering logic for reference during implementation. Primary specs are in PR4_Product_PRD.md and PR4_Tech_PRD.md.

> **Note**: All file paths in this document are relative to the `chartsmith/` directory.

---

## ValidationResults Component

**File**: `chartsmith-app/components/chat/ValidationResults.tsx`

**Props**:
- `result`: ValidationResult object from Go backend

**Pattern**: Follow `PlanChatMessage.tsx` structure

**Render Logic**:
```
COMPONENT ValidationResults(result):
  STATE expandedIssues = Set<string>()
  STATE showFullDetails = false

  // Status badge with color
  statusColor = SWITCH result.overall_status:
    "pass" -> "text-green-500 bg-green-500/10"
    "warning" -> "text-yellow-500 bg-yellow-500/10"
    "fail" -> "text-red-500 bg-red-500/10"

  // Collect all issues from all sources
  allIssues = [
    ...result.results.helm_lint?.issues || [],
    ...result.results.helm_template?.issues || [],
    ...result.results.kube_score?.issues || []
  ]

  // Sort by severity (critical first)
  sortedIssues = allIssues.sort(bySeverity)

  RENDER Card:
    // Header
    <div className="flex items-center justify-between">
      <span className="font-medium">Validation Results</span>
      <Badge className={statusColor}>{result.overall_status}</Badge>
    </div>

    // Summary
    <div className="text-sm text-muted-foreground">
      {countBySeverity(allIssues)} // e.g., "1 critical, 2 warnings"
    </div>

    // Issues list
    IF sortedIssues.length > 0:
      FOR each issue in sortedIssues:
        <IssueItem
          issue={issue}
          expanded={expandedIssues.has(issue.id)}
          onToggle={() => toggleExpanded(issue.id)}
        />
    ELSE:
      <div className="text-green-500">No issues found</div>

    // Kube-score summary
    IF result.results.kube_score:
      <div className="border-t pt-2 mt-2">
        <span>Best Practices Score: </span>
        <span className="font-medium">
          {result.results.kube_score.score}/10
        </span>
        <span className="text-muted-foreground">
          ({result.results.kube_score.passed_checks}/{result.results.kube_score.total_checks} checks)
        </span>
      </div>

    // Actions
    <div className="flex gap-2 mt-2">
      <Button variant="ghost" onClick={() => setShowFullDetails(!showFullDetails)}>
        {showFullDetails ? "Hide" : "Show"} details
      </Button>
    </div>
```

**IssueItem Subcomponent**:
```
COMPONENT IssueItem(issue, expanded, onToggle):
  severityIcon = SWITCH issue.severity:
    "critical" -> <AlertCircle className="text-red-500" />
    "warning" -> <AlertTriangle className="text-yellow-500" />
    "info" -> <Info className="text-blue-500" />

  RENDER:
    <div
      className="p-2 rounded cursor-pointer hover:bg-muted/50"
      onClick={onToggle}
    >
      <div className="flex items-start gap-2">
        {severityIcon}
        <div className="flex-1">
          <div className="font-medium">{issue.message}</div>
          IF issue.file:
            <div className="text-xs text-muted-foreground">
              {issue.file}{issue.line ? `:${issue.line}` : ""}
            </div>
        </div>
        <ChevronIcon direction={expanded ? "up" : "down"} />
      </div>

      IF expanded:
        <div className="mt-2 pl-6 text-sm">
          <div className="text-muted-foreground">
            Source: {issue.source}
            {issue.check && ` (${issue.check})`}
          </div>
          IF issue.suggestion:
            <div className="mt-1 p-2 bg-muted rounded">
              <span className="font-medium">Fix: </span>
              {issue.suggestion}
            </div>
        </div>
    </div>
```

---

## LiveProviderSwitcher Component

**File**: `chartsmith-app/components/chat/LiveProviderSwitcher.tsx`

**Props**:
- `currentProvider`: string - Currently selected provider ID
- `currentModel`: string - Currently selected model ID
- `onSwitch`: (provider: string, model: string) => void
- `disabled?`: boolean - Disable during streaming

**Pattern**: Similar to existing ProviderSelector.tsx but always visible

**Render Logic**:
```
COMPONENT LiveProviderSwitcher(currentProvider, currentModel, onSwitch, disabled):
  STATE isOpen = false
  REF dropdownRef = useRef()

  // Get display info
  providerConfig = getProviderById(currentProvider)
  modelConfig = getModelById(currentModel)

  // Click outside to close
  useEffect(() => {
    function handleClickOutside(event) {
      if (dropdownRef.current && !dropdownRef.current.contains(event.target)) {
        setIsOpen(false)
      }
    }
    document.addEventListener("mousedown", handleClickOutside)
    return () => document.removeEventListener("mousedown", handleClickOutside)
  }, [])

  FUNCTION handleSelect(provider, model):
    onSwitch(provider, model)
    setIsOpen(false)

  RENDER:
    <div ref={dropdownRef} className="relative">
      // Trigger button
      <Button
        variant="ghost"
        onClick={() => !disabled && setIsOpen(!isOpen)}
        disabled={disabled}
        className="flex items-center gap-1"
      >
        <ProviderIcon provider={currentProvider} />
        <span>{modelConfig?.name || currentModel}</span>
        <ChevronDown className={isOpen ? "rotate-180" : ""} />
      </Button>

      // Dropdown
      IF isOpen:
        <div className="absolute right-0 mt-1 w-56 rounded-md border bg-popover shadow-lg z-50">
          FOR each provider in AVAILABLE_PROVIDERS:
            <div className="px-2 py-1 text-xs text-muted-foreground">
              {provider.name}
            </div>
            FOR each model in getModelsForProvider(provider.id):
              <div
                className="flex items-center gap-2 px-3 py-2 cursor-pointer hover:bg-muted"
                onClick={() => handleSelect(provider.id, model.modelId)}
              >
                <Check
                  className={currentModel === model.modelId ? "opacity-100" : "opacity-0"}
                />
                <span>{model.name}</span>
                IF model.modelId === currentModel:
                  <span className="text-xs text-muted-foreground">(current)</span>
              </div>
    </div>
```

**ProviderIcon Helper**:
```
FUNCTION ProviderIcon({ provider }):
  SWITCH provider:
    "anthropic" -> <Bot className="w-4 h-4" />
    "openai" -> <Cpu className="w-4 h-4" />
    DEFAULT -> <Cpu className="w-4 h-4" />
```

---

## useAISDKChatAdapter Changes

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Current State** (lines 138-148):
```typescript
// BEFORE - hardcoded values
const getChatBody = useCallback(
  (messageId?: string) => ({
    provider: DEFAULT_PROVIDER,  // Always "anthropic"
    model: DEFAULT_MODEL,        // Always "anthropic/claude-sonnet-4"
    workspaceId,
    revisionNumber,
    persona: currentPersonaRef.current,
    chatMessageId: messageId,
  }),
  [workspaceId, revisionNumber]
);
```

**Required Changes**:
```typescript
// AFTER - dynamic provider state

// Add state near top of hook (after existing state declarations)
const [selectedProvider, setSelectedProvider] = useState<string>(DEFAULT_PROVIDER);
const [selectedModel, setSelectedModel] = useState<string>(DEFAULT_MODEL);

// Update getChatBody to use state
const getChatBody = useCallback(
  (messageId?: string) => ({
    provider: selectedProvider,     // Now from state
    model: selectedModel,           // Now from state
    workspaceId,
    revisionNumber,
    persona: currentPersonaRef.current,
    chatMessageId: messageId,
  }),
  [workspaceId, revisionNumber, selectedProvider, selectedModel]
);

// Add switch function
const switchProvider = useCallback((provider: string, model: string) => {
  setSelectedProvider(provider);
  setSelectedModel(model);
}, []);

// Update return value
return {
  // ... existing returns
  selectedProvider,
  selectedModel,
  switchProvider,
};
```

---

## ChatContainer Integration

**File**: `chartsmith-app/components/ChatContainer.tsx`

**Current State**: Has role selector (persona) but no provider selector

**Add Import**:
```typescript
import { LiveProviderSwitcher } from "@/components/chat/LiveProviderSwitcher";
```

**Update Hook Usage** (get new values from adapter):
```typescript
const {
  // ... existing destructuring
  selectedProvider,
  selectedModel,
  switchProvider,
} = useAISDKChatAdapter(/* ... */);
```

**Add to Form Button Area** (integration point is line 214: `<div className="absolute right-4 top-[18px] flex gap-2">`):

The provider switcher should be added inside the existing flex container alongside the role selector:
```typescript
{/* Line 214: <div className="absolute right-4 top-[18px] flex gap-2"> */}
  <LiveProviderSwitcher
    currentProvider={selectedProvider}
    currentModel={selectedModel}
    onSwitch={switchProvider}
    disabled={status === "streaming"}
  />

  {/* Existing role selector (lines 216-276) */}
  <div ref={roleMenuRef} className="relative">
    {/* ... existing role dropdown ... */}
  </div>
{/* </div> */}
```

---

## validateChart Tool Definition

**File**: `chartsmith-app/lib/ai/tools/validateChart.ts`

**Complete Implementation**:
```typescript
import { tool } from "ai";
import { z } from "zod";
import { callGoEndpoint } from "./utils";

interface ValidationIssue {
  severity: "critical" | "warning" | "info";
  source: "helm_lint" | "helm_template" | "kube_score";
  resource?: string;
  check?: string;
  message: string;
  file?: string;
  line?: number;
  suggestion?: string;
}

interface ValidationResult {
  overall_status: "pass" | "warning" | "fail";
  timestamp: string;
  duration_ms: number;
  results: {
    helm_lint: {
      status: "pass" | "fail";
      issues: ValidationIssue[];
    };
    helm_template: {
      status: "pass" | "fail";
      rendered_resources: number;
      output_size_bytes: number;
      issues: ValidationIssue[];
    };
    kube_score?: {
      status: "pass" | "warning" | "fail" | "skipped";
      score: number;
      total_checks: number;
      passed_checks: number;
      issues: ValidationIssue[];
    };
  };
}

interface ValidationResponse {
  validation: ValidationResult;
}

export function createValidateChartTool(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number
) {
  return tool({
    description:
      "Validate a Helm chart for syntax errors, template rendering issues, and " +
      "Kubernetes best practices. Use this tool when the user asks to validate, " +
      "check, lint, verify, or review their chart, or when they ask about errors, " +
      "issues, or problems with the chart.",
    inputSchema: z.object({
      values: z
        .record(z.any())
        .optional()
        .describe("Values to override for template rendering"),
      strictMode: z
        .boolean()
        .optional()
        .describe("Treat warnings as failures"),
      kubeVersion: z
        .string()
        .optional()
        .describe("Target Kubernetes version (e.g., '1.28')"),
    }),
    execute: async (params: {
      values?: Record<string, unknown>;
      strictMode?: boolean;
      kubeVersion?: string;
    }) => {
      try {
        const response = await callGoEndpoint<ValidationResponse>(
          "/api/validate",
          {
            workspaceId,
            revisionNumber,
            values: params.values,
            strictMode: params.strictMode,
            kubeVersion: params.kubeVersion,
          },
          authHeader
        );
        return response;
      } catch (error) {
        return {
          success: false,
          message:
            error instanceof Error
              ? error.message
              : "Validation failed unexpectedly",
        };
      }
    },
  });
}
```

**Register in tools/index.ts**:
```typescript
import { createValidateChartTool } from "./validateChart";

export function createTools(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number,
  chatMessageId?: string
) {
  const tools: Record<string, Tool> = {
    getChartContext: createGetChartContextTool(workspaceId, revisionNumber, authHeader),
    textEditor: createTextEditorTool(authHeader, workspaceId, revisionNumber),
    latestSubchartVersion: createLatestSubchartVersionTool(authHeader),
    latestKubernetesVersion: createLatestKubernetesVersionTool(authHeader),
    validateChart: createValidateChartTool(authHeader, workspaceId, revisionNumber), // NEW
  };

  if (chatMessageId) {
    tools.convertK8sToHelm = createConvertK8sTool(authHeader, workspaceId, chatMessageId);
  }

  return tools;
}
```

**Also add to bufferedTools.ts** (as non-buffered):
```typescript
// In createBufferedTools, add validateChart alongside other read-only tools
validateChart: createValidateChartTool(authHeader, workspaceId, revisionNumber),
```

---

## Tool Result Rendering

**IMPORTANT**: The codebase uses **property-based detection** for tool results, NOT tool-result part parsing. This follows the pattern used by `responsePlanId`, `responseRenderId`, and `responseConversionId`.

### Step 1: Update Message Interface

**File**: `chartsmith-app/components/types.ts`

Add the new property to the Message interface (around line 35):
```typescript
export interface Message {
  // ... existing properties (id, prompt, response, etc.)
  responseRenderId?: string;
  responsePlanId?: string;
  responseConversionId?: string;
  responseValidationId?: string;  // NEW: Add this property
  // ... rest of properties
}
```

### Step 2: Create Validation Atoms

**File**: `chartsmith-app/atoms/validationAtoms.ts` (NEW)

```typescript
import { atom } from 'jotai';
import type { ValidationResult } from '@/lib/ai/tools/validateChart';

export interface ValidationData {
  id: string;
  result: ValidationResult;
  workspaceId: string;
  timestamp: Date;
}

// Store all validations
export const validationsAtom = atom<ValidationData[]>([]);

// Getter atom for fetching by ID
export const validationByIdAtom = atom((get) => {
  const validations = get(validationsAtom);
  return (id: string) => validations.find(v => v.id === id);
});

// Action atom for adding/updating validations
export const handleValidationUpdatedAtom = atom(
  null,
  (get, set, validation: ValidationData) => {
    const current = get(validationsAtom);
    const existing = current.findIndex(v => v.id === validation.id);
    if (existing >= 0) {
      const updated = [...current];
      updated[existing] = validation;
      set(validationsAtom, updated);
    } else {
      set(validationsAtom, [...current, validation]);
    }
  }
);
```

### Step 3: Update ChatMessage.tsx

**File**: `chartsmith-app/components/ChatMessage.tsx`

Add validation data fetching (after other atom getters, around line 95-101):
```typescript
import { validationByIdAtom } from '@/atoms/validationAtoms';

// Inside component, after other atom getters:
const [validationGetter] = useAtom(validationByIdAtom);
const validation = message?.responseValidationId
  ? validationGetter(message.responseValidationId)
  : undefined;
```

Add to SortedContent render (after conversion results section, around line 204-217):
```typescript
{/* 5. Validation results (if responseValidationId exists) */}
{message?.responseValidationId && (
  <div className="mt-4">
    {(message.response || message.responsePlanId || message.responseRenderId || message.responseConversionId) && (
      <div className="border-t border-gray-200 dark:border-dark-border/30 pt-4 mb-2">
        <div className="text-xs text-gray-500 dark:text-gray-400 mb-2">Validation Results:</div>
      </div>
    )}
    {validation ? (
      <ValidationResults validationId={message.responseValidationId} />
    ) : (
      <LoadingSpinner message="Loading validation results..." />
    )}
  </div>
)}
```

### Step 4: Update ValidationResults Component Props

**File**: `chartsmith-app/components/chat/ValidationResults.tsx`

The component should receive the validation ID and fetch data from atoms:
```typescript
interface ValidationResultsProps {
  validationId: string;  // Changed from `result: ValidationResult`
}

export function ValidationResults({ validationId }: ValidationResultsProps) {
  const [validationGetter] = useAtom(validationByIdAtom);
  const validationData = validationGetter(validationId);

  if (!validationData) {
    return <LoadingSpinner message="Loading validation..." />;
  }

  const result = validationData.result;
  // ... rest of component using `result`
}
```

### Pattern Summary

The rendering flow is:
1. Tool execute() returns result with a unique validation ID
2. Result is stored in validationsAtom via handleValidationUpdatedAtom
3. Message gets `responseValidationId` set to the validation ID
4. ChatMessage detects `responseValidationId` and renders ValidationResults
5. ValidationResults fetches full data from validationByIdAtom using the ID

---

## Grade to Severity Mapping (Go)

**File**: `pkg/validation/kubescore.go`

```go
func mapGradeToSeverity(grade int) string {
    switch {
    case grade <= 1:
        return "critical"
    case grade <= 5:
        return "warning"
    default:
        return "" // Grade 7-10 = pass, don't include as issue
    }
}
```

---

## Suggestion Lookup Table (Go)

**File**: `pkg/validation/kubescore.go`

```go
var suggestions = map[string]string{
    "container-resources":         "Add resources.limits.memory and resources.limits.cpu to container spec",
    "container-security-context":  "Set securityContext.runAsNonRoot: true and readOnlyRootFilesystem: true",
    "container-image-tag":         "Use specific image tag instead of :latest (e.g., nginx:1.21.0)",
    "pod-probes":                  "Add readinessProbe and livenessProbe to containers",
    "pod-networkpolicy":           "Create a NetworkPolicy for this namespace",
    "deployment-replicas":         "Set replicas >= 2 for high availability",
    "container-ephemeral-storage": "Set resources.limits.ephemeral-storage to prevent disk exhaustion",
}

func getSuggestion(checkName string) string {
    if suggestion, ok := suggestions[checkName]; ok {
        return suggestion
    }
    return "Review Kubernetes best practices documentation for this check"
}
```

---

## Validation Progress Component

**File**: `chartsmith-app/components/chat/ValidationProgress.tsx` (NEW - Optional)

For showing progress during validation:

```typescript
interface ValidationProgressProps {
  stages: {
    name: string;
    status: "pending" | "running" | "complete" | "failed";
  }[];
}

export function ValidationProgress({ stages }: ValidationProgressProps) {
  return (
    <div className="rounded-lg border p-4">
      <div className="flex items-center gap-2 mb-3">
        <Loader2 className="w-4 h-4 animate-spin" />
        <span className="font-medium">Validating chart...</span>
      </div>
      <div className="space-y-2">
        {stages.map((stage) => (
          <div key={stage.name} className="flex items-center gap-2 text-sm">
            {stage.status === "complete" && <Check className="w-4 h-4 text-green-500" />}
            {stage.status === "running" && <Loader2 className="w-4 h-4 animate-spin" />}
            {stage.status === "pending" && <Circle className="w-4 h-4 text-muted-foreground" />}
            {stage.status === "failed" && <X className="w-4 h-4 text-red-500" />}
            <span className={stage.status === "pending" ? "text-muted-foreground" : ""}>
              {stage.name}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## File Summary

### Files to Create

**Go Backend**:
- `pkg/validation/types.go`
- `pkg/validation/pipeline.go`
- `pkg/validation/helm.go`
- `pkg/validation/kubescore.go`
- `pkg/validation/parser.go`
- `pkg/api/handlers/validate.go`

**Frontend**:
- `chartsmith-app/lib/ai/tools/validateChart.ts`
- `chartsmith-app/components/chat/ValidationResults.tsx`
- `chartsmith-app/components/chat/LiveProviderSwitcher.tsx`
- `chartsmith-app/components/chat/ValidationProgress.tsx` (optional)
- `chartsmith-app/atoms/validationAtoms.ts` - Jotai atoms for validation state

### Files to Modify

**Go Backend**:
- `pkg/api/server.go` - Add route registration (lines 18-38 is the route block)

**Frontend**:
- `chartsmith-app/lib/ai/tools/index.ts` - Register validateChart
- `chartsmith-app/lib/ai/tools/bufferedTools.ts` - Add validateChart
- `chartsmith-app/hooks/useAISDKChatAdapter.ts` - Add provider state
- `chartsmith-app/components/ChatContainer.tsx` - Add LiveProviderSwitcher (integration point: line 214)
- `chartsmith-app/components/ChatMessage.tsx` - Add validation result rendering (SortedContent component)
- `chartsmith-app/components/types.ts` - Add `responseValidationId` to Message interface

---

*Document End - PR4 Supplemental Details*
