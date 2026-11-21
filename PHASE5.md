# Phase 5 — Phone tools (tiers 0–2) + real undo

**Status: complete, 2026-11-15.** App version `0.5.0-phase5`. 156 files total
(was 140 in Phase 4).

Phase 5 implements the phone-control tools from Master Prompt §3 and §6
(tiers 0–2 only — Tier 3 sensitive actions stay gated and are NOT
implemented as tools) plus real undo actions for reversible tools. The
app now has 20 tools: the 3 pure tools from Phase 3 and 17 phone tools.

## Scope

- Tier 0 (read-only): battery, network, contacts search, installed apps.
- Tier 1 (reversible): flashlight, volume, brightness, Wi-Fi, Bluetooth,
  alarm, timer, open app, Do Not Disturb.
- Tier 2 (external/visible): dialer, messaging, email, share sheet —
  **intent-only**: Jarvis opens the system app with everything
  pre-filled; Zaid always takes the final step there.
- Real undo: every reversible tool records the previous device state,
  and the Audit Log's Undo button runs the inverse through the
  Confirmation Engine.
- Content binding: an SMS/email body edited on the approval card is
  what actually runs (or the tool refuses to run edited content).
- Permissions: least-privilege requests + a Settings → App permissions
  screen. CALL_PHONE and SEND_SMS are deliberately NOT declared.

## What was built

### `:core` — undo plumbing

- `confirmation/AuditLog.kt` — `AuditEntry` gains `undoData: String?`
  (the JSON payload a tool captured at execution time, e.g. the
  previous volume level) and a new `UndoInfo` data class
  (description + data).
- `confirmation/ConfirmationEngine.kt` — `execute()` gains two optional
  parameters: `undoInfoProvider` (consulted only after a *successful*
  run, so denied/expired/failed actions never get undo data) and
  `paramsJsonProvider` (consulted at audit time so the audit log
  records the parameters that actually ran after an edit). The Phase 2
  placeholder text ("undo lands with the tool in Phase 5") is gone:
  an entry is undoable only when the tool actually supplied undo data.

### `:tools` — contract, device abstraction, 17 tools

- `ToolStub.kt` — `JarvisTool.execute` now returns
  `JarvisResult<ToolOutcome>` where `ToolOutcome` carries the message
  plus optional `undoData` / `undoDescription`. New undo contract:
  `supportsUndo`, `undoPreview(params, undoData)`,
  `undo(params, undoData)`. New content-binding hook:
  `paramsFromEditedContent(editedContent, originalParams)` — tools
  that can parse edited preview text re-derive their parameters from
  it; tools that return null refuse edited content.
- `ToolSchema.kt` — `ParamSpec` + `objectSchema(...)` for
  multi-parameter schemas (with optional `enum` values for the LLM).
- `JsonParams.kt` (new) — defensive typed parameter readers
  (`stringParam`, `intParam`, `boolParam`, …); tools never crash on
  bad input.
- `phone/PhoneDevice.kt` (new) — the pure-Kotlin device interface the
  tools program against, with data types (`BatteryInfo`, `NetworkInfo`,
  `Contact`, `AppInfo`, `VolumeStream`, `VolumeInfo`,
  `BrightnessChange`, `DndMode`, `WifiToggleResult`,
  `BluetoothToggleResult`). State-changing methods return the
  *previous* state so tools can record real undo data. Every method
  returns a `JarvisResult` — missing permissions, missing hardware,
  and Android version restrictions are failures with plain-language
  messages, never crashes.
- `phone/SystemAction.kt` (new) — pure-data descriptions of the intents
  Jarvis asks Android to start (Dial, SendSms, SendEmail, ShareText,
  OpenApp, SetAlarm, SetTimer, OpenWifiPanel, RequestBluetoothEnable,
  OpenClockApp). Tools build them; the device layer converts them to
  Intents. Tests assert exactly which action a tool produced.
- `phone/ContactResolver.kt` (new) — pure name → number resolution:
  exact → prefix → word-prefix → substring, with ties reported as
  Ambiguous (max 5 candidates) so the planner asks Zaid. Jarvis never
  guesses which "John" to call.
- `phone/NoOpPhoneDevice.kt` (new) — a device that does nothing (for
  tests and builds without phone access); every call fails clearly.
