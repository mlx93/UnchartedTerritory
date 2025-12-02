# PR2 Implementation Instructions

**Prerequisite**: PR1 merged and working

## Pre-Implementation: Verify Go Backend Structure (30 min)

### Step 1: Explore Existing Go Packages
```bash
cd chartsmith

# Find existing Go package structure
find . -name "*.go" -type f | head -20

# Check for existing API handlers
find . -path "*/api/*" -name "*.go"
find . -path "*/pkg/*" -type d

# Look for existing router/server setup
grep -r "http.Handle\|mux\|router" --include="*.go"
```

### Step 2: Verify CLI Tool Availability
```bash
# Check helm is installed
helm version

# Check kube-score is installed (install if missing)
kube-score version || go install github.com/zegl/kube-score/cmd/kube-score@latest

# Verify both work with test chart
helm lint test_chart/
helm template test-release test_chart/ | kube-score score -
```

### Step 3: Identify Integration Points
Document answers to:
- [ ] Where is the Go HTTP server defined?
- [ ] How are routes registered?
- [ ] What's the existing package naming convention?
- [ ] Is there an existing `/api/` prefix pattern?

### Step 4: Verify PR1 Tool Calling Setup
```bash
cd chartsmith-app

# Confirm streamText is configured
grep -r "streamText" --include="*.ts" --include="*.tsx"

# Check if tools parameter exists in route
grep -r "tools:" --include="*.ts" src/app/api/
```

---

## PRD Verification Checklist

Before coding, confirm these assumptions from PRDs:

### Go Backend (PRD_Tech_PR2.md)
- [ ] `pkg/` directory exists and is the right place for new packages
- [ ] Can add new route to existing server without major refactor
- [ ] Test charts exist in `test_chart/` or `testdata/`

### Frontend (PRD_Product_PR2.md)
- [ ] useChat hook from PR1 supports `tools` option
- [ ] Message parts rendering exists from PR1
- [ ] Provider selector component exists to upgrade

---

## Implementation Sequence (Days 4-6)

### Day 4: Go Validation Pipeline
**Morning**: 
1. Create `pkg/validation/` with types.go, helm.go
2. Implement `runHelmLint` and `runHelmTemplate`
3. Unit test with `test_chart/`

**Afternoon**:
1. Implement `kubescore.go` 
2. Implement `pipeline.go` orchestration
3. Integration test full pipeline

### Day 5: API + Frontend Integration
**Morning**:
1. Create `/api/validate` handler in `pkg/api/validate.go`
2. Register route, test with curl
3. Create `validateChart` tool definition

**Afternoon**:
1. Build `ValidationResults` component
2. Build `LiveProviderSwitcher` component
3. Wire tool result rendering in message component

### Day 6: Polish + Demo
**Morning**:
1. End-to-end testing (chat → tool → Go → results)
2. Test error scenarios (missing tools, bad charts)
3. Fix bugs

**Afternoon**:
1. Update ARCHITECTURE.md
2. Record demo video
3. Final PR prep

---

## Key Files to Create

| File | Purpose |
|------|---------|
| `pkg/validation/types.go` | All Go type definitions |
| `pkg/validation/helm.go` | Lint and template functions |
| `pkg/validation/kubescore.go` | Kube-score integration |
| `pkg/validation/pipeline.go` | Orchestration logic |
| `pkg/api/validate.go` | HTTP handler |
| `src/lib/tools/validateChart.ts` | AI SDK tool definition |
| `src/components/chat/ValidationResults.tsx` | Results display |
| `src/components/chat/LiveProviderSwitcher.tsx` | Provider dropdown |

## Validation Checklist
- [ ] "Validate my chart" triggers tool invocation
- [ ] Go backend runs helm lint → template → kube-score
- [ ] Results display with severity colors
- [ ] AI explains issues in natural language
- [ ] Live provider switching preserves history
- [ ] All PR1 functionality still works
