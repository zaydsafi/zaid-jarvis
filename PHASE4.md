# Phase 4 — Voice & UI (voice engine, chat polish, overlay bubble, widget, tile)

**Status: complete, 2026-11-15.** App version `0.4.0-phase4`. 140 files total
(was 111 in Phase 3).

Phase 4 implements Master Prompt §5 (Voice & UI): the voice engine
(SpeechRecognizer + TTS + wake word + "Jarvis stop"), a polished chat UI,
the floating overlay bubble, the home-screen widget, and the Quick
Settings tile. Phone control is still Phase 5 — Jarvis can now *talk*,
but its tools are still only the safe Tier 0 set from Phase 3.

## Scope

- Voice engine: speech-to-text with partial results, text-to-speech at
  the Settings speed, wake-word listening, "Jarvis stop" voice kill switch.
- Chat UI polish on the Phase 3 ChatScreen: mic button, listening
  indicator, timestamps, tool-progress lines, per-message speak button,
  suggested prompts.
- Overlay bubble (SYSTEM_ALERT_WINDOW) with a mini chat that reuses the
  Phase 2 ApprovalCard.
- Home-screen widget (classic AppWidget): orb + mic + last-answer line.
- Quick Settings tile: tap to talk, state reflects listening.
- Settings: wake word, voice replies, floating bubble toggles.
- Permissions: RECORD_AUDIO (first mic tap only), foreground-service
  types, overlay permission flow.

## What was built

### `:core` — pure voice logic (`core/voice/`, all unit-tested)

- `VoiceState.kt` — `VoiceState` (Idle / Listening / Processing /
  Speaking / Error) and `SpeechInputState` (platform-neutral recognizer
  events), shared by chat, overlay, and tile.
- `WakeWordMatcher.kt` — "Hey Jarvis" detection: case- and
  punctuation-insensitive, word-sequence based (never a substring, so
  "I asked Jarvis yesterday" does not wake the app), with common
  mis-hearings ("hey jar vis", "a jarvis", "ok jarvis") and bare
  "Jarvis" on its own. `stripWakeWord` removes a leading wake word so
  the rest goes to the planner.
- `VoiceCommandParser.kt` — "Jarvis stop" / "Stop Jarvis" detection
  (checked on partial transcripts too, so a stop lands mid-sentence);
  anything else is a planner request with the wake word stripped.
- `SpeechTextSanitizer.kt` — strips markdown, code fences, links,
  URLs, bullets, and emoji before TTS reads an answer aloud; caps
  utterances at 2,000 characters. The chat bubble keeps the original.
- `OverlayStateMachine.kt` — Hidden / Collapsed / Dragging / Expanded
  with pure transitions; a tap that ends a drag never expands the bubble.
- `LastAnswer.kt` — flattens/truncates the latest answer for the widget
  (80 chars + ellipsis).
- `model/JarvisSettings.kt` — gains `wakeWordEnabled` (default off),
  `voiceRepliesEnabled` (default on), `floatingBubbleEnabled`
  (default off). Voice language reuses the existing `language` setting;
  voice speed already existed.

### `:data` — settings + last answer

- `datastore/SettingsRepository.kt` — persists the three new settings.
- `repository/LastAnswerRepository.kt` (new) — own small DataStore for
  the latest answer + timestamp, so the widget (a BroadcastReceiver)
  can read it without the app running. Only the latest answer is kept.
- `di/DataModule.kt` — provides `LastAnswerRepository`.

### `:app` — voice engine (`voice/`)

- `SpeechInput.kt` — `SpeechInput` interface + `AndroidSpeechInput`
  (SpeechRecognizer + RecognitionListener, partial results, language
  from Settings, plain-language error messages for every recognizer
  error code, graceful message when no recognizer is installed).
- `SpeechOutput.kt` — `SpeechOutput` interface + `AndroidSpeechOutput`
  (TextToSpeech, init queueing, language fallback to the device voice,
  speech rate from Settings, sanitized text, stop on demand).
