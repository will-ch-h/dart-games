# Flutter Drive Phantom Failure Investigation

**Date:** 2026-04-16/17
**Status:** In Progress
**Branch:** server-side-updates

## Problem Statement

When running integration tests via `flutter drive`, the driver process reports test failures (exit code 1) even though the test body passes completely — all assertions succeed, all debug output confirms correct values, and the `IntegrationTestWidgetsFlutterBinding.results` map shows `success`.

This affects multiple test suites (target_tag_menu_and_mechanics, target_tag_add_player, target_tag_results_screen, carnival_derby_ui) and was initially misdiagnosed as a `loadPlayers` race condition.

## Reproduction

```bash
# Start chromedriver
cd chromedriver/chromedriver-win64 && ./chromedriver.exe --port=4444

# Run isolated debug test
flutter drive --driver=test_driver/integration_test.dart \
  --target=integration_test/target_tag/debug_test6_test.dart -d chrome
```

The debug test file is at `integration_test/target_tag/debug_test6_test.dart`.

## Key Findings

### 1. Test Logic Passes, Driver Reports Failure

The debug test isolates Test 6 from `target_tag_menu_and_mechanics_test.dart`. Every run shows:

- All assertions pass (opponentTarget has a value, shields match, etc.)
- `binding.results` shows `{Test 6: success}` at tearDown time
- `binding.failureMethodsDetails` is `[]` (empty)
- `capturedFlutterErrors count: 0` — zero Flutter errors caught
- Test counter shows `+2: All tests passed!` (from test framework in browser)

Yet the flutter drive process exits with code 1 and prints:
```
Failure Details:
Failure in method: Test 6 with cleanup monitoring
Expected: not null
  Actual: <null>
Opponent target should not be null
```

### 2. Failure is NOT in binding.results

We instrumented `binding.results` at every checkpoint:
- Before test: `{}`
- After each step: `{}`
- Final state in test body: `{}`
- In tearDown callback: `{Test 6: success}`
- `failureMethodsDetails: []`

The failure reported by the driver does NOT exist in the binding's results map.

### 3. FlutterError.onError Interceptor Catches Nothing

We installed a custom `FlutterError.onError` handler that logs every error. It was deliberately NOT restored after the test body (to stay active during post-test cleanup). Result: zero errors caught at any point.

### 4. Intermittent on Second Run

- **Run 1** (stale chromedriver): Failed
- **Run 2** (same chromedriver session): Passed (exit code 0)
- **Run 3** (fresh chromedriver restart): Failed
- **Run 4** (same session): Failed

This suggests the issue is related to initial chromedriver/browser state but is not strictly tied to it.

### 5. Stack Trace Points to Test Code

The reported failure stack trace references our test file and shared helpers:
```
debug_test6_test.dart 139:7     <fn>
shared/ui_test_helpers.dart 97:3   <fn>
shared/pump_sequences.dart 28:3   <fn>
```

These are async continuation frames, not a direct call stack. The actual `expect` at that line passed during execution.

## Architecture of the Failure Reporting

### How Results Flow from Browser to Driver

1. **Test runs in browser** within a forked error zone (`_runTest` in `binding.dart:1480`)
2. **Results stored** in `IntegrationTestWidgetsFlutterBinding.results` map (key=test description, value=`success` or `Failure`)
3. **`reportTestException` handler** (set in binding constructor, `integration_test.dart:88-92`) writes `Failure` to `results` when called
4. **`runTest` completion** (`integration_test.dart:238`): `results[description] ??= _success` — won't overwrite existing Failure
5. **`tearDownAll`** (`integration_test.dart:51`): completes `_allTestsPassed` based on `failureMethodsDetails.isEmpty`
6. **Driver calls `requestData(null)`** (`integration_test_driver.dart:71`)
7. **Web callback** (`_callback_web.dart:137-148`): awaits `allTestsPassed.future`, builds `Response`
8. **Driver deserializes** Response, checks `allTestsPassed`, prints result, exits

### Key Source Files (Flutter SDK)

- `packages/integration_test/lib/integration_test.dart` — `IntegrationTestWidgetsFlutterBinding`, `reportTestException` handler, `runTest`, `tearDownAll`
- `packages/integration_test/lib/integration_test_driver.dart` — `integrationDriver()`, driver-side result handling
- `packages/integration_test/lib/common.dart` — `Response`, `Failure` classes, JSON serialization
- `packages/integration_test/lib/src/_callback_web.dart` — `_WebCallbackManager`, `_requestData`, web service extension
- `packages/integration_test/lib/src/_extension_web.dart` — `registerWebServiceExtension`, `window.$flutterDriver` JS interop
- `packages/flutter_test/lib/src/binding.dart` — `_runTest` (line 1480), `_runTestBody` (line 1665), error zone setup, `handleUncaughtError`, post-test cleanup

### Post-Test Cleanup Sequence (binding.dart:1682-1708)

After `await testBody()` returns, the framework runs:
```dart
asyncBarrier();                    // drain microtasks
runApp(Container(key: UniqueKey())); // unmount entire widget tree
await pump();                       // process frame — pending async could complete here
invariantTester();                  // check invariants
_verifyReportTestExceptionUnset();  // verify reportTestException wasn't changed
_verifyInvariants();               // check debug variables
```

