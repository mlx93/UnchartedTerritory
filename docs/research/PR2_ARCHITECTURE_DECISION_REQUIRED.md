# PR2 Architecture Decision Required

## Critical Decision: Frontend ↔ Go Backend Communication for Validation Tool

**Status**: DECIDED - Option A (HTTP Endpoint)
**Impact**: HIGH - Affects entire PR2 implementation approach
**Date**: 2024-12-01
**Last Updated**: 2024-12-01

**Decision Date**: 2025-12-02
**Rationale**: Synchronous response required for AI SDK tool execute(). User confirmed "API adjustments should be in PR2".

---

## Context

The PR2 PRDs describe a `validateChart` tool that needs to:
1. Accept a request from the AI (via Vercel AI SDK tool calling)
2. Execute `helm lint`, `helm template`, and `kube-score` on a chart
3. Return structured results to the AI for interpretation

**The Problem**: The PRDs assumed HTTP REST communication between Next.js and Go, but the actual Chartsmith architecture uses **PostgreSQL LISTEN/NOTIFY + Centrifugo WebSocket**. The Go backend has **no HTTP server**.

### Current Chartsmith Communication Pattern

```
┌─────────────────┐                    ┌─────────────────┐
│   Next.js App   │                    │   Go Worker     │
│   (port 3000)   │                    │   (NO HTTP!)    │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │ INSERT + pg_notify()                 │ LISTEN
         │                                      │
         └──────────────┬───────────────────────┘
                        │
              ┌─────────▼─────────┐
              │   PostgreSQL      │
              │   work_queue      │
              └─────────┬─────────┘
                        │
              ┌─────────▼─────────┐
              │   Centrifugo      │
              │   WebSocket       │
              └───────────────────┘
```

**Key Files** (from codebase analysis):
- `chartsmith-app/lib/utils/queue.ts:9-20` - Enqueue work via pg_notify
- `chartsmith/pkg/listener/start.go:13-119` - Go LISTEN handlers
- `chartsmith/pkg/realtime/centrifugo.go:48-66` - Real-time event publishing

---

## Three Options for PR2

### Option A: Add HTTP Endpoint to Go Backend (New Pattern)

**Summary**: Add a minimal HTTP server to the Go backend specifically for the validation endpoint.

```
┌─────────────────┐                    ┌─────────────────┐
│   Next.js App   │   POST /validate   │   Go Backend    │
│   /api/chat     │ ──────────────────►│   NEW HTTP      │
│   (tool call)   │ ◄──────────────────│   Server        │
└─────────────────┘   JSON Response    └────────┬────────┘
                                                │
                                       ┌────────▼────────┐
                                       │   helm lint     │
                                       │   helm template │
                                       │   kube-score    │
                                       └─────────────────┘
```

**Implementation**:
1. Create `cmd/http.go` or add to existing `cmd/run.go`
2. Create `pkg/api/validate.go` with HTTP handler
3. Start HTTP server alongside PostgreSQL listener
4. Tool execute() calls Go directly via HTTP

**New Files**:
```
pkg/
├── api/
│   └── validate.go       # HTTP handler
└── validation/
    ├── pipeline.go       # Orchestration
    ├── helm.go           # helm lint/template
    └── kubescore.go      # kube-score
```

**Environment Variables**:
```env
GO_BACKEND_URL=http://localhost:8080  # New
```

**Pros**:
- Synchronous response - tool gets results immediately
- Simple integration with AI SDK tools
- Clean separation of concerns
- Easy to test independently

**Cons**:
- Introduces new pattern to codebase (HTTP in Go)
- Requires running additional server/port
- Diverges from existing architecture
- May need additional auth considerations

**Complexity**: Medium
**Risk**: Low (isolated change)

---

### Option B: Use Existing Queue Pattern (Async)

**Summary**: Add a `validate_chart` channel to the existing PostgreSQL LISTEN/NOTIFY system.

```
┌─────────────────┐                    ┌─────────────────┐
│   Next.js App   │                    │   Go Worker     │
│   /api/chat     │                    │   (existing)    │
│   (tool call)   │                    └────────┬────────┘
└────────┬────────┘                             │
         │                                      │ LISTEN
         │ pg_notify('validate_chart')          │ 'validate_chart'
         │                                      │
         └──────────────┬───────────────────────┘
                        │
              ┌─────────▼─────────┐
              │   PostgreSQL      │
              │   work_queue      │
              └─────────┬─────────┘
                        │
              ┌─────────▼─────────┐
              │   Centrifugo      │◄─────────── Results published
              │   WebSocket       │              to frontend
              └───────────────────┘
```

**Implementation**:
1. Add `validate_chart` handler to `pkg/listener/start.go`
2. Create `pkg/listener/validate-chart.go` for processing
3. Frontend enqueues work, waits for Centrifugo event
4. Tool execute() becomes async - polls or subscribes

**Changes to Existing Files**:
```go
// pkg/listener/start.go
case "validate_chart":
    l.handleValidateChart(ctx, conn, notification)
```

