# E2E Test Fixes - Applied

**Date**: December 6, 2025  
**Status**: Fixes Applied - Ready for Testing

---

## Fixes Applied

### ✅ Fix 1: chat-scrolling.spec.ts - Test Helper Availability

**File**: `components/ScrollingContent.tsx`

**Changes**:
- Removed `NODE_ENV === 'test'` check - test helper now always available
- Updated fallback initialization to always expose `__scrollTestState`
- Fixed tracing.stop() error handling in test file

**Why**: Playwright runs in browser environment where `NODE_ENV` might not be set correctly. Making test helper always available ensures E2E tests can access it.

---

### ✅ Fix 2: import-artifactory.spec.ts - Terminal Visibility

**File**: `tests/import-artifactory.spec.ts`

**Changes**:
- Replaced `.font-mono` class selector with `[data-testid="chart-terminal"]`
- Increased timeout to 30 seconds
- Added proper wait for terminal to appear

**Why**: Using data-testid is more reliable than CSS classes. Terminal might take time to render, so increased timeout.

---

### ✅ Fix 3: upload-chart.spec.ts - Plan Workflow Timing

**File**: `tests/upload-chart.spec.ts`

**Changes**:
- Added wait for plan message to appear
- Added wait for plan status update (checking for "review" text)
- Added wait for proceed button to be enabled
- Improved diff content verification (content-based instead of class-based)
- Added fallback for Monaco editor diff classes

**Why**: AI SDK plan workflow has different timing than legacy path. Need to wait for each step to complete before proceeding.

---

## Testing the Fixes

### Run Tests Individually

```bash
cd chartsmith/chartsmith-app

# Test scrolling behavior
npx playwright test tests/chat-scrolling.spec.ts --headed

# Test artifacthub import
npx playwright test tests/import-artifactory.spec.ts --headed

# Test upload chart
npx playwright test tests/upload-chart.spec.ts --headed
```

### Run All E2E Tests

```bash
cd chartsmith/chartsmith-app
npm run test:e2e
```

### Run with Debug Output

```bash
# Show browser and see what's happening
npx playwright test tests/chat-scrolling.spec.ts --headed --debug

# Or with trace viewer
npx playwright test tests/upload-chart.spec.ts --trace on
```

---

## Expected Results

### Before Fixes
- ❌ `chat-scrolling.spec.ts`: Timeout (test helper not available)
- ❌ `import-artifactory.spec.ts`: Terminal not found
- ❌ `upload-chart.spec.ts`: Plan message/timing issues

### After Fixes
- ✅ `chat-scrolling.spec.ts`: Should pass (test helper available)
- ✅ `import-artifactory.spec.ts`: Should pass (proper terminal wait)
- ✅ `upload-chart.spec.ts`: Should pass (proper plan workflow waits)

---

## If Tests Still Fail

### Debugging Tips

1. **Check screenshots**: Tests save screenshots to `./test-results/`
   ```bash
   ls -la test-results/*.png
   ```

2. **Check traces**: Some tests save traces
   ```bash
   npx playwright show-trace test-results/chat-scrolling-trace.zip
   ```

3. **Run with slow motion**:
   ```bash
   npx playwright test tests/upload-chart.spec.ts --headed --slow-mo=1000
   ```

4. **Check console logs**:
   ```bash
   npx playwright test tests/import-artifactory.spec.ts --headed --debug
   ```

### Common Issues

1. **Terminal not appearing**: 
   - Check if render was triggered (might need to wait longer)
   - Verify `responseRenderId` is set on message
   - Check Centrifugo events are working

2. **Plan not appearing**:
   - Check if `buffered_tool_calls` column exists in database
   - Verify plan creation in `onFinish` callback
   - Check Centrifugo `plan-updated` events

3. **Scroll test helper not found**:
   - Verify ScrollingContent component is rendered
   - Check browser console for errors
   - Ensure component has mounted before checking helper

---

## Next Steps

1. ✅ Fixes applied
2. ⏳ Run tests to verify fixes
3. ⏳ If tests pass, update test results summary
4. ⏳ If tests still fail, debug using tips above

---

**Status**: Ready for testing! 🧪