- `phone/Tier0PhoneTools.kt` (new) — `get_battery`, `get_network`,
  `get_contacts` (READ_CONTACTS, checked before the query),
  `list_installed_apps` (launcher apps only, optional filter).
- `phone/Tier1PhoneTools.kt` (new) — `set_flashlight` (CAMERA),
  `set_volume` (6 streams, level validated against the stream max),
  `set_brightness` (0–100%, WRITE_SETTINGS handled by the device
  layer, auto-brightness noted), `set_wifi` (Android 10+ panel
  fallback), `set_bluetooth` (BLUETOOTH_CONNECT, Android 13+ enable
  dialog), `set_alarm` / `set_timer` (Clock app intents),
  `open_app` (exact package/name → launch; ambiguous → candidates),
  `set_do_not_disturb` (4 modes, policy access handled by the device
  layer). Every tool's `dryRun` shows the exact effect with the
  current state (e.g. "Set media volume to 7/15 (was 4/15)"); a no-op
  (setting a value already set) records no undo data.
- `phone/Tier2PhoneTools.kt` (new) — `make_call`, `send_sms`,
  `send_email`, `share_text`. All four only produce a `SystemAction`;
  the dialer/messaging/email app opens pre-filled and Zaid taps Call
  or Send. `send_sms` / `send_email` / `make_call` / `share_text`
  implement `paramsFromEditedContent` so edits on the approval card
  are honoured; mangled edits are refused.
- `GatedToolExecutor.kt` — adapts `ToolOutcome` to the planner-facing
  `JarvisResult<String>`, passes the undo/params providers to the
  engine, resolves edited content before executing, and adds
  `executeUndo(entry)`: undo runs through the same Confirmation
  Engine at the tool's own tier, so a Tier 1 undo shows its own
  approval card.
- `ToolsModule.kt` — the registry now takes a `PhoneDevice` and
  registers all 20 tools.
- `builtin/` — GetTimeTool, CalculatorTool, EchoTool return
  `ToolOutcome` now (planner-facing behaviour unchanged).

### `:data` — audit undo data, network type

- `db/AuditLogEntry.kt` — new `undoData` column.
- `db/JarvisDatabase.kt` — version 3 with a real `MIGRATION_2_3`
  (`ALTER TABLE … ADD COLUMN undoData TEXT`).
- `repository/RoomAuditLog.kt` — maps `undoData` both ways.
- `device/DeviceContextProvider.kt` — new `networkType()` (Wi-Fi /
  Mobile / Ethernet / VPN / None) for `get_network`.

### `:app` — Android implementation, permissions, undo UI

- `phone/AndroidPhoneDevice.kt` (new) — the real `PhoneDevice`:
  CameraManager torch (with a TorchCallback keeping state in sync),
  AudioManager volumes, Settings.System brightness (WRITE_SETTINGS
  check + auto-brightness detection), WifiManager (direct toggle
  pre-API 29, settings panel on API 29+), BluetoothAdapter (direct
  toggle, system enable dialog on API 33+), NotificationManager DND
  (policy-access check), ContactsContract search, launcher-apps list
  via `queryIntentActivities` (no QUERY_ALL_PACKAGES needed), and
  `launch()` building every intent (ACTION_DIAL — never ACTION_CALL;
  ACTION_SENDTO for SMS/email; chooser for sharing; AlarmClock
  intents with the Clock UI shown). All work on Dispatchers.IO,
  all failures plain-language.
- `permissions/PermissionHelper.kt` (new) — the permission catalogue:
  runtime permissions (Contacts, Camera, Bluetooth, Microphone) with
  rationales, plus special-access checks and intents (Modify system
  settings, Do Not Disturb access, app settings page).
- `di/AppModule.kt` — provides the singleton `PhoneDevice`.
- `viewmodel/AuditLogViewModel.kt` — `undo(entry)` now runs the real
  undo through the gated executor and marks the entry undone only
  after the inverse succeeds; the result appears in `undoMessage`.
- `screens/AuditLogScreen.kt` — shows the undo result dialog from the
  ViewModel's message flow.
- `screens/SettingsScreen.kt` — new "App permissions" section: every
  tool permission with its grant status and a Grant button (runtime
  dialog or system settings screen), refreshed on resume.
