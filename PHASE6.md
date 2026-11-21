# Phase 6 — Accessibility automation + macro recorder ✅

Status: **Complete** (2026-11-15). App version `0.6.0-phase6`.

Phase 6 adds screen automation: a real `AccessibilityService` that can
read the current screen and perform taps, typing, scrolling and
navigation — always through the Phase 2 Confirmation Engine, never on
its own — plus a macro recorder that turns the user's own taps into
replayable macros stored in the encrypted Room database.

**Everything in this phase still goes through the gate.** The service
never acts on its own, never reads or stores password fields, never
acts in banking/payment apps or on screens containing password fields,
and treats screen content as untrusted data (Phase 3 prompt-injection
rule).

## What was built

### `:core` — pure automation logic (`core/automation/`)

| File | What it does |
| --- | --- |
| `ScreenSnapshot.kt` | `ScreenNode` (text, contentDescription, viewId, className, clickable, editable, `isPassword` — always `false` in real snapshots, bounds), `ScreenSnapshot` (packageName, nodes, `hasPasswordField`, `interactiveNodes()`, `find()` preferring interactive matches), `NodeSelector` (@Serializable: text, contentDescription, viewId, className, x, y; AND matching — text/desc case-insensitive substring, viewId exact, className simple name, coordinates via bounds; an empty selector matches nothing; password nodes never match) |
| `SnapshotSanitizer.kt` | `format()` — numbered list of interactive nodes for the LLM (max 30, labels truncated to 60 chars, password nodes dropped defensively, empty-screen message, password-field note); `describeNode()` — `"[Button] “Search” (tappable)"` |
| `BlockedApps.kt` | Best-effort banking/payment/wallet blocklist: **segment-based** matching (never raw substring, so `com.example.display` is NOT blocked for containing "pay"), `BLOCKED_SEGMENTS` (bank, pay, wallet, paypal, venmo, chase, hsbc, coinbase, …) + `BLOCKED_PACKAGE_PREFIXES` (PayPal, Venmo, Cash App, Google Wallet, Samsung Pay, Revolut, Wise, Monzo, Starling, Chase). `isBlocked()` / `blockReason()` |
| `Macro.kt` | `MacroStep` (@Serializable: action, selector, text, waitMillis), `MacroAction` (TAP, TAP_AT, TYPE_TEXT, SCROLL_FORWARD, SCROLL_BACK, BACK, HOME, WAIT), `Macro` (@Serializable: id, name, steps, createdAtMillis, lastRunAtMillis; `isExternal` = any TAP/TAP_AT/TYPE_TEXT step), `MacroLimits` (max 50 steps, 10 s per-step timeout, max 30 s wait, max 60-char name, max 500-char text), `MacroValidation` (per-step validation with step numbers in the message) |
| `RecorderLogic.kt` | `ObservedUiEvent` / `ObservedEventKind` (CLICKED, TEXT_CHANGED, SCROLLED_FORWARD, SCROLLED_BACK); `eventToStep()` skips own-package events, password text/clicks, clicks without a selector, blank text; `appendStep()` coalesces consecutive TYPE_TEXT steps for the same field (keystroke events become one step) |
| `JarvisSettings.kt` | `automationEnabled: Boolean = false` — off by default; the toggle is greyed until the Accessibility service is enabled |

### `:tools` — automation tools (`tools/automation/`)

