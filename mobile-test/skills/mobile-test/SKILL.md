---
name: mobile-test
description: Test native and cross-platform mobile apps end to end on Android and iOS. Use for APK, AAB, IPA, React Native, Expo, Flutter, Swift, Kotlin, and mobile flows involving permissions, lifecycle, interruptions, offline behavior, accessibility, screenshots, recordings, logs, and reproducible reports.
argument-hint: [app-or-feature, build-path-or-repo, and critical-flows]
---

# Mobile Test

Test a mobile app as a user would, while collecting enough technical evidence to reproduce every failure.

Do not treat "the app launched" as a passing test. Verify behavior, state, platform integration, recovery, and observable outcomes.

## Inputs

Use `$ARGUMENTS` as the test brief. Extract, when available:

- App or feature under test
- Source repository or build artifact
- Platforms requested
- Critical user journeys
- Backend/environment URL
- Test accounts or fixture requirements
- Supported OS versions and device classes
- Acceptance criteria

If essential information is missing, inspect the project before asking questions. Infer framework, package/bundle ID, build commands, supported platforms, and existing test configuration from repository files.

## Non-Negotiable Rules

- Test a real build, not a browser recreation of the mobile UI.
- Record the exact build identity: commit SHA, version, build number, package/bundle ID, and artifact checksum when available.
- Match the app's supported OS range. Include at least one representative current OS; add the minimum supported OS when compatibility is important.
- Never guess tap coordinates when an accessibility tree, semantic locator, resource ID, or visible label is available.
- Never mark a step passed based only on the absence of a crash. Check the expected UI, persisted state, network effect, and logs where relevant.
- Reset state between independent scenarios. A scenario must not pass because a previous scenario prepared hidden state.
- Capture evidence around the interaction, not minutes of setup or idle time.
- Redact passwords, tokens, personal data, notification contents, and sensitive logs from artifacts.
- Separate app defects from environment failures and test-data failures.
- Do not silently skip blocked scenarios.

## Phase 1 — Inspect and Prepare

### 1. Identify the app shape

Inspect the repository and classify it:

- Native Android: Gradle, Kotlin, Java
- Native iOS: Xcode project/workspace, Swift, Objective-C
- React Native or Expo
- Flutter
- Kotlin Multiplatform
- Prebuilt APK/IPA only

Determine:

- Android application ID and launch activity
- iOS bundle ID and scheme
- Build variants/configurations
- API base URL and environment injection method
- Authentication and seed-data needs
- Deep links or universal/app links
- Existing unit, integration, and device tests
- Minimum and target OS versions

### 2. Select platforms and devices

Choose platforms after inspecting the app:

1. Test both Android and iOS when the request is cross-platform or parity matters.
2. Use Android first for fast feedback unless the user requests iOS first.
3. Test only a supported platform when the app or request is platform-specific.
4. If one requested platform is unavailable, mark that platform row `BLOCKED`, continue on the available platform, and use `PARTIAL` overall if the completed testing is otherwise useful.

Use real devices when hardware, push notifications, biometrics, camera quality, Bluetooth, NFC, background execution, or store-distributed builds are central. Otherwise, an emulator or simulator is acceptable.

Verify prerequisites before promising coverage:

```bash
adb devices -l
emulator -list-avds
appium --version
appium driver list --installed   # require uiautomator2 and/or xcuitest for requested platforms
# macOS with Xcode only
xcrun simctl list devices available
```

Assume Appium and its platform drivers are provisioned by the test environment. If the required driver is absent and the project has no established alternative such as Maestro or Detox, mark locator-based interaction coverage `BLOCKED`; do not fall back to guessed coordinates.

### 3. Define a compact test charter

Before execution, write a prioritized scenario list with expected results. Always include `P0` launch and the critical happy path. Select applicable scenarios from authentication, validation, server errors, lifecycle, permissions, networking, layout, and accessibility; mark important inapplicable categories `N/A` rather than inventing irrelevant tests.

### 4. Prepare deterministic state