- `AndroidManifest.xml` — READ_CONTACTS, CAMERA, BLUETOOTH_CONNECT
  (+ legacy BLUETOOTH/BLUETOOTH_ADMIN maxSdkVersion 30),
  ACCESS_NETWORK_STATE, ACCESS_WIFI_STATE, CHANGE_WIFI_STATE,
  MODIFY_AUDIO_SETTINGS, SET_ALARM, ACCESS_NOTIFICATION_POLICY,
  WRITE_SETTINGS, and the optional flash `uses-feature`. CALL_PHONE
  and SEND_SMS are deliberately absent (documented in the manifest).
- App version `0.5.0-phase5`.

### `:agent` — offline commands

- `Planner.kt` — offline set grows: battery ("how much battery") and
  network ("am I online") now work offline too, through the same
  gated executor (Tier 0).

## Tool table

| Tool | Tier | Permissions / access | Undo |
|---|---|---|---|
| get_time (Phase 3) | 0 | — | — |
| calculator (Phase 3) | 0 | — | — |
| echo (Phase 3) | 0 | — | — |
| get_battery | 0 | — | — |
| get_network | 0 | ACCESS_NETWORK_STATE (normal) | — |
| get_contacts | 0 | READ_CONTACTS (runtime) | — |
| list_installed_apps | 0 | — | — |
| set_flashlight | 1 | CAMERA (runtime) | Restores previous on/off |
| set_volume | 1 | MODIFY_AUDIO_SETTINGS (normal) | Restores previous level |
| set_brightness | 1 | WRITE_SETTINGS (special) | Restores previous % |
| set_wifi | 1 | CHANGE_WIFI_STATE (normal) | Restores previous state (direct toggle only; panel path has no undo) |
| set_bluetooth | 1 | BLUETOOTH_CONNECT (runtime) | Restores previous state (direct toggle only) |
| set_alarm | 1 | SET_ALARM (normal) | Opens Clock app for manual deletion* |
| set_timer | 1 | SET_ALARM (normal) | Opens Clock app for manual cancellation* |
| open_app | 1 | — | None (can't un-open an app) |
| set_do_not_disturb | 1 | ACCESS_NOTIFICATION_POLICY + policy access (special) | Restores previous mode |
| make_call | 2 | READ_CONTACTS when resolving a name | None (external) |
| send_sms | 2 | READ_CONTACTS when resolving a name | None (external) |
| send_email | 2 | — | None (external) |
| share_text | 2 | — | None (external) |

\* Android has no API for an app to cancel an alarm or timer created
through the Clock app, so alarm/timer undo opens the Clock app and
Zaid deletes it there. The undo preview and result say exactly that —
Jarvis never claims a programmatic cancel it cannot perform.

## Permissions (what Zaid will see, and when)

| Permission / access | When asked | Why |
|---|---|---|
| Contacts (READ_CONTACTS) | First contact search / call-or-text by name | Find contacts by name |
| Camera (CAMERA) | First flashlight use | Torch control |
| Bluetooth (BLUETOOTH_CONNECT) | First Bluetooth toggle | Switch Bluetooth |
| Microphone (RECORD_AUDIO) | First mic tap / wake word (Phase 4) | Voice input |
| Modify system settings | First brightness change (error points here) | Screen brightness |
| Do Not Disturb access | First DND change (error points here) | DND mode |
| Display over other apps | Floating bubble (Phase 4) | Overlay bubble |

Nothing is requested at app start. Wi-Fi, volume, alarm/timer, and
network-state permissions are normal permissions granted at install.
All of the above can also be granted ahead of time in Settings → App
permissions.

## Verification

**Full Gradle build (`./gradlew assembleDebug`) still CANNOT run in this VM**
— same limitation as Phases 1–4. Tried once this phase:
`./gradlew --no-daemon -q help` fails with the same SSL/proxy-tunnelling
errors (Java HTTP clients cannot traverse the egress proxy); the Gradle
daemon dispatch failure also still applies. **Do not treat the project as
build-verified until `./gradlew assembleDebug` passes on a real machine or
in Android Studio, and no on-device behaviour has been tested — the first
real build + on-device test must happen on Salahuddin's machine.**

What WAS verified (2026-11-15, in this VM) via
`~/opt/verify-pom/verify-phase5.sh` (kotlinc 2.4.21, `-jvm-target 17`,
+ Compose + serialization plugins, against the real dependency JARs and
`android.jar` API 36). **Final run: exit 0, all passed.**

- **`:core` compiles cleanly** (99 class files, 0 errors); **81/81 unit
  tests pass** — 79 from Phases 1–4 + 2 new: undo info recorded on a
  successful run; no undo info on a denied or failed run.
- **`:tools` compiles cleanly on the JVM** (109 class files, 0 errors);
  **83/83 unit tests pass** — 13 from Phases 1–3 + 70 new: PhoneToolsTest
  (schemas for all 17 tools, dry-run previews, execution against the
  fake device, undo round-trips for flashlight/volume/brightness/wifi/
  bluetooth/DND, permission-missing failures, Tier 2 intent production,
  alarm/timer action construction, open-app ambiguity), 9
  ContactResolverTest (exact, case-insensitive, prefix, word, ties,
  none, number detection), 9 GatedUndoTest (undo data in the audit log,
  undo through the gate restoring state, denied undo changing nothing,
  already-undone/non-undoable refusals, no undo data on failure, SMS
  body edited on the card being what actually runs, unparseable edits
  refused, 20-tool registry).
- **All Android module sources compile cleanly** (`app`, `data`, `ui`,
  `agent`, `tools`, `plugins`): **433 class files, 0 errors**. Only the
  2 pre-existing Phase 1 deprecation warnings remain (Chat icon,
  `menuAnchor()`); one new warning found in the first run (unnecessary
  safe call on `loadLabel`) was fixed and the compile re-run clean.
  This validates syntax and types — but NOT annotation processing
  (Hilt/Room codegen, incl. the Room v3 migration), resource merging,
  or the Android manifest.
- **`:agent` tests: 43/43 pass** (unchanged; PlannerTest updated for
  the `ToolOutcome` return type and the `PhoneDevice` registry param).
- **`:data` tests: 10/10 pass** — 6 from Phase 3 + 4 new
  RoomAuditLogTest (undo-data round-trip, markUndone, null undo data,
  search/delete).
- **Total: 217 unit tests pass** (81 + 83 + 43 + 10).
- **No secrets in code**: grep for API-key/password/token patterns across
  the project found nothing.

Bugs found and fixed during verification (all re-verified in the final run):
`GatedToolExecutor.executeUndo` passed a no-arg lambda where the engine
expects `(String) -> JarvisResult<String>`; `AndroidPhoneDevice.setWifi` /
`setBluetooth` called the suspend `launch()` from a non-suspend `io`
lambda (the helper now takes a suspend block); an unnecessary safe call
on `ResolveInfo.loadLabel`.

## What was NOT verified (needs a real device)

- Flashlight, volume, brightness, Wi-Fi, Bluetooth, DND, contacts,
  app listing/opening, dialer/SMS/email/share intents, alarm/timer
  intents, and the undo flows end to end — none of these can run in
  this VM (no emulator, no system services, no system apps).
- The Room v2→v3 migration, Hilt codegen for the new `PhoneDevice`
  binding, resource merging, and the manifest (new permissions,
  `uses-feature`) — all need a real Gradle build.
- The permission-request flows in Settings → App permissions and the
  special-access settings screens.

## Known risks for the first real Gradle run / on-device test

- Hilt + KSP codegen is untested here — this phase adds the
  `PhoneDevice` binding and a `ToolsModule` provider that takes it.
- Room's schema validation for the v3 migration is untested here; if
  Room disagrees with the migration, the app will crash on start with
  a schema error (check logcat; the migration is a single
  `ALTER TABLE … ADD COLUMN`).
- `BluetoothAdapter.enable()` / `disable()` are deprecated on API 33+;
  they still work on most devices, but if a device refuses, the tool
  reports the refusal honestly instead of claiming success.
- `WifiManager.setWifiEnabled` only exists as a direct toggle below
  API 29; the panel fallback is the only path on modern Android.
- The torch state is tracked in memory (CameraManager has no getter);
  if another app toggles the torch while Jarvis is not running, the
  first `dryRun` after relaunch may report the wrong current state
  until the TorchCallback fires.
- AGP 9.5's built-in Kotlin support may interact with the explicitly
  applied `kotlin-android` plugin (carried over from Phase 1).

## What remains

- **Phase 6** — AccessibilityService automation + macro recorder.
- **Phases 7–12** — Research/content, social publishing,
  productivity, plugins, hardening, release build.
