# Playwright Parity Plan: Main vs Branch (AI Chat Helm Flow)

## Purpose
- Validate feature parity between `main` and `branch` for the Helm chart chat experience, grounded in `docs/testing/MANUAL_AI_CHAT_TEST_CASES.md`.
- Capture behavioral diffs for streaming chat, tool calls (chart context, textEditor, version lookups), and Helm change application (plan + apply).

## Scope
- Compare two running frontends: `MAIN_BASE_URL` (main) and `BRANCH_BASE_URL` (branch).
- Same Go worker and database; both frontends talk to the same backend services.
- Focused on `/` (or `/test-ai-chat` if branch-specific) chat flow that builds/edits a Helm chart via AI tools.

## Preconditions
- Go worker running on `:8080` (`make run-worker`), DB reachable.
- Next.js dev servers:
  - main at `http://localhost:3000`
  - branch at `http://localhost:3100` (set `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` if required for branch behavior)
- Test auth available via `/login-with-test-auth`.
- Test chart tarball available at `../testdata/charts/empty-chart-0.1.0.tgz` (relative to `chartsmith-app/tests`).
- Env keys set as needed (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, `GO_BACKEND_URL=http://localhost:8080`).

## Playwright Configuration
- File: `chartsmith-app/playwright.config.ts`
- Add/ensure two projects via env:
  - `MAIN_BASE_URL` (default `http://localhost:3000`)
  - `BRANCH_BASE_URL` (default `http://localhost:3100`)
- Reporter: `html`; traces on first retry.

## How to Run
```bash
cd chartsmith/chartsmith-app
MAIN_BASE_URL=http://localhost:3000 \
BRANCH_BASE_URL=http://localhost:3100 \
npx playwright test tests/chat-helm-parity.spec.ts
```
- Ensure both servers are up before running; reuse existing servers (`reuseExistingServer: true`).
- Artifacts expected under `test-results/{main|branch}/` (screenshots, traces, JSON summaries if implemented).

## Test Cases (mapped to MANUAL_AI_CHAT_TEST_CASES)
1) Smoke + Streaming (Manual §1, Quick Smoke #1-2)
   - Send “Hello”; assert streaming (incremental chunks), message order preserved, no console 4xx/5xx.
2) Chart Context Tool (Manual §3, Quick Smoke #3)
   - Prompt “What files are in my chart?”; expect tool call success and file list including `Chart.yaml`, `values.yaml`.
3) Version Lookups (Manual §5, §6, Quick Smoke #4)
   - “What’s the latest PostgreSQL chart version?” -> semver response.
   - Optional: “What’s the current K8s minor version?” -> expect `1.32`.
4) File View + Edit (Manual §4 View/str_replace, Quick Smoke #5-6)
   - “Show me values.yaml” -> content visible.
   - “Change replicas to 5” -> plan shown, approve, diff/pending change shows `replicaCount: 5`.
5) Multi-Tool Helm Workflow (Manual §7)
   - “Add resource limits (cpu=100m, memory=128Mi) to the deployment” -> textEditor apply; verify limits in `templates/deployment.yaml`.
   - “Add postgresql as a dependency with latest version” -> latestSubchartVersion + textEditor updates `Chart.yaml`; dependency block present.
6) UI/UX Parity (Manual §9)
   - User vs AI bubbles styled, loading indicator during stream, input disabled while streaming, jump-to-latest button behavior (reuse scroll assertions).
7) Error Handling (Manual §8)
   - Force tool failure (e.g., route-block `/api/tools/context`); assert graceful error UI (no crash).

## Execution Steps Per Project (main, branch)
1. Login: `await loginTestUser(page)` (navigates to `/login-with-test-auth` then `/`).
2. Create workspace: on `/`, upload `../testdata/charts/empty-chart-0.1.0.tgz`; wait for `/workspace/{id}`.
3. Run chat prompts in order of test cases above; capture screenshots and traces.
4. Record tool POSTs (`/api/tools/**`) and responses for parity comparison (status, message text).
5. Save artifacts to `test-results/{project}/...`.

## Expected Outputs
- Parity: Both projects pass all assertions; same tool endpoints hit; no console errors.
- Known acceptable diff: Real-time updates (Centrifugo) may differ if flagged off in branch (per Feature Matrix).

## Troubleshooting
- If streaming is missing: verify `NEXT_PUBLIC_USE_AI_SDK_CHAT` on the branch and server logs.
- If tool calls fail: confirm Go worker on 8080 and `GO_BACKEND_URL` env in Next dev server.
- If upload fails: check file path and that DB schema is migrated.

## Handoff Notes for Agents
- Tests live in `chartsmith-app/tests/chat-helm-parity.spec.ts` (to be added/updated).
- Keep console open; fail fast on console errors.
- Ensure both servers are warmed up before running to avoid flakiness.

