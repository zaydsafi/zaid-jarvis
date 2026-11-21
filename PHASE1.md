# Phase 1 — Skeleton (complete)

**Scope:** multi-module project, theme, navigation, Hilt, settings, secure key
storage. No agent, no voice, no phone tools, no publishing — those are later
phases.

## What was built

- **Project structure** — 7 Gradle modules (`app`, `core`, `data`, `ui`,
  `agent`, `tools`, `plugins`), Gradle version catalog
  (`gradle/libs.versions.toml`), `gradlew` wrapper (Gradle 9.6.0), AGP 9.5.1,
  Kotlin 2.3.21, Compose BOM 2026.09.00.
- **App** — `Zaid Jarvis` (`com.zaid.jarvis`), single activity
  (`MainActivity`), Navigation Compose with type-safe destinations (Home,
  Chat, Settings) and a bottom bar.
- **Theme** — Material 3, Light / Dark / AMOLED (true black) / Follow system,
  applied from the saved setting.
- **Home screen** — voice-first layout with an animated Jarvis orb (breathing
  glow; live voice state wiring is Phase 4) and quick-action chips.
- **Chat screen** — placeholder; real chat lands in Phases 3–4.
- **Settings screen** — theme, language (English/Urdu/Arabic), voice speed
  slider, personality (formal/friendly/concise), LLM provider + model
  selection (Claude/OpenAI/Gemini), masked API-key fields, token/cost counter
  placeholder.
- **Secure storage** — API keys in EncryptedSharedPreferences backed by a
  Keystore master key; Room database encrypted with SQLCipher using a random
  passphrase stored the same way. No secrets in code.
- **Settings persistence** — DataStore Preferences via `SettingsRepository`.
- **Stubs with TODOs** — `agent` (LLM client contract + planner stub),
  `tools` (tool contract + risk tiers), `plugins` (plugin contract).
- **Build** — R8 enabled for release, ProGuard rules, `scripts/make_keystore.sh`
  (passwords prompted, never committed), `.gitignore` for keystores/APKs.
- **Tests** — `core` unit tests for settings/provider consistency
  (`JarvisSettingsTest`, 4 tests).

## Verification

**Full Gradle build (`./gradlew assembleDebug`) could NOT run in this VM.**
The environment was set up from scratch (Temurin JDK 17.0.21, Android SDK
platform 36 + build-tools 36.0.0 installed manually, Gradle 9.6.0), but the
sandbox intercepts localhost TCP: the Gradle daemon starts and listens, yet
the Gradle client cannot dispatch a single message to it
(`DaemonConnectionException: Could not dispatch a message to the daemon`,
reproduced across heap sizes, daemon/no-daemon modes, and with proxy env
vars unset). Java-based HTTP clients also cannot get through the egress
proxy (sdkmanager, Gradle wrapper download, and Maven all fail with
`SSLException` / proxy-tunnelling errors), so dependency resolution via
Gradle/Maven is impossible here. **The project has therefore not had a
Gradle build run against it — do not treat it as build-verified until
`./gradlew assembleDebug` passes on a real machine or in Android Studio.**

What WAS verified (2026-11-15, in this VM):

- **`:core` compiles cleanly** with kotlinc 2.4.21 + serialization plugin,
  and **all 4 unit tests pass** (`JarvisSettingsTest`, JUnit 4.13.2:
  `OK (4 tests)`). The tests caught and fixed a real bug: a circular
  static-initialisation dependency between the `LlmProvider` and `LlmModel`
  enums (see the NOTE in `LlmProvider.kt`).
- **All Android module sources compile cleanly** with kotlinc 2.4.21 +
  Compose compiler plugin + serialization plugin, against the real
  dependency JARs (99 artifacts resolved with curl from Google Maven /
  Maven Central) and `android.jar` (API 36): **60 class files, 0 errors**.
  This validates syntax and types across `app`, `data`, `ui`, `agent`,
  `tools`, `plugins` — but NOT annotation processing (Hilt/Room codegen),
  resource merging, or the Android manifest. Those need a real Gradle run.
- **Every version in `gradle/libs.versions.toml` was checked to exist**
  (HTTP 200 on its POM): AGP 9.5.1, Kotlin 2.3.21, KSP 2.3.14, Compose BOM
  2026.09.00, Hilt 2.52, Room 2.7.1, Navigation 2.9.0, DataStore 1.1.1,
  WorkManager 2.9.0, Retrofit 2.11.0, OkHttp 4.12.0, SQLCipher 4.5.4,
  security-crypto 1.1.0-alpha06, serialization 1.8.0, coroutines 1.10.1,
  lifecycle 2.8.7, activity-compose 1.10.1, sqlite-ktx 2.5.2,
  hilt-navigation-compose 1.2.0, Gradle 9.6.0.
  Two corrections came out of this: KSP has no 2.4.x release yet, so the
  project uses Kotlin 2.3.21 + KSP 2.3.14; and the available Android platform
  is API 36, so `compileSdk` is 36 (not 37).
- **No secrets in code**: grep for API-key/password patterns across the
  project found nothing. API keys live only in Keystore-backed
  EncryptedSharedPreferences at runtime.
- `gradle/libs.versions.toml` parses as valid TOML; `gradlew` /
  `gradlew.bat` / `gradle-wrapper.jar` are the genuine Gradle 9.6.0 wrapper
  files (script from the Gradle repo's v9.6.0 tag, jar extracted from the
  9.6.0 distribution).

## Known risks for the first real Gradle run

- Hilt + KSP codegen and Room schema generation are untested here.
- AGP 9.5's built-in Kotlin support may interact with the explicitly
  applied `kotlin-android` plugin; if the build complains, drop the
  explicit plugin (AGP 9.0+ configures Kotlin itself).
- Compose BOM 2026.09.00 pins the material-icons version; the manual
  verification used material-icons 1.7.8 (latest published).

## What remains

- **Phase 2** — Confirmation Engine + audit log (the permission gate every
  tool call passes through).
- **Phase 3** — Agent core: real Claude/OpenAI/Gemini API clients, planner
  loop, memory, token/cost reporting.
- **Phase 4** — Voice (SpeechRecognizer/TTS, wake word), chat UI, overlay
  bubble, widget, Quick Settings tile.
- **Phases 5–12** — Phone tools, Accessibility automation, research/content,
  social publishing, productivity, plugins, hardening, release build.