- `VoiceEngine.kt` — coordinates input + output; one owner for the mic
  and the speaker. Final transcripts go to a SharedFlow that only the
  ChatViewModel collects. The kill switch stops listening and speaking
  immediately. `@ActivityRetainedScoped` (services/tiles share state
  through `VoiceStateStore`, never the engine itself).
- `VoiceStateStore.kt` (new, `@Singleton`) — shared `isListening` flow
  (for the tile) and `voiceStartRequests` / `openChatRequests` counters
  (deep-links from widget / tile / wake word / Home orb).
- `WakeWordService.kt` (new) — foreground service (`microphone` type)
  that loops the platform SpeechRecognizer and watches transcripts for
  "Hey Jarvis", then opens Chat with voice listening. **Honest limits:**
  this is NOT true always-on hotword detection — Android has no public
  always-on hotword API for third-party apps; real hotword spotting
  needs a dedicated library (e.g. Porcupine, Vosk) or OEM support. The
  loop holds the mic continuously and uses more battery; it pauses
  while another app holds the mic and restarts afterwards, with
  backoff after errors and a 10-second cooldown after each wake.
- `di/VoiceModule.kt` (new) — Hilt bindings for `SpeechInput` /
  `SpeechOutput`, deliberately unscoped so chat and wake-word get
  separate recognizers (sharing one would make them steal the mic
  from each other; if both listen at once, one gets
  ERROR_RECOGNIZER_BUSY and reports it).

### `:app` — chat (`chat/`, `viewmodel/`, `screens/`)

- `chat/ChatSession.kt` (new, `@Singleton`) — the single conversation
  session, extracted from ChatViewModel so the chat screen and the
  overlay bubble share one planner run, one message list, one kill
  switch. `ChatUiMessage` gains an id and a timestamp. Every final
  answer is saved to `LastAnswerRepository` and the widget is refreshed.
- `viewmodel/ChatViewModel.kt` (rewritten) — delegates conversation to
  the session and adds voice: routes final transcripts (stop command →
  kill switch + silence + "Stopped."; request → planner as a voice
  run), watches partial transcripts for "Jarvis stop", speaks assistant
  answers for voice runs when voice replies are on, exposes the voice
  state and the deep-link voice-start counter.