- Use dedicated test accounts and non-production data.
- Seed fixtures through an API or script when possible.
- Document any prerequisite that cannot be automated.
- Clear app data, reinstall, or use a fresh simulator snapshot for first-run tests.
- Preserve a warm authenticated state only for scenarios explicitly designed to reuse it.
- Set known locale, timezone, clock mode, orientation, appearance, and network condition.

## Phase 2 — Build and Install

### Android

Build the requested variant and retain the APK or AAB path. Prefer an APK for direct emulator/device testing.

Typical APK flow:

```bash
./gradlew assembleDebug
adb -s <serial> install -r <app.apk>
adb -s <serial> shell pm clear <application-id>   # only for a clean-state scenario
adb -s <serial> shell monkey -p <application-id> -c android.intent.category.LAUNCHER 1
adb -s <serial> shell pidof <application-id>
adb -s <serial> shell dumpsys window | grep -E 'mCurrentFocus|mFocusedApp'
```

An AAB cannot be passed to `adb install`. Convert and install it with Bundletool:

```bash
bundletool build-apks --bundle=<app.aab> --output=<app.apks> --mode=universal
bundletool install-apks --apks=<app.apks>
```

Always target a specific device with `-s <serial>` when more than one device may exist.

### iOS

Build a simulator `.app` for interactive testing or a signed `.ipa` for a physical device. On non-macOS hosts, route Xcode work to an available macOS builder rather than attempting to install Xcode locally.

Typical simulator flow on macOS:

```bash
xcrun simctl boot <device-udid>
open -a Simulator
xcodebuild -workspace <App.xcworkspace> -scheme <Scheme> \
  -configuration Debug -sdk iphonesimulator \
  -destination 'platform=iOS Simulator,id=<device-udid>' \
  -derivedDataPath ./build build
xcrun simctl install booted ./build/Build/Products/Debug-iphonesimulator/<App>.app
xcrun simctl launch booted <bundle-id>
```

For a signed IPA on a connected physical device, use `xcrun devicectl device install app --device <device-id> <app.ipa>` or the project's established deployment tool. Verify the expected first screen through the test driver. For cloud simulators, record the build job and uploaded artifact identity.

### Backend connectivity

A device or cloud simulator usually cannot reach a development backend through the host's `localhost`.

- Use an emulator-safe host address, LAN address, or HTTPS tunnel as appropriate.
- Inject the reachable URL before building when configuration is compile-time.
- Verify a health endpoint from the same network context when possible.
- Never point destructive tests at production unless the user explicitly authorizes it.

## Phase 3 — Execute Scenarios

For every scenario:

1. State the precondition.
2. Start from a known app state.
3. Capture the initial screen when useful.
4. Perform one user-meaningful action at a time.
5. Wait on an observable condition, not an arbitrary long sleep.
6. Assert the visible result and relevant side effect.
7. Inspect logs when behavior is ambiguous or failed.
8. Capture final evidence.
9. Record `PASS`, `FAIL`, or `BLOCKED` immediately.

### Interaction strategy

Prefer Appium with the UiAutomator2 driver on Android and XCUITest driver on iOS so both platforms use accessibility-based selectors and explicit assertions. If the project already has Maestro, Detox, or native UI tests, reuse that established driver instead of introducing another framework.

Use selectors in this order:

1. Accessibility identifier or resource ID
2. Visible label/text
3. Content description or semantic role
4. UI hierarchy bounds
5. Coordinates only as a last resort

Refresh the UI hierarchy after navigation, dialogs, keyboard changes, rotation, or scrolling. Element positions and references can become stale.

For lightweight Android inspection and input:

```bash
adb -s <serial> exec-out uiautomator dump /dev/tty
adb -s <serial> shell input tap <x> <y>
adb -s <serial> shell input swipe <x1> <y1> <x2> <y2> <duration-ms>
adb -s <serial> shell input text '<text-without-spaces>'
adb -s <serial> shell input keyevent KEYCODE_SPACE
adb -s <serial> shell input keyevent KEYCODE_BACK
```

Re-dump after dismissing the keyboard. On continuously animating screens, Appium/Maestro is more reliable than waiting for `uiautomator dump` to become idle.

### Functional coverage

