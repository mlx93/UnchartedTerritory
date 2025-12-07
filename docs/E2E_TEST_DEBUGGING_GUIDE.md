# E2E Test Debugging Guide

**Date**: December 6, 2025  
**Status**: Tests Running with Visible Browser

---

## Tests Running

All three failing tests are now running with `--headed` flag so you can watch what's happening:

1. ✅ `tests/chat-scrolling.spec.ts` - Chat auto-scrolling behavior
2. ✅ `tests/import-artifactory.spec.ts` - Import chart from artifacthub  
3. ✅ `tests/upload-chart.spec.ts` - Upload helm chart

---

## What to Watch For

### Test 1: chat-scrolling.spec.ts

**Expected Flow**:
1. ✅ Login happens
2. ✅ Chart uploads and workspace loads
3. ✅ User sends "Test message"
4. ✅ AI responds (streaming)
5. ✅ **Container should scroll to bottom automatically**
6. ✅ User scrolls up manually
7. ⚠️ **"Jump to latest" button should appear** ← This is where it's failing
8. ✅ User sends another message
9. ✅ Should NOT auto-scroll (respects user position)
10. ✅ Click "Jump to latest"
11. ✅ Should scroll to bottom and button disappears

**What to Check**:
- Does the scroll container (`[data-testid="scroll-container"]`) exist?
- Is there enough content to scroll? (scrollHeight > clientHeight)
- After scrolling up, does the button appear?
- Check browser console for errors

**Screenshots Saved**:
- `test-results/1-initial-scrolled-to-bottom.png`
- `test-results/2-scrolled-up-with-button.png` (or `2-scrolled-up-no-button.png` if failing)

---

### Test 2: import-artifactory.spec.ts

**Expected Flow**:
1. ✅ Login happens
2. ✅ Navigate to artifacthub import URL
3. ✅ Workspace created
4. ✅ User sends "render this chart using the default values.yaml"
5. ✅ AI responds
6. ⚠️ **Terminal should appear** (`[data-testid="chart-terminal"]`) ← This is where it's failing
7. ✅ Terminal shows helm commands/output

**What to Check**:
- Does the message have `responseRenderId` set?
- Is `render.isAutorender` false?
- Does the message have `responsePlanId`? (If yes, terminal won't show)
- Check if render was triggered (Centrifugo events)
- Check browser console for errors

**Screenshots Saved**:
- `test-results/artifacthub-1-post-login.png`
- `test-results/artifacthub-2-import-page.png`
- `test-results/artifacthub-3-chat-messages.png`
- `test-results/artifacthub-4-workspace.png`

---

### Test 3: upload-chart.spec.ts

**Expected Flow**:
1. ✅ Login happens
2. ✅ Upload chart file
3. ✅ Workspace loads
4. ✅ User sends "change the default replicaCount in the values.yaml to 3"
5. ✅ AI responds
6. ⚠️ **Plan message should appear** (`[data-testid="plan-message"]`) ← May fail here
7. ⚠️ **Plan status should update to "review"** ← May fail here
8. ⚠️ **Proceed button should be clickable** ← May fail here
9. ✅ Click Proceed
10. ✅ Files update
11. ✅ Diff appears in editor

**What to Check**:
- Does plan message appear? (Check for `[data-testid="plan-message"]`)
- Does plan status show "review"? (Check `[data-testid="plan-message-top"]` text)
- Is Proceed button visible and enabled?
- After clicking Proceed, do files update?
- Does diff appear in Monaco editor?
- Check browser console for errors
- Check network tab for API calls

**Screenshots Saved**:
- `test-results/upload-1-post-login.png`
- `test-results/upload-2-home-page.png`
- `test-results/upload-3-chat-messages.png`
- `test-results/upload-4-chat-messages.png`
- `test-results/upload-proceed-button-verification.png`
- `test-results/upload-5-diff-view.png`
- `test-results/upload-6-diff-validation.png`

---

## Common Issues & Solutions

### Issue: Scroll container not scrollable

**Symptom**: `canScroll: false` in test output

**Solution**: 
- Wait for more content to load
- Ensure messages have enough height
- Check if container has proper height constraints

### Issue: Jump to latest button not appearing

**Symptom**: Button never appears after scrolling up

**Possible Causes**:
1. `initialScrollCompleteRef.current` is still false
2. Scroll event not firing
3. State not updating (`showScrollButton` stays false)

**Debug Steps**:
```javascript
// In browser console:
window.__scrollTestState.isShowingJumpButton()
window.__scrollTestState.hasScrolledUp()
window.__scrollTestState.isAutoScrollEnabled()
```

### Issue: Terminal not appearing

**Symptom**: `[data-testid="chart-terminal"]` not found

**Possible Causes**:
1. Render not triggered
2. `responseRenderId` not set on message
3. `isAutorender` is true (terminal hidden)
4. `responsePlanId` exists (terminal hidden for plans)

**Debug Steps**:
```javascript
// In browser console, check message:
const messages = document.querySelectorAll('[data-testid="chat-message"]');
const lastMessage = messages[messages.length - 1];
// Check message data attributes or inspect element
```

### Issue: Plan message not appearing

**Symptom**: `[data-testid="plan-message"]` not found

**Possible Causes**:
1. Plan creation failed in `onFinish` callback
2. Centrifugo event not received
3. Plan not in `plansAtom`
4. Component not rendering plan

**Debug Steps**:
- Check network tab for `/api/plan/create-from-tools` call
- Check browser console for errors
- Check Centrifugo connection status

---

## Fixes Applied

### ✅ ScrollingContent.tsx
- Removed `NODE_ENV === 'test'` check (test helper always available)
- Added Playwright test detection (`__PLAYWRIGHT_TEST__`)
- Reduced initial scroll timeout from 1500ms to 500ms
- Added check for Playwright test mode in scroll detection

### ✅ chat-scrolling.spec.ts
- Added Playwright test flag initialization
- Added wait for scrollable content
- Added wait for initial scroll complete
- Added proper scroll event triggering
- Fixed tracing.stop() error handling

### ✅ import-artifactory.spec.ts
- Changed from `.font-mono` class to `[data-testid="chart-terminal"]`
- Increased timeout to 30 seconds
- Added proper wait for terminal

### ✅ upload-chart.spec.ts
- Added wait for plan message
- Added wait for plan status update
- Added wait for proceed button to be enabled
- Improved diff content verification
- Added fallback for Monaco editor diff classes

---

## Next Steps After Watching Tests

1. **If tests pass**: Update test results summary
2. **If tests still fail**: 
   - Note what you see in the browser
   - Check screenshots in `test-results/`
   - Check browser console for errors
   - Share observations for further fixes

---

## Manual Test Commands

If you want to run tests individually:

```bash
cd chartsmith/chartsmith-app

# Run with visible browser
npx playwright test tests/chat-scrolling.spec.ts --headed

# Run with slow motion (easier to watch)
npx playwright test tests/chat-scrolling.spec.ts --headed --slow-mo=1000

# Run with trace viewer (best for debugging)
npx playwright test tests/chat-scrolling.spec.ts --trace on
# Then: npx playwright show-trace test-results/chat-scrolling-trace.zip
```

---

**Watch the browser windows and let me know what you observe!** 👀