- `screens/ChatScreen.kt` (rewritten) — mic button (RECORD_AUDIO is
  requested only on the first tap, never at app start), listening
  indicator (pulsing orb + live partial transcript + "Say Jarvis stop
  to cancel"), speaking/error rows, message bubbles with timestamps,
  quiet full-width system lines for tool progress ("Running
  get_time…", "get_time finished."), per-message speak button on
  Jarvis messages, suggested prompts in the empty state, auto-scroll.
- `screens/HomeScreen.kt` — tapping the orb opens Chat and starts
  voice listening.
- `MainActivity.kt` — `singleTop` + intent extras
  (`EXTRA_DESTINATION`, `EXTRA_START_VOICE`); warm launches route
  through `VoiceStateStore` in `onNewIntent`.
- `navigation/JarvisNavGraph.kt` — start destination from the launch
  intent; observes the deep-link counters and navigates to Chat.

### `:app` — overlay bubble (`overlay/`)

- `BubbleService.kt` (new) — foreground service (`specialUse` type)
  drawing the orb over other apps (TYPE_APPLICATION_OVERLAY). Tap
  expands to the mini chat; drag moves it (drag never counts as a
  tap); long-press dismisses it and turns the Settings toggle off.
  The window hosts Compose without an Activity: it owns its own
  `Recomposer` (via `setParentCompositionContext`) and a minimal
  `LifecycleOwner` (RESUMED while shown). Collapsed = small
  non-focusable window; expanded = focusable bottom sheet-sized
  window so the keyboard works.
- `OverlayChat.kt` (new) — the mini chat: same `ChatSession` as the
  main screen (text input + send + close + mic button that opens the
  app to Chat with voice), and the **Phase 2 `ApprovalCard` reused
  in place** when a tool call awaits a decision (Allow / Edit / Deny
  / trust, with the expiry countdown). Second confirmations and
  biometric checks need the app (BiometricPrompt requires a
  FragmentActivity), so those states show an "Open app to confirm"
  button instead of failing silently.
- `OverlayPermission.kt` (new) — `canDrawOverlays` check + the system
  settings intent for "Display over other apps".

### `:app` — widget + tile (`widget/`, `tile/`)

- `widget/JarvisWidgetProvider.kt` + `widget/WidgetUpdater.kt` (new)
  — **classic AppWidget (RemoteViews), not Glance**: no new
  dependency, works on every supported API level, and the layout is
  a single row. Orb + last-answer line + mic button (tap → Chat with
  voice listening; tap elsewhere → Chat). Updates are pushed after
  every answer (`updatePeriodMillis="0"`, no polling).
  Resource ids are resolved with `getIdentifier` instead of `R.*` —
  the kotlinc verification in this VM has no generated R class, so
  Kotlin code must not reference R directly; on a real Gradle build
  these resolve to the same ids.
- `res/layout/widget_jarvis.xml`, `res/xml/jarvis_widget_info.xml` (new).
- `tile/JarvisTileService.kt` (new) — Quick Settings tile "Talk to
  Jarvis": tap opens Chat with voice listening; state is ACTIVE
  while the mic is live (via `VoiceStateStore`), INACTIVE otherwise.
  Uses the PendingIntent `startActivityAndCollapse` overload on API
  34+ (the Intent overload is deprecated there).

### Settings + manifest

- `screens/SettingsScreen.kt` — "Voice & personality" gains: speak
  answers aloud, wake word (asks for mic permission on enable, with
  the mic/battery note), floating bubble (greyed out with a "Grant
  Display over other apps" button until the permission is granted;
  state refreshes on resume).
- `viewmodel/SettingsViewModel.kt` — setters start/stop the
  wake-word and bubble services; wake word refuses to start without
  mic permission.
- `AndroidManifest.xml` — RECORD_AUDIO (already declared), new:
  FOREGROUND_SERVICE, FOREGROUND_SERVICE_MICROPHONE,
  FOREGROUND_SERVICE_SPECIAL_USE, SYSTEM_ALERT_WINDOW; WakeWordService
  (microphone type), BubbleService (specialUse type + subtype
  property), widget receiver, tile service
  (BIND_QUICK_SETTINGS_TILE). MainActivity is now `singleTop`.
- `res/values/strings.xml` — tile label, widget description.
- App version `0.4.0-phase4`.

## Permissions (what Zaid will see, and when)

| Permission | When asked | Why |
|---|---|---|
| Microphone (RECORD_AUDIO) | First mic tap, or when enabling the wake word | Voice input / wake word |
| Display over other apps | Settings → Floating bubble → system screen | The overlay bubble |
| (Foreground services) | Granted at install (normal permissions) | Wake word + bubble services |

Nothing is requested at app start. TTS needs no permission. The tile
cannot request the mic itself — Chat asks on the first mic tap.

## Verification

**Full Gradle build (`./gradlew assembleDebug`) still CANNOT run in this VM**
— same limitation as Phases 1–3. Tried once this phase:
`./gradlew --no-daemon -q help` fails with the same SSL/proxy-tunnelling
errors (Java HTTP clients cannot traverse the egress proxy); the Gradle
daemon dispatch failure also still applies. **Do not treat the project as
build-verified until `./gradlew assembleDebug` passes on a real machine or
in Android Studio, and no on-device behaviour has been tested — the first
real build + on-device test must happen on Salahuddin's machine.**

What WAS verified (2026-11-15, in this VM) via
`~/opt/verify-pom/verify-phase4.sh` (kotlinc 2.4.21, `-jvm-target 17`,
+ Compose + serialization plugins, against the real dependency JARs and
`android.jar` API 36). **Final run: exit 0, all passed.**

- **`:core` compiles cleanly** (98 class files, 0 errors); **79/79 unit
  tests pass** — 49 from Phases 1–3 + 30 new: 8 WakeWordMatcherTest
  (variants, mis-hearings, false positives, strip), 7
  VoiceCommandParserTest (stop variants, mid-sentence stop,
  non-command "stop"), 6 SpeechTextSanitizerTest (markdown/code/links/
  emoji/truncation), 6 OverlayStateMachineTest (show/tap/drag/
  long-press/hidden), 4 LastAnswerTest.
- **`:tools` compiles cleanly on the JVM** (15 class files, 0 errors);
  **13/13 unit tests pass** (unchanged).
- **All Android module sources compile cleanly** (`app`, `data`, `ui`,
  `agent`, `tools`, `plugins`): **311 class files, 0 errors**. Only the
  2 pre-existing Phase 1 deprecation warnings remain (Chat icon,
  `menuAnchor()`); the new deprecation warnings found in the first run
  (Send / VolumeUp / OpenInNew icons, `startActivityAndCollapse`) were
  fixed to the AutoMirrored icons and the API 34+ PendingIntent
  overload and the compile re-run clean. This validates syntax and
  types — but NOT annotation processing (Hilt/Room codegen), resource
  merging, or the Android manifest.
- **`:agent` tests: 43/43 pass** (unchanged).
- **`:data` tests: 6/6 pass** (unchanged).
- **Total: 141 unit tests pass** (79 + 13 + 43 + 6).
- **No secrets in code**: grep for API-key/password/token patterns across
  the project found nothing.

Bugs found and fixed during verification (all re-verified in the final run):
`ViewTreeLifecycleOwner` is a Kotlin file facade, not a class — kotlinc
cannot resolve it as a type (javac can, which is how the cause was
isolated); the service now uses the `setViewTreeLifecycleOwner`
extension function. `AbstractComposeView` has no public
`compositionContext` property; the overlay now passes its Recomposer
via `setParentCompositionContext`. New deprecation warnings (above)
were fixed rather than left.

## What was NOT verified (needs a real device)

- SpeechRecognizer / TextToSpeech behaviour, wake-word loop timing and
  battery use, "Jarvis stop" end-to-end, overlay drag/tap/long-press
  and the approval card inside the overlay, widget updates and taps,
  tile state changes, the overlay-permission and mic-permission flows,
  and the foreground-service notifications — none of these can run in
  this VM (no emulator, no mic, no system UI).
- Hilt codegen for the new bindings (VoiceModule, ActivityRetainedScoped
  VoiceEngine, @AndroidEntryPoint services/tile), Room (unchanged this
  phase), resource merging (widget layout/info XML, strings), and the
  manifest (foreground-service types, tile, widget receiver) — all need
  a real Gradle build.

## Known risks for the first real Gradle run / on-device test

- Hilt + KSP codegen is untested here — this phase adds a new Hilt
  module, an ActivityRetainedScoped engine, and entry-point services.
- The overlay's manual Recomposer + LifecycleOwner is the standard
  pattern for Compose outside an Activity, but it is only exercised
  on a real device; if composition does not start, the bubble will
  show nothing (check logcat for the Recomposer).
- Wake-word and chat recognizers are separate instances: if both try
  to listen at once, one reports ERROR_RECOGNIZER_BUSY and the UI
  says to try again. Turning the wake word off while chatting avoids it.
- AGP 9.5's built-in Kotlin support may interact with the explicitly
  applied `kotlin-android` plugin (carried over from Phase 1).

## What remains

- **Phase 5** — Phone tools (tiers 0–2) + real undo actions for reversible tools.
- **Phases 6–12** — Accessibility automation, research/content, social
  publishing, productivity, plugins, hardening, release build.