Test only relevant items, but consider:

- First launch, onboarding, login, logout, and session expiry
- Form validation, keyboard type, focus order, submit behavior
- Loading, empty, partial, success, and error states
- Navigation, back behavior, gestures, and deep links
- Create/update/delete effects and persistence after relaunch
- Pagination, pull-to-refresh, retry, and duplicate submission prevention
- File upload, camera/gallery, sharing, and downloads
- Push-notification entry points and in-app notification handling
- Cross-platform parity in copy, state, and business rules

### Permissions

For each relevant permission, test fresh-state branches separately:

- Allow
- Deny
- Deny permanently / "Don't ask again," where supported
- Grant or revoke from system settings, then return to the app

Use platform controls when appropriate:

```bash
adb -s <serial> shell pm grant <package> <permission>
adb -s <serial> shell pm revoke <package> <permission>
xcrun simctl privacy booted grant <service> <bundle-id>
xcrun simctl privacy booted revoke <service> <bundle-id>
xcrun simctl privacy booted reset <service> <bundle-id>
```

Use grant/revoke commands only to arrange preconditions; they bypass the real prompt, and revoke may kill the app. Exercise deny and permanently-denied behavior through the actual system dialog from a reset state (`pm clear` or `simctl privacy reset`). Verify that the app explains why access is needed, degrades safely, and recovers after permission changes. Do not assume permission state resets on reinstall; verify it.

### Lifecycle and interruptions

Exercise relevant transitions:

- Background then foreground
- Terminate then cold launch
- Screen lock/unlock
- Rotation or window resize
- Incoming notification or system dialog
- Keyboard open/close
- Process recreation or memory pressure when feasible

Check that drafts, navigation, authentication, timers, media, and pending requests recover correctly without duplicate actions.

### Network conditions

For networked features, test:

- Offline before opening the screen
- Connection loss during a request
- Slow response
- Server error or malformed response when controllable
- Recovery after connectivity returns

Android device controls include:

```bash
adb -s <serial> shell svc wifi disable
adb -s <serial> shell svc data disable
# restore after the scenario
adb -s <serial> shell svc wifi enable
adb -s <serial> shell svc data enable
```

iOS Simulator shares the host network and cannot independently toggle connectivity. Use a controlled proxy, host-level network shaping, Network Link Conditioner on a physical device, or mark the scenario blocked with the exact limitation.

Verify bounded loading indicators, actionable messages, safe retries, no duplicate writes, and consistent local state.

### Visual and accessibility checks

At minimum, inspect:

- Small and representative device sizes
- Text clipping, overlap, safe areas, keyboard avoidance, and touch target size
- Light and dark appearance when supported
- Increased font size / Dynamic Type
- Screen reader labels for actionable controls
- Focus order and meaningful error announcements
- Color-independent communication of state

Do not claim full accessibility compliance from a brief manual pass. State exactly what was checked.

## Phase 4 — Observe Failures

### Android evidence

Capture focused logs around each failure:

```bash
adb -s <serial> logcat -b all -c
APP_PID=$(adb -s <serial> shell pidof -s <application-id> | tr -d '\r')
# reproduce; APP_PID was captured before a possible crash
adb -s <serial> logcat -d --pid="$APP_PID" -v threadtime > android-logcat.txt
adb -s <serial> logcat -b crash -d -v threadtime > android-crash.txt
adb -s <serial> exec-out screencap -p > failure.png
adb -s <serial> exec-out uiautomator dump /dev/tty > window.xml
```

If no PID exists before reproduction, capture a bounded time window and filter it by package/process name instead of passing an empty `--pid`. Look for crashes, ANRs, network exceptions, permission failures, and the app package's logs. Attach only the focused excerpt that proves the issue.

Record Android video with a bounded duration, then pull and verify it:

```bash
adb -s <serial> shell screenrecord --time-limit 180 /sdcard/run.mp4
# Run screenrecord in the background; send SIGINT to its local adb process to finish early.
adb -s <serial> pull /sdcard/run.mp4 recordings/
ffprobe recordings/run.mp4
```

### iOS evidence

