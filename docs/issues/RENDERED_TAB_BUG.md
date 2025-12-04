# Rendered Tab Display Bug

**Status:** Documented for future fix (post-PR2)  
**Priority:** Low (workaround available via PR2 ValidationResults)  
**Discovered:** December 2024

---

## Problem Summary

The "Rendered" tab in the workspace view fails to display rendered Kubernetes manifests even when `helm template` completes successfully. The backend correctly generates rendered files, but the frontend doesn't display them.

---

## Symptoms

1. User creates/modifies a chart
2. Render job completes successfully (visible in Terminal widget)
3. Click "Rendered" tab
4. Select a file (e.g., `service.yaml`)
5. **Expected:** See rendered Kubernetes YAML
6. **Actual:** Shows "This file was not included in the rendered output"

---

## Evidence That Backend Works

Terminal widget shows successful render:
```
% /usr/local/bin/helm dependency update .
Found Chart.yaml at Chart.yaml
...
% /usr/local/bin/helm template chartsmith . --include-crds --values /dev/stdin
✓ templates/serviceaccount.yaml
✓ templates/service.yaml
```

Database confirms rendered files exist:
```sql
SELECT file_path, length(content) FROM workspace_rendered_file 
WHERE workspace_id = 'DG2A46F6JvQP';

           file_path           | content_length 
-------------------------------+----------------
 templates/serviceaccount.yaml |            313
 templates/service.yaml        |            472
```

Render job completed successfully:
```sql
SELECT is_success, helm_template_stderr FROM workspace_rendered_chart;
 is_success | helm_template_stderr 
------------+----------------------
 t          |                      
```

---

## Root Cause Analysis

### Suspected Issues

1. **WebSocket State Sync**
   - `useCentrifugo` hook may not receive render completion event
   - `rendersAtom` doesn't get updated with new render data
   
2. **Revision Mismatch**
   - Frontend may be looking at wrong revision number
   - Rendered files stored for revision X, UI querying revision Y

3. **Atom Hydration Timing**
   - `renderedFilesAtom` derived from `rendersAtom`
   - Race condition between render completion and atom update

### Key Files to Investigate

```
chartsmith-app/
├── atoms/workspace.ts
│   ├── rendersAtom
│   ├── renderedFilesAtom
│   └── renderedFilesByRevisionAtom
├── hooks/useCentrifugo.ts
│   └── Handles WebSocket events for renders
├── components/WorkspaceContent.tsx
│   └── Passes initialRenders to atoms
└── lib/workspace/actions/list-workspace-renders.ts
    └── Fetches render data from database
```

### Relevant Atoms (workspace.ts)

```typescript
// Atom for getting rendered files for a specific revision
export const renderedFilesByRevisionAtom = atom(get => {
  const renders = get(rendersAtom);
  return (revisionNumber: number) => {
    const revisionRenders = renders.filter(r => r.revisionNumber === revisionNumber);
    return revisionRenders.flatMap(render =>
      render.charts.flatMap(chart =>
        chart.renderedFiles || []
      )
    );
  };
});
```

---

## Reproduction Steps

1. Start fresh: `docker compose down && docker compose up -d`
2. Run schema and bootstrap
3. Login at `http://localhost:3000/login?test-auth=true`
4. Create workspace: "Simple nginx chart"
5. Wait for AI to create files
6. Accept all changes (triggers render)
7. Click "Rendered" tab
8. Click any file → Shows "not included" despite successful render

---

## Workarounds

### Current Workaround
Refresh page multiple times until state syncs (unreliable)

### Post-PR2 Workaround
Use `validateChart` tool - results display inline in chat:
```
User: "Validate my chart"
→ Shows helm lint, helm template, kube-score output in chat
→ No reliance on Rendered tab!
```

---

## Potential Fixes

### Option 1: Fix WebSocket Handler
Ensure `useCentrifugo` properly updates `rendersAtom` when render completes:
```typescript
// In useCentrifugo.ts
case 'render_complete':
  setRenders(prev => [...prev, event.render]);
  break;
```

### Option 2: Polling Fallback
Add polling mechanism if WebSocket fails:
```typescript
useEffect(() => {
  const interval = setInterval(async () => {
    const renders = await listWorkspaceRendersAction(session, workspace.id);
    setRenders(renders);
  }, 5000);
  return () => clearInterval(interval);
}, [workspace.id]);
```

### Option 3: Server-Side Rendering
Load rendered files via server action on tab switch instead of relying on WebSocket:
```typescript
const handleTabChange = async (tab: 'source' | 'rendered') => {
  if (tab === 'rendered') {
    const freshRenders = await listWorkspaceRendersAction(session, workspace.id);
    setRenders(freshRenders);
  }
};
```

### Option 4: Direct Database Query
Bypass renders atom, query rendered files directly when file selected:
```typescript
const loadRenderedContent = async (filePath: string) => {
  const content = await getRenderedFileAction(session, workspace.id, revision, filePath);
  return content;
};
```

---

## Testing the Fix

1. Create new workspace
2. Accept changes → Render triggered
3. **Immediately** click Rendered tab
4. Click `service.yaml` → Should show YAML content
5. Refresh page → Content should persist
6. Create new changes → Accept → New render should display

---

## Related Code References

- `chartsmith-app/components/CodeEditor.tsx` - Displays rendered content
- `chartsmith-app/atoms/workspace.ts` - State management
- `chartsmith-app/hooks/useCentrifugo.ts` - WebSocket handling
- `pkg/listener/render-workspace.go` - Backend render logic
- `pkg/realtime/types/render-stream.go` - WebSocket event types

---

## Priority Justification

**Low priority because:**
- PR2's ValidationResults provides alternative way to see rendered output
- Core functionality (chart creation, editing) still works
- Bug is intermittent/timing-related, not a complete failure

**Consider fixing if:**
- Users specifically need Rendered tab functionality
- Time permits after PR1, PR1.5, PR2 complete
- Want to polish the overall UX

---

*Document created: December 2024*
*Last updated: December 2024*

