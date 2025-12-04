# Test Automation Ideas for Chartsmith

This document outlines opportunities to automate manual UI test cases to reduce testing time and improve reliability.

---

## Current Test Infrastructure

Chartsmith already has:
- **Jest** for unit tests (`npm run test:unit`)
- **Playwright** for E2E tests (`npm run test:e2e`)
- **Go tests** (`go test ./...`)

Existing Playwright tests:
- `tests/login.spec.ts` - Login flow
- `tests/chat-scrolling.spec.ts` - Chat scroll behavior
- `tests/upload-chart.spec.ts` - Chart upload
- `tests/import-artifactory.spec.ts` - Artifact import

---

## Automation Priority Matrix

| Test Case | Effort | Value | Priority | Recommendation |
|-----------|--------|-------|----------|----------------|
| TC-1: Auth Flow | Low | High | **P1** | Already partially automated |
| TC-2: Home Page | Low | Medium | **P1** | Simple presence checks |
| TC-3: Prompt Creation | Medium | High | **P1** | Core user journey |
| TC-6: Artifact Hub | Medium | Medium | **P2** | Requires seeded data |
| TC-7: Chat | Medium | High | **P1** | Core functionality |
| TC-8: Editor | High | Medium | **P2** | Monaco editor complexity |
| TC-11: Rendered View | Medium | High | **P1** | Important for validation |
| TC-4/5: Uploads | Medium | Medium | **P2** | File handling complexity |

---

## Recommended Automated Tests

### 🔐 Auth Flow (TC-1) - Expand Existing

```typescript
// tests/auth.spec.ts
test.describe('Authentication', () => {
  test('redirects unauthenticated users to login', async ({ page }) => {
    await page.goto('/workspace/some-id');
    await expect(page).toHaveURL(/\/login/);
  });

  test('test-auth login redirects to home', async ({ page }) => {
    await page.goto('/login?test-auth=true');
    await expect(page).toHaveURL('/');
    await expect(page.locator('[data-testid="user-avatar"]')).toBeVisible();
  });

  test('authenticated user can access home', async ({ page }) => {
    await loginTestUser(page);
    await page.goto('/');
    await expect(page.locator('h1')).toContainText('ChartSmith');
  });
});
```

### 🏠 Home Page Elements (TC-2)

```typescript
// tests/home.spec.ts
test.describe('Home Page', () => {
  test.beforeEach(async ({ page }) => {
    await loginTestUser(page);
    await page.goto('/');
  });

  test('displays welcome header', async ({ page }) => {
    await expect(page.locator('h1')).toContainText('Welcome to ChartSmith');
  });

  test('displays prompt input with placeholder', async ({ page }) => {
    const input = page.locator('textarea');
    await expect(input).toHaveAttribute('placeholder', /Tell me about the application/);
  });

  test('displays all action buttons', async ({ page }) => {
    await expect(page.getByText('Upload a Helm chart')).toBeVisible();
    await expect(page.getByText('Upload Kubernetes manifests')).toBeVisible();
    await expect(page.getByText('Start from a chart in Artifact Hub')).toBeVisible();
  });

  test('displays footer links', async ({ page }) => {
    await expect(page.getByText('Terms')).toBeVisible();
    await expect(page.getByText('Privacy')).toBeVisible();
  });
});
```

### ✏️ Prompt-Based Creation (TC-3)

```typescript
// tests/workspace-creation.spec.ts
test.describe('Workspace Creation', () => {
  test.beforeEach(async ({ page }) => {
    await loginTestUser(page);
    await page.goto('/');
  });

  test('typing in prompt shows submit button', async ({ page }) => {
    const textarea = page.locator('textarea');
    await textarea.fill('Create a simple nginx chart');
    await expect(page.locator('button[type="submit"]')).toBeVisible();
  });

  test('shows character limit warning near 500 chars', async ({ page }) => {
    const textarea = page.locator('textarea');
    const longText = 'a'.repeat(501);
    await textarea.fill(longText);
    await expect(page.getByText(/works best with clear, concise prompts/)).toBeVisible();
  });

  test('submitting prompt creates workspace', async ({ page }) => {
    const textarea = page.locator('textarea');
    await textarea.fill('Simple nginx pod');
    await textarea.press('Enter');
    
    // Wait for navigation to workspace
    await expect(page).toHaveURL(/\/workspace\/[a-zA-Z0-9]+/);
  });
});
```

### 🔍 Artifact Hub Search (TC-6)

```typescript
// tests/artifact-hub.spec.ts
test.describe('Artifact Hub Search', () => {
  test.beforeEach(async ({ page }) => {
    await loginTestUser(page);
    await page.goto('/');
  });

  test('opens search modal', async ({ page }) => {
    await page.getByText('Start from a chart in Artifact Hub').click();
    await expect(page.locator('input[placeholder*="Artifact Hub"]')).toBeVisible();
  });

  test('search returns results for nginx', async ({ page }) => {
    await page.getByText('Start from a chart in Artifact Hub').click();
    const searchInput = page.locator('input[placeholder*="Artifact Hub"]');
    await searchInput.fill('nginx');
    
    // Wait for results
    await expect(page.getByText(/charts? found/)).toBeVisible({ timeout: 5000 });
  });

  test('closes modal on escape', async ({ page }) => {
    await page.getByText('Start from a chart in Artifact Hub').click();
    await page.keyboard.press('Escape');
    await expect(page.locator('input[placeholder*="Artifact Hub"]')).not.toBeVisible();
  });
});
```

### 💬 Chat Functionality (TC-7)