```bash
xcrun simctl io booted screenshot failure.png
xcrun simctl io booted recordVideo --force failure.mp4 & VIDEO_PID=$!
# reproduce, then finalize the file:
kill -INT "$VIDEO_PID" && wait "$VIDEO_PID"
ffprobe failure.mp4
xcrun simctl spawn booted log show --last 5m \
  --predicate 'process == "<process-name>"' > ios.log
xcrun simctl openurl booted '<deep-link>'
xcrun simctl push booted <bundle-id> payload.apns
```

Start `recordVideo` immediately before reproduction, finalize it with `SIGINT`, and verify the output. Include the relevant crash report from `~/Library/Logs/DiagnosticReports` when the process terminates. Preserve only the focused excerpt and identify the process, timestamp, and scenario.

### Bug quality bar

A failure entry must include:

- Short title
- Severity: blocker, critical, major, minor, or cosmetic
- Platform, OS, and device
- Build identity
- Preconditions
- Minimal numbered reproduction steps
- Expected result
- Actual result
- Reproduction rate, such as `3/3`
- Screenshot or recording path
- Relevant log excerpt or crash report
- Whether it reproduces after clean reinstall/reset

Retest a suspected defect once from a clean state. Do not hide flaky behavior; report the observed reproduction rate.

## Phase 5 — Evidence and Report

Save output under:

```text
.claude/reports/mobile-test/<app-or-feature-slug>/
├── REPORT.md
├── screenshots/
├── recordings/
├── logs/
└── build-info.txt
```

Ensure `.claude/reports/` is ignored by the product repository before writing artifacts so `/commit-all` cannot stage logs or recordings. Capture screenshots for key assertions, recordings only for the critical happy path and important defects, and logs only when diagnosing failures. Confirm every artifact opens and corresponds to the reported build.

Use this report structure:

```markdown
# Mobile Test Report: <App or Feature>

**Date:** <ISO-8601 timestamp>
**Overall:** PASS | FAIL | PARTIAL | BLOCKED
**Build:** <commit, version, build number, checksum>
**Environment:** <backend/test environment>

## Coverage
| Platform | OS | Device | Scenarios | Result |
|---|---|---|---:|---|

## Results
| ID | Priority | Scenario | Expected | Result | Evidence |
|---|---|---|---|---|---|

## Defects
### <Severity>: <Title>
- Environment:
- Preconditions:
- Steps:
- Expected:
- Actual:
- Reproduction rate:
- Evidence:
- Logs:

## Blocked or Skipped Coverage
- <item and exact reason>

## Residual Risk
- <important behavior not proven by this run>
```

### Status rules

Apply status precedence in this order:

- `FAIL`: at least one tested acceptance criterion failed, even if other coverage is blocked.
- `BLOCKED`: no meaningful validation could be completed because of environment, build, credentials, or data.
- `PARTIAL`: useful testing passed, but meaningful planned coverage remains blocked or untested.
- `PASS`: all planned critical scenarios passed and no critical coverage is blocked.

A build failure is not an app test pass. A platform not tested is not implied coverage.

## Completion Checklist

Before finishing, confirm:

- [ ] Build identity and environment recorded
- [ ] Platform/device/OS matrix recorded
- [ ] Critical happy path tested end to end
- [ ] Relevant error, permission, lifecycle, and network paths tested
- [ ] State reset between independent scenarios
- [ ] Expected outcomes asserted, not merely observed
- [ ] Failures reproduced and supported by logs/evidence
- [ ] Sensitive information removed from artifacts
- [ ] Blocked and skipped coverage listed explicitly
- [ ] Report and artifact paths provided to the user

## Example Invocations

```text
/mobile-test Test login and offline sync in the React Native app. Build from this repo; test Android and iOS. Critical flow: sign in, create a note offline, reconnect, and verify one synced note.
```

```text
/mobile-test Test the Android APK at ./builds/app-release.apk. Verify onboarding, notification permission deny/allow, background recovery, and logout.
```

```text
/mobile-test Test the iOS checkout flow on a small and large simulator using the staging backend. Include invalid card, interrupted payment, and app relaunch after success.
```