Any async exception during this cleanup is caught by the zone's `handleUncaughtError` and reported via `reportTestException` — but our interceptor shows zero errors during this phase.

### Error Zone Behavior (binding.dart:1544-1645)

```dart
void handleUncaughtError(Object exception, StackTrace stack) {
  if (testCompleter.isCompleted) {
    // Error AFTER test completed — reports with "but after the test had completed" context
    reportTestException(...);
    return;
  }
  // Error DURING test — reports with "running a test" context
  FlutterError.reportError(...);  // Sets _pendingExceptionDetails
  _parentZone!.run<void>(testCompletionHandler);
}
```

The reported error says "running a test" (not "after test completed"), suggesting it's caught during the test execution — but our monitoring shows no errors during execution.

## Web Communication Layer (Prime Suspect)

The failure appears to be injected between the app-side results (which show success) and the driver-side reception (which shows failure). The web communication layer is the prime suspect:

### `_extension_web.dart` — JS Interop

```dart
void registerWebServiceExtension(callback) {
  _window.setProperty(r'$flutterDriverResult'.toJS, null);
  _window.setProperty(r'$flutterDriver'.toJS, (JSAny message) {
    (() async {
      try {
        final result = await callback(params);
        _window.setProperty(r'$flutterDriverResult'.toJS, json.encode(result).toJS);
      } catch (error, stackTrace) {
        _window.setProperty(r'$flutterDriverResult'.toJS,
          json.encode({'isError': true, 'response': '$error\n$stackTrace'}).toJS);
      }
    })();
  }.toJS);
}
```

This sets `window.$flutterDriver` as a JS function. The driver calls it and polls `window.$flutterDriverResult` for the response.

### Potential Issues in the Web Layer

1. **Race condition in JS polling**: The driver polls `$flutterDriverResult` for a non-null value. If a previous invocation's result is still present, the driver could read stale data.
2. **Double invocation**: If `$flutterDriver` is called twice, the second call overwrites the first result.
3. **Serialization boundary**: The response passes through `json.encode` → JS string → driver-side `json.decode`. Any corruption here would cause misinterpretation.
4. **`registerServiceExtension` vs `registerWebServiceExtension`**: Both are registered on web (`integration_test.dart:219-223`). The VM service extension (`registerServiceExtension(name: 'driver', callback: callback)`) might also be active on web, creating a duplicate path.

## Hypotheses (Ranked by Likelihood)

### H1: Stale `$flutterDriverResult` from Prior Interaction (High)
The `$flutterDriverResult` is initialized to `null` once. If the driver sends a health check or handshake before `requestData(null)`, the result from that earlier call could persist. The driver's polling loop might read a stale result.

### H2: Dual Service Extension Conflict (Medium)
On web, both `registerWebServiceExtension` (JS interop) and `registerServiceExtension` (VM service) are registered with the same callback. The driver might use one path while results are returned through the other.

### H3: Timing Issue in Async Callback (Medium)
The `_requestData` method awaits `testRunner.allTestsPassed.future`. If the driver's polling times out or reads a partial result before this future completes, it might get an unexpected value.

### H4: Prior Test Run Contamination (Low-Medium)
Even with chromedriver restart, the compiled JS bundle might cache results in a way that bleeds between runs.

## Next Steps

1. **Investigate the web driver polling mechanism** — Look at `flutter_driver`'s web driver implementation to see how it polls `$flutterDriverResult` and whether it could read stale data
2. **Add logging to `_requestData`** — Print the actual boolean value of `allTestsPassed` and the serialized response JSON to see what's being sent back
3. **Check for dual path conflict** — Determine if the VM service extension path is being used alongside the JS interop path on web
4. **Test with `flutter test` instead of `flutter drive`** — `flutter test integration_test/...` uses a different communication mechanism and may not have this issue
5. **Test with `--no-dds` flag** — Disable Dart Development Service to eliminate potential proxy issues

## Workaround Candidates

- **Use `flutter test` instead of `flutter drive`** for web integration tests (different driver mechanism)
- **Add explicit delay before driver requests results** to ensure all async operations settle
- **Modify `run_ui_tests.bat`** to use `flutter test -d chrome` instead of `flutter drive`

## Debug Test File

The debug test is at `integration_test/target_tag/debug_test6_test.dart`. It:
- Monitors `binding.results` at every step
- Installs a persistent `FlutterError.onError` interceptor (not restored, stays active during cleanup)
- Adds 5 seconds of extra pumps to flush pending async before test body ends
- Has a `tearDown` callback that prints binding state
- Runs the core Test 6 logic (add 2 players, start game, check opponent target, throw dart)

## Test Output Examples

### Full output from run with persistent interceptor (exit code 1):
```
binding.results BEFORE test: {}
... [test execution, all assertions pass] ...
binding.results: {} (at every checkpoint)
========== TEST BODY COMPLETE (errors: 0) ==========

========== tearDown CALLBACK ==========
  binding.results: {Test 6 with cleanup monitoring: success}
  binding.failureMethodsDetails: []
  capturedErrors count: 0

00:30 +2: All tests passed!
Failure Details:
Failure in method: Test 6 with cleanup monitoring
Expected: not null
  Actual: <null>
Opponent target should not be null
```

Note: "All tests passed!" (exclamation) is from the test framework in the browser.
The driver's "All tests passed." (period) was NOT printed — instead "Failure Details:" was printed.