**Pros**:
- Consistent with existing architecture
- No new servers or ports
- Uses proven patterns
- Already have auth/context handling

**Cons**:
- Async complicates AI SDK tool calling
- Need to implement polling or WebSocket wait in tool
- Tool execute() must handle timeout/retry
- More complex UX (validation feels less immediate)

**Complexity**: High (async tool calling is tricky)
**Risk**: Medium (async UX may feel slow)

---

### Option C: Hybrid - Next.js API Wraps Go Execution

**Summary**: Keep validation logic in Go but call it from Next.js API route using spawn/exec or internal HTTP.

```
┌─────────────────┐                    ┌─────────────────┐
│   Next.js App   │                    │   Go Binary     │
│   /api/validate │ ──────────────────►│   (validate     │
│   (API route)   │   exec.spawn()     │    command)     │
│                 │ ◄──────────────────│                 │
└────────┬────────┘   stdout/stderr    └─────────────────┘
         │
         │ Tool calls this
┌────────▼────────┐
│   /api/chat     │
│   (AI SDK)      │
└─────────────────┘
```

**Implementation**:
1. Add `cmd/validate.go` CLI command to Go
2. Next.js `/api/validate` route spawns Go binary
3. Go runs validation, outputs JSON to stdout
4. Next.js parses and returns to tool

**Alternative Hybrid**: Internal HTTP between Next.js and Go (Go listens on localhost only)

**New Go Command**:
```go
// cmd/validate.go
var validateCmd = &cobra.Command{
    Use:   "validate",
    Short: "Run chart validation",
    Run: func(cmd *cobra.Command, args []string) {
        result := validation.RunValidation(chartPath, values)
        json.NewEncoder(os.Stdout).Encode(result)
    },
}
```

**Pros**:
- Synchronous from tool perspective
- Validation logic in Go (learning goal)
- No persistent HTTP server needed
- Clean separation

**Cons**:
- Spawning processes is slower
- Error handling more complex
- Need to ensure Go binary available in Next.js environment
- May have path/permission issues

**Complexity**: Medium
**Risk**: Medium (deployment considerations)

---

## Comparison Matrix

| Aspect | Option A: HTTP | Option B: Queue | Option C: Hybrid |
|--------|---------------|-----------------|------------------|
| **Synchronous?** | Yes | No | Yes |
| **Consistency with codebase** | Low (new pattern) | High | Medium |
| **Implementation complexity** | Medium | High | Medium |
| **Tool integration** | Easy | Hard | Medium |
| **UX feel** | Immediate | Async/polling | Immediate |
| **Deployment complexity** | Medium (new port) | Low | Medium (binary path) |
| **Testing ease** | Easy | Medium | Medium |
| **Go learning opportunity** | High | Medium | High |

---

## Recommendation

**For initial implementation, Option A (HTTP Endpoint) is recommended** because:

1. **Simplest tool integration** - AI SDK tools expect synchronous execute()
2. **Immediate UX** - User asks "validate my chart", gets instant feedback
3. **Clear learning path** - Writing HTTP handlers in Go is valuable experience
4. **Isolated change** - Doesn't affect existing queue-based features
5. **Easy to test** - Can curl the endpoint directly

However, if maintaining architectural consistency is paramount, Option B can work with additional complexity in the tool layer (implementing polling/timeout logic).

---

## Questions for Decision Maker

1. **Is adding HTTP to Go acceptable?** It's a new pattern but isolated to validation.

2. **Priority: consistency vs simplicity?** Queue is consistent but async; HTTP is simpler but new.

3. **Deployment constraints?** Can Go run an HTTP server in your environment?

4. **Future tools?** If more tools will call Go, HTTP may be worth the pattern change.

---

## After Decision: What Changes

### If Option A (HTTP) Chosen:

**PR2 Tech PRD changes**:
- Architecture diagram shows HTTP call to Go
- Go package structure includes `pkg/api/`
- Environment variables include `GO_BACKEND_URL`
- Tool definition uses `fetch()` to Go endpoint

### If Option B (Queue) Chosen:

**PR2 Tech PRD changes**:
- Architecture diagram shows pg_notify flow
- Tool execute() includes polling/timeout logic
- Add Centrifugo subscription for validation events
- More complex frontend state management

### If Option C (Hybrid) Chosen:

**PR2 Tech PRD changes**:
- Add `cmd/validate.go` CLI command
- Next.js `/api/validate` uses child_process or exec
- Error handling for process spawn failures
- Binary path configuration

---

## Related Documents

- `docs/research/2025-12-02-PRD-FALSE-ASSUMPTIONS-ANALYSIS.md` - Full analysis of PRD assumptions
- `docs/research/PR2_CODEBASE_FINDINGS.md` - Codebase architecture details
- `PRDs/PR2_GAPS_ANALYSIS.md` - Gap analysis with architecture findings
- `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` - Current architecture documentation

---

*Document created to facilitate architecture decision before PR2 implementation*
