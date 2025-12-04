# Setup Agent Prompt

**Role**: You are the Setup Agent for the Chartsmith development environment. Your job is to verify prerequisites, configure environment variables, and ensure all services are running so that PR1 implementation can begin.

---

## Prerequisites Checklist

Verify these are installed on the system:

- [ ] macOS
- [ ] Docker Desktop (running)
- [ ] Go 1.24 or later
- [ ] Node.js 18 (via nvm)
- [ ] npm
- [ ] Schemahero (binary named `schemahero` on PATH)
- [ ] Helm CLI (`brew install helm`) - **Required for chart rendering**

## Required Environment Variables

Ensure these are configured in the shell environment:

| Variable | Value/Source |
|----------|--------------|
| `ANTHROPIC_API_KEY` | User's personal key |
| `GROQ_API_KEY` | User's personal key |
| `VOYAGE_API_KEY` | User's personal key |
| `CHARTSMITH_PG_URI` | `postgresql://postgres:password@localhost:5432/chartsmith?sslmode=disable` |
| `CHARTSMITH_CENTRIFUGO_ADDRESS` | `http://localhost:8000/api` |
| `CHARTSMITH_CENTRIFUGO_API_KEY` | `api_key` |

---

## Setup Steps

### Step 1: Start Docker Environment

```bash
cd chartsmith/hack/chartsmith-dev
docker compose up -d
```

Verify containers are running:
```bash
docker ps
```

Expected: postgres and centrifugo containers running.

### Step 2: Create `.env.local` in `chartsmith-app/`

Create file at `chartsmith/chartsmith-app/.env.local`:

```env
NEXT_PUBLIC_GOOGLE_CLIENT_ID=730758876435-8v7frmnqtt7k7v65edpc6u3hso9olqbe.apps.googleusercontent.com
NEXT_PUBLIC_GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google
GOOGLE_CLIENT_SECRET=<get from 1password or leave blank for local dev>
HMAC_SECRET=not-secure
CENTRIFUGO_TOKEN_HMAC_SECRET=change.me
NEXT_PUBLIC_CENTRIFUGO_ADDRESS=ws://localhost:8000/connection/websocket
TOKEN_ENCRYPTION=H5984PaaBSbFZTMKjHiqshqRCG4dg49JAs0dDdLbvEs=
ANTHROPIC_API_KEY=<your anthropic key>
NEXT_PUBLIC_ENABLE_TEST_AUTH=true
ENABLE_TEST_AUTH=true
NEXT_PUBLIC_API_ENDPOINT=http://localhost:3000/api
```

### Step 3: Enable PGVector Extension

```bash
cd chartsmith
make pgvector
```

Or manually:
```bash
docker exec -it chartsmith-dev-postgres-1 psql -U postgres -d chartsmith -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Step 4: Deploy Database Schema

```bash
cd chartsmith
make schema
```

### Step 5: Bootstrap Chart Data (CRITICAL)

This step is **required** for the application to function:

```bash
cd chartsmith
make bootstrap
```

### Step 6: Install Frontend Dependencies

```bash
cd chartsmith/chartsmith-app
npm install
```

---

## Verification Checklist

After setup, verify:

- [ ] `docker ps` shows postgres and centrifugo containers running
- [ ] Database schema deployed without errors
- [ ] Bootstrap completed successfully
- [ ] `node_modules/` exists in `chartsmith-app/`
- [ ] `.env.local` file exists with required values

## Ready State

When setup is complete, these services can be started:

| Terminal | Command | Purpose |
|----------|---------|---------|
| 1 | `cd chartsmith/chartsmith-app && npm run dev` | Frontend (port 3000) |
| 2 | `cd chartsmith && make run-worker` | Backend worker |

**First Login**: Navigate to http://localhost:3000/login?test-auth=true — the first user to log in gets admin access.

---

## Troubleshooting

### PGVector Extension Error
If `make schema` fails with `ERROR: type "vector" does not exist`:
```bash
make pgvector
make schema
```

### Docker Not Running
Ensure Docker Desktop is started before running `docker compose up -d`.

### Schemahero Not Found
Download from https://schemahero.io/docs/installation/, rename binary to `schemahero`, and add to PATH.

### Helm Not Found / Render Fails
If chart rendering fails with "helm executable file not found":
```bash
brew install helm
# Then restart the worker
pkill -f chartsmith-worker
make run-worker
```

### Artifact Hub Search Returns No Results
The search cache may be empty. Populate it:
```bash
export CHARTSMITH_PG_URI="postgresql://postgres:password@localhost:5432/chartsmith?sslmode=disable"
./bin/chartsmith-worker artifacthub --verbose
```

---

## Output Required

When complete, report:

1. **Prerequisites**: All verified ✓/✗
2. **Setup Steps**: All completed ✓/✗
3. **Issues**: Any problems encountered and how they were resolved
4. **Ready State**: Confirmation that environment is ready for PR1 implementation

---

*This setup must be complete before spawning the PR1 Implementation Agent.*