```typescript
// tests/chat.spec.ts
test.describe('Workspace Chat', () => {
  let workspaceUrl: string;

  test.beforeAll(async ({ browser }) => {
    // Create a workspace once for all chat tests
    const page = await browser.newPage();
    await loginTestUser(page);
    await page.goto('/');
    await page.locator('textarea').fill('Simple test chart');
    await page.locator('textarea').press('Enter');
    await page.waitForURL(/\/workspace\//);
    workspaceUrl = page.url();
    await page.close();
  });

  test('displays chat input', async ({ page }) => {
    await loginTestUser(page);
    await page.goto(workspaceUrl);
    await expect(page.locator('textarea[placeholder*="Ask a question"]')).toBeVisible();
  });

  test('sends message on enter', async ({ page }) => {
    await loginTestUser(page);
    await page.goto(workspaceUrl);
    
    const chatInput = page.locator('textarea[placeholder*="Ask a question"]');
    await chatInput.fill('What files are in this chart?');
    await chatInput.press('Enter');
    
    // Verify message appears
    await expect(page.getByText('What files are in this chart?')).toBeVisible();
  });

  test('shows role selector dropdown', async ({ page }) => {
    await loginTestUser(page);
    await page.goto(workspaceUrl);
    
    // Find and click role selector (sparkle icon)
    await page.locator('[title*="Perspective"]').click();
    await expect(page.getByText('Auto-detect')).toBeVisible();
    await expect(page.getByText('Chart Developer')).toBeVisible();
    await expect(page.getByText('End User')).toBeVisible();
  });
});
```

### 🎨 Rendered View (TC-11)

```typescript
// tests/rendered-view.spec.ts
test.describe('Rendered View', () => {
  test('switches between Source and Rendered tabs', async ({ page }) => {
    await loginTestUser(page);
    // Navigate to workspace with completed render
    await page.goto('/workspace/existing-workspace-id');
    
    // Click Rendered tab
    await page.getByText('Rendered').click();
    await expect(page.getByText('Rendered')).toHaveClass(/active|selected/);
    
    // Click Source tab
    await page.getByText('Source').click();
    await expect(page.getByText('Source')).toHaveClass(/active|selected/);
  });
});
```

---

## API/Integration Tests (Non-UI)

These can run faster without browser overhead:

### Server Actions

```typescript
// __tests__/actions/workspace.test.ts
import { createWorkspaceFromPromptAction } from '@/lib/workspace/actions/create-workspace-from-prompt';

describe('Workspace Actions', () => {
  test('createWorkspaceFromPromptAction creates workspace', async () => {
    const session = mockSession();
    const workspace = await createWorkspaceFromPromptAction(session, 'Test prompt');
    
    expect(workspace.id).toBeDefined();
    expect(workspace.charts).toHaveLength(0); // No charts initially
  });
});
```

### Artifact Hub Search

```typescript
// __tests__/artifacthub.test.ts
import { searchArtifactHubCharts } from '@/lib/artifacthub/artifacthub';

describe('Artifact Hub', () => {
  test('returns results for valid query', async () => {
    const results = await searchArtifactHubCharts('nginx');
    expect(results.length).toBeGreaterThan(0);
    expect(results[0]).toContain('artifacthub.io');
  });

  test('returns empty for short query', async () => {
    const results = await searchArtifactHubCharts('a');
    expect(results).toEqual([]);
  });

  test('handles direct URL input', async () => {
    const url = 'https://artifacthub.io/packages/helm/bitnami/nginx';
    const results = await searchArtifactHubCharts(url);
    expect(results).toEqual([url]);
  });
});
```

---

## Test Data Fixtures

Create reusable fixtures for consistent testing:

```typescript
// tests/fixtures/index.ts
export const TEST_CHARTS = {
  minimal: {
    name: 'test-chart',
    files: ['Chart.yaml', 'values.yaml', 'templates/deployment.yaml']
  },
  withDependencies: {
    name: 'app-chart',
    files: ['Chart.yaml', 'values.yaml', 'charts/redis-17.0.0.tgz']
  }
};

export const TEST_PROMPTS = {
  simple: 'Create a simple nginx deployment',
  complex: 'Create a chart with deployment, service, ingress, and HPA'
};
```

---

## CI Integration Recommendations

### GitHub Actions Workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: chartsmith
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - name: Install Helm
        run: |
          curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
      
      - name: Install dependencies
        run: |
          cd chartsmith-app
          npm ci
          npx playwright install --with-deps
      
      - name: Run E2E tests
        run: |
          cd chartsmith-app
          npm run test:e2e
```

---

## Estimated Automation ROI

| Test Suite | Manual Time | Automated Time | Time Saved/Run |
|------------|-------------|----------------|----------------|
| Auth (TC-1) | 2 min | 10 sec | 1.8 min |
| Home (TC-2) | 1 min | 5 sec | 0.9 min |
| Prompt (TC-3) | 3 min | 30 sec | 2.5 min |
| Artifact Hub (TC-6) | 2 min | 15 sec | 1.75 min |
| Chat (TC-7) | 5 min | 45 sec | 4.25 min |
| **Total** | **13 min** | **~2 min** | **11 min** |

With 10 test runs per week: **110 minutes saved weekly**

---

## Implementation Roadmap

### Phase 1 (Quick Wins) - 1-2 days
- [ ] Expand `login.spec.ts` with full auth flow
- [ ] Add `home.spec.ts` for page elements
- [ ] Add basic prompt submission test

### Phase 2 (Core Flows) - 2-3 days
- [ ] Add `artifact-hub.spec.ts`
- [ ] Add `chat.spec.ts`
- [ ] Create test fixtures

### Phase 3 (Advanced) - 3-5 days
- [ ] Add `rendered-view.spec.ts`
- [ ] Add file editor tests
- [ ] Add API/action tests
- [ ] CI integration

---

*Last Updated: December 2024*

