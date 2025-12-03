# PR1.5 Go Handlers - Implementation Notes

**Scope**: 3 HTTP handlers for AI SDK tool execution

| Endpoint | Handler | Purpose |
|----------|---------|---------|
| `POST /api/tools/editor` | TextEditor | File view/create/str_replace |
| `POST /api/tools/versions/subchart` | GetSubchartVersion | ArtifactHub lookup |
| `POST /api/tools/versions/kubernetes` | GetKubernetesVersion | Hardcoded K8s version |

---

## Existing Functions to Reuse

**CRITICAL**: Reuse these production functions - do not reimplement.

### textEditor handler
```go
workspace.ListFiles(revisionID)                      // Get files for revision
workspace.GetFile(fileID)                            // Get file content
workspace.SetFileContentPending(fileID, content)     // Update file
workspace.AddFileToChart(chartID, path, content)     // Create new file
llm.PerformStringReplacement(content, oldStr, newStr) // Fuzzy str_replace
```

### latestSubchartVersion handler
```go
recommendations.GetLatestSubchartVersion(chartName)  // Returns version string or "?"
```

### latestKubernetesVersion handler
```go
// No external function - return hardcoded values:
// major: "1", minor: "1.32", patch: "1.32.1"
```

### Database connection (all handlers)
```go
conn := persistence.MustGetPooledPostgresSession()
defer conn.Release()
```

---

## Request/Response Contracts

### textEditor

```go
// Request
type TextEditorRequest struct {
    Command     string `json:"command"`     // "view" | "create" | "str_replace"
    WorkspaceID string `json:"workspaceId"`
    Path        string `json:"path"`
    Content     string `json:"content,omitempty"`  // create only
    OldStr      string `json:"oldStr,omitempty"`   // str_replace only
    NewStr      string `json:"newStr,omitempty"`   // str_replace only
}

// Response
type TextEditorResponse struct {
    Success bool   `json:"success"`
    Content string `json:"content,omitempty"`
    Message string `json:"message"`
}
```

**Command logic**:
- `view`: Return file content, or error "File does not exist. Use create instead."
- `create`: Create file, or error "File already exists. Use str_replace instead."
- `str_replace`: Replace text, or error "String not found in file."

### latestSubchartVersion

```go
// Request
type SubchartVersionRequest struct {
    ChartName  string `json:"chartName"`
    Repository string `json:"repository,omitempty"`
}

// Response
type SubchartVersionResponse struct {
    Success bool   `json:"success"`
    Version string `json:"version"`  // e.g., "15.2.0" or "?" if unknown
    Name    string `json:"name"`
}
```

### latestKubernetesVersion

```go
// Request
type KubernetesVersionRequest struct {
    SemverField string `json:"semverField,omitempty"` // "major"|"minor"|"patch"
}

// Response - hardcoded values
type KubernetesVersionResponse struct {
    Success bool   `json:"success"`
    Version string `json:"version"`  // "1" | "1.32" | "1.32.1"
    Field   string `json:"field"`
}
```

---

## Error Response Contract

```go
type ErrorResponse struct {
    Success bool   `json:"success"`          // always false
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`   // VALIDATION_ERROR, NOT_FOUND, etc.
}

func WriteError(w http.ResponseWriter, status int, code, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Success: false,
        Message: message,
        Code:    code,
    })
}
```

| Code | HTTP Status | When |
|------|-------------|------|
| VALIDATION_ERROR | 400 | Bad request params |
| NOT_FOUND | 404 | Workspace/file missing, OR unauthorized (security) |
| INTERNAL_ERROR | 500 | Unexpected error |
| EXTERNAL_API_ERROR | 502 | ArtifactHub failure |

---

## Auth Pattern (TO IMPLEMENT)

```go
// Extract and validate token - pattern to follow
func getAuthenticatedUserID(r *http.Request) (string, error) {
    authHeader := r.Header.Get("Authorization")
    if authHeader == "" {
        return "", errors.New("missing authorization header")
    }
    
    token := strings.TrimPrefix(authHeader, "Bearer ")
    
    // Validate against extension_token table
    // Return userID if valid, error if not
}
```

**Security**: For textEditor, verify user owns workspace. Return 404 (not 403) if unauthorized to avoid leaking workspace existence.

---

## Route Registration

```go
// pkg/api/server.go
func StartHTTPServer(ctx context.Context, port string) error {
    mux := http.NewServeMux()
    
    mux.HandleFunc("POST /api/tools/editor", handlers.TextEditor)
    mux.HandleFunc("POST /api/tools/versions/subchart", handlers.GetSubchartVersion)
    mux.HandleFunc("POST /api/tools/versions/kubernetes", handlers.GetKubernetesVersion)
    
    server := &http.Server{Addr: ":" + port, Handler: mux}
    return server.ListenAndServe()
}
```

Start in `cmd/run.go` as goroutine alongside existing queue listener.

---

*End of Implementation Notes*