| File | What it does |
| --- | --- |
| `AutomationDevice.kt` | `AutomationDevice` interface (`checkAvailable()`, `snapshot()`, `tap()`, `tapAt()`, `typeText()`, `scroll()`, `pressBack()`, `pressHome()`, all returning `JarvisResult`); `MacroStore` interface (`getByName`, `markRun`); `NoOpAutomationDevice` + `NoMacroStore` (so `:tools` stays pure Kotlin and tests don't need Android) |
| `MacroPlayer.kt` | `MacroPlayer(device, killSwitch, stepTimeoutMillis = 10s, sleeper)` with `PlaybackEvent` (StepStarted/StepFinished). Validates the macro, checks availability, checks the kill-switch generation before every step, per-step `withTimeout`, verifies TAP/TYPE_TEXT targets exist via a snapshot before acting (with a password-field hint when the target is missing on a password screen), stops at the first failure reporting step number + screen summary (top 10 nodes), WAIT uses an injected sleeper |
| `AutomationTools.kt` | `FindOnScreenTool`, `TapOnScreenTool`, `TypeTextTool`, `ScrollScreenTool`, `PressBackTool`, `PressHomeTool` (see tool table below). All call `checkAvailable()` first |
| `RunMacroTool.kt` | `run_macro`: `riskTier` = EXTERNAL (fail-safe default); `tierFor(params)` returns EXTERNAL if the macro is external (tap/type) or not found, REVERSIBLE if navigation-only; `dryRun` lists every numbered step; `execute` plays via `MacroPlayer` and calls `markRun` only on success |
| `ToolStub.kt` | Added `suspend fun tierFor(params: JsonObject): RiskTier = riskTier` hook so a tool's tier can depend on its parameters |
| `GatedToolExecutor.kt` | `execute()` now uses `tool.tierFor(params)` (with failure handling) instead of `tool.riskTier` directly; `executeUndo` still uses the static tier |
| `ToolsModule.kt` | `registerAll(registry, device, automationDevice = NoOpAutomationDevice, macroStore = NoMacroStore, killSwitch = KillSwitch())` — defaults keep existing callers compiling; registers all 7 automation tools and constructs the `MacroPlayer`; `provideToolRegistry` now takes `PhoneDevice, AutomationDevice, MacroStore, KillSwitch` |

### `:data` — macro storage

| File | What it does |
| --- | --- |
| `db/MacroEntity.kt` | `MacroEntity` (table `macros`: id, name, stepsJson, createdAtMillis, lastRunAtMillis) + `MacroDao` (observeAll ordered by name, getByName COLLATE NOCASE, getById, upsert REPLACE, markRun, deleteById, deleteAll, count) |
| `repository/MacroRepository.kt` | observeAll/getByName/getById/count, `save()` (trims name, validates via `MacroValidation`, enforces case-insensitive unique names, JSON-encodes steps), `rename()`, delete/deleteAll, `markRun()`; `toMacro()` returns null on unreadable steps JSON (never crashes) |
| `JarvisDatabase.kt` | Version **4**, entities += `MacroEntity`, real `MIGRATION_3_4` (CREATE TABLE `macros`), `macroDao()`, migrations list updated |
| `DataModule.kt` | Provides `MacroDao` + `MacroRepository` |
| `SettingsRepository.kt` | Persists `automationEnabled` (new `KEY_AUTOMATION` boolean key) |

### `:app` — the service, recorder, and UI (`accessibility/`, `screens/`, …)

| File | What it does |
| --- | --- |
| `accessibility/JarvisAccessibilityService.kt` | The real `AccessibilityService`. `onServiceConnected`/`onUnbind` register/clear the holder + state. While recording, `onAccessibilityEvent` maps TYPE_VIEW_CLICKED / TYPE_VIEW_TEXT_CHANGED / TYPE_VIEW_SCROLLED to `ObservedUiEvent`s (password text dropped at the mapping site, own package and blocked apps skipped, scroll direction from fromIndex/toIndex). Public suspend ops, all on `Dispatchers.Main.immediate`, all returning `JarvisResult`: `snapshot()` (sanitized, caps 200 nodes / depth 12 / 100 chars of text; password nodes skipped entirely, `hasPasswordField` recorded), `tap()` (finds node preferring interactive matches, clicks node or nearest clickable ancestor), `tapAt()` (dispatchGesture, 100 ms stroke, 5 s timeout), `typeText()` (matched field or `findFocus(FOCUS_INPUT)`; refuses password/non-editable), `scroll()` (first scrollable node), `pressBack()`/`pressHome()` (global actions — allowed on password screens and in blocked apps, they only navigate away). `actionGuard()` (blocklist + password-screen check) gates tap/type/scroll/gesture. **No logging of screen content anywhere** |
| `accessibility/AccessibilityState.kt` | `AccessibilityState` (`isConnected` StateFlow, `setConnected`, `isServiceEnabled(context)` via the AccessibilityManager enabled-service list, `openAccessibilitySettingsIntent()`); `AccessibilityServiceHolder` with `@Volatile var service` |
| `accessibility/MacroRecorder.kt` | `@Singleton` recorder: `RecordingState` (Idle / Recording(steps, skippedPasswordEntries, startedAtMillis)), `start()`/`observe()` (password text events bump the skipped counter, never recorded)/`stop()`/`cancel()`/`skippedPasswordEntries()` |
| `accessibility/AccessibilityAutomationDevice.kt` | `@Singleton` `AutomationDevice` impl: caches `automationEnabled` from the SettingsRepository Flow; `checkAvailable()` fails clearly when the toggle is off or the service is not connected; delegates to the bound service. Banking/password guards live in the service, not here |
| `accessibility/RoomMacroStore.kt` | `@Singleton` `MacroStore` adapter over `MacroRepository` |
| `res/xml/accessibility_service_config.xml` | Event types typeViewClicked\|typeViewTextChanged\|typeViewScrolled only; feedbackGeneric; flags `flagReportViewIds\|flagRetrieveInteractiveWindows`; canRetrieveWindowContent + canPerformGestures true; notificationTimeout 100. Touch-exploration NOT requested |
| `AndroidManifest.xml` | Service entry (exported=true, `BIND_ACCESSIBILITY_SERVICE` permission, intent-filter, meta-data to the config XML). No new Android permissions — Accessibility is a user-enabled service, not a permission |
| `res/values/strings.xml` | `accessibility_service_label` ("Zaid Jarvis automation") + `accessibility_service_description` — the prominent Play-policy disclosure text |
| `di/AppModule.kt` | Provides `AutomationDevice` (AccessibilityAutomationDevice) + `MacroStore` (RoomMacroStore) |
| `viewmodel/MacrosViewModel.kt` | Macros, recordingState, automationEnabled, accessibilityConnected, one-shot `message` flow; `runMacro()` (gated executor, job registered with the kill switch), `startRecording()`/`cancelRecording()`/`stopRecordingAndSave(name)` (refuses empty recordings, reports skipped password entries), `renameMacro()`/`deleteMacro()` |
| `screens/MacrosScreen.kt` | Explanation, recording banner (step count, skipped passwords, Stop & save / Cancel), Start recording button (disabled with a reason when the service is disconnected or automation is off), macro list with expandable step lists, Run/Rename/Delete, rename + save-recording + message dialogs (date format "d MMM yyyy, HH:mm") |
| `navigation/JarvisNavGraph.kt` | New `MacrosDestination`, reachable from Home and Settings, back button shown |
| `screens/HomeScreen.kt` | "Macros" suggestion chip |
| `screens/SettingsScreen.kt` | New "Automation" section: explanation, service status Connected / Enabled (connecting…) / Not enabled, "Open Accessibility settings" button when not enabled, automation toggle enabled only when the service is enabled, macro count, "View macros" button. Service status re-read on resume |
| `viewmodel/SettingsViewModel.kt` | Injects MacroRepository; exposes `accessibilityConnected` + `macroCount` flows; `setAutomationEnabled()` + `isAccessibilityServiceEnabled()` |
| `agent/Planner.kt` | System prompt gained rule (6): automation capability (find/tap/type/scroll + run_macro when Accessibility is enabled; still approved first; never password fields or banking apps) |

## Tool table (7 new tools; 27 total)

| Tool | Tier | What it does | Gate / notes |
| --- | --- | --- | --- |
| `find_on_screen` | 0 — Read-only | Describes the current screen's interactive elements (text + description + bounds, password fields excluded); optional `query` filter | Ask once per session (Phase 2 Tier 0 rule) |
| `tap_on_screen` | 1 — Reversible | Taps a node matched by text/description/viewId, or at x,y coordinates | Card each time; dryRun shows exactly what will be tapped |
| `type_text` | 1 — Reversible | Sets text in the focused editable field or a matched field; refuses password fields | Card each time; dryRun shows the target + exact text |
| `scroll_screen` | 1 — Reversible | Scrolls forward/back on the current screen | Card each time |
| `press_back` | 1 — Reversible | Global back action | Card each time |
| `press_home` | 1 — Reversible | Global home action | Card each time |
| `run_macro` | 1 or 2 — see rule | Executes a saved macro by name | **Tier rule:** Tier 2 (External) if the macro contains any TAP/TAP_AT/TYPE_TEXT step or the macro is not found (fail-safe); Tier 1 (Reversible) only for navigation-only macros (scroll/back/home/wait). The whole macro is approved once as a single card listing every step |

No new Android permissions. Accessibility is a user-enabled service,
not a permission — the user must enable it manually (see Setup below).

## Safety rules (hard requirements, all implemented)

- **Never acts on its own.** The service only executes steps that came
  through the `GatedToolExecutor` after user approval. There is no
  background automation, no scheduled automation, and no code path from
  the service to an action except through a gated tool call.
- **Never reads or stores passwords.** Nodes with `isPassword()` are
  skipped entirely in snapshots (`snapshot()`), in text-setting
  (`typeText()` refuses), in node matching (`NodeSelector` never
  matches a password node), and in recording (`RecorderLogic` drops
  password text/clicks; the recorder counts skipped entries and
  reports them — it does not record a placeholder step, because a
  placeholder step would replay as a wrong action).
- **Never acts in banking/payment apps or on password screens.**
  `BlockedApps` (segment-based package matching, best-effort — see
  Limitations) + a password-field check on the live screen gate
  tap/type/scroll/gesture. Back/home are the only actions allowed on
  such screens, because they only navigate away. Recording is also
  disabled in blocked apps.
- **Screen content is data.** Text read from the screen is scanned by
  the Phase 3 `InjectionGuard` before the planner acts on it (the
  planner scans every tool result, including `find_on_screen` output).
- **Snapshots are never stored.** Snapshots exist only for the current
  run, in memory, and are never logged (the service logs no screen
  content at all).
- **Playback is verified step by step.** `MacroPlayer` verifies the
  target exists (via a fresh snapshot) before each TAP/TYPE_TEXT step,
  checks the kill-switch generation before every step, enforces a
  10 s per-step timeout and a 50-step maximum, and stops at the first
  failure, reporting exactly which step failed and what was seen
  instead. It never continues blindly.

## Setup (user action required)

1. Open **Settings → Automation** in the app and tap
   **"Open Accessibility settings"** (or go to Android
   Settings → Accessibility → Downloaded apps → Zaid Jarvis
   automation) and turn the service on.
2. Return to the app — the Automation section will show "Connected".
3. Turn on the **automation tools** toggle (it stays greyed until the
   service is enabled). It is **off by default**: automation is a
   powerful capability that should only be live when the user has
   deliberately enabled it.
4. Record a macro from **Home → Macros** (or Settings → View macros):
   tap "Start recording", do the steps in the other app, come back,
   tap "Stop & save", name it. Run it later from the same screen or
   by asking Jarvis.

## Verification

Gradle cannot run in this VM (sandbox intercepts localhost TCP, so
the Gradle daemon never receives dispatches; Java HTTP clients cannot
traverse the egress proxy). Verified with kotlinc 2.4.21 + the
Compose/serialization plugins against real dependency JARs and
`android.jar` API 36, and JUnitCore for unit tests
(`~/opt/verify-pom/verify-phase6.sh` — not part of the project):

- **300 unit tests pass** — :core 123, :tools 117, :agent 43, :data 17
  (83 new in this phase: 42 core, 34 tools, 7 data).
- **All Android sources compile** via kotlinc — **525 class files,
  0 errors** (only the 2 pre-existing Phase 1 deprecation warnings
  remain: `Icons.Filled.Chat`, `menuAnchor()`).
- **Secret scan clean** — no secrets in code or logs.

Bugs found and fixed during verification:

- `JarvisAccessibilityService.findNode()` / `findScrollable()` first
  returned raw `AccessibilityNodeInfo` references that the
  traversal's `finally` block then recycled — fixed by returning
  `AccessibilityNodeInfo.obtain()` copies and recycling them at the
  call site.
- `recycle()` / `obtain()` are deprecated on API 36 (no-ops there)
  but still the correct node-lifecycle calls on older APIs
  (minSdk 26) — kept deliberately, deprecation suppressed at the
  class level with an explanatory comment.
- One redundant cast in `MacroPlayerTest` removed.

### What was NOT verified (needs Salahuddin's machine / a real device)

- The real Gradle build (`./gradlew assembleDebug`), Hilt/KSP codegen,
  Room schema + the v3→v4 migration (a schema mismatch would crash
  the app on start — check logcat), resource merging, and the
  manifest.
- **All on-device behaviour**: the AccessibilityService connecting,
  node finding, gestures, macro recording/playback, and the
  service-status UI all need a real device with the service enabled.
  No emulator or Accessibility framework exists in this VM.
- The first real build + on-device test must happen on Salahuddin's
  machine / Android Studio.

## Honest limitations

- **Automation is inherently fragile.** It depends on other apps'
  UI structure (view IDs, text, layout). An app update can move a
  button, rename a view ID, or change a label, and a macro that
  worked yesterday will fail at that step. `MacroPlayer` reports
  exactly which step failed and what it saw instead, so a broken
  macro is diagnosable — but macros will need re-recording as apps
  change.
- **The banking/payment blocklist is a best-effort guard, not a
  guarantee.** It covers well-known packages and common name
  segments, but it cannot know every banking app in every country.
  The password-field rule (never act on a screen containing a
  password field) is the real backstop, and the approval card is
  the final check: the user sees exactly what will be tapped or
  typed before it happens.
- **Screen reading is approximate.** Snapshots are capped at 200
  nodes / depth 12 / 100 characters of text per node, and only
  interactive nodes are described to the LLM (max 30). Very dense
  screens may hide the element the user wants; `find_on_screen`
  with a `query` filter helps, and coordinates (`tap_at`) are the
  fallback.
- **The wake-word recognizer and chat recognizer are separate
  instances** (Phase 4 known risk, unchanged): if both listen at
  once, one gets ERROR_RECOGNIZER_BUSY.
- **Play Store policy:** Accessibility use requires a prominent
  disclosure — the disclosure text is already in
  `accessibility_service_description` and in the Settings →
  Automation explanation. The full Play-policy checklist is part of
  Phase 11 hardening.

## What remains

- **Phase 7** — Research & content engine (needs approval).
- **Phase 8** — Social publishing (OAuth + scheduler).
- **Phase 9** — Productivity, proactive & wellness.
- **Phase 10** — Plugin system, backup, analytics.
- **Phase 11** — Hardening (security review, performance, Play-policy
  checklist — incl. the Accessibility disclosure already in place).
- **Phase 12** — Release build (signed APK/AAB — will require manual
  steps from Salahuddin: enable Accessibility, create social
  developer accounts, paste API keys/OAuth credentials).
