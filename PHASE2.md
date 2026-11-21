# Phase 2 — Confirmation Engine + audit log (complete)

**Scope:** the permission gate every tool call passes through (Master Prompt
§6), plus the audit log, the kill switch, and tests. No agent, no voice, no
phone tools, no publishing — those are later phases. No new top-level
modules were created; the engine lives in `:core` (pure Kotlin, JVM-tested).

## What was built

### Core engine — `:core` (`core/confirmation/`)
- **`RiskTier`** — the canonical tier enum (levels 0–3), moved here from
  `:tools` so the engine needs no Android dependency. `:tools` keeps the
  `ToolRiskTier` name as a typealias, so existing references still work.
- **`ConfirmationEngine`** — the gate. Tier routing (0: ask once per
  session, then auto-allow unless `alwaysAskReadOnly`; 1: card each time;
  2: card + second confirmation; 3: card + biometric), 2-minute approval
  expiry, SHA-256 content binding (the action runs with the *approved*
  content; an edit creates a brand-new request), single-use approvals,
  trusted-action auto-allow for tiers 0–1 only, kill-switch generation
  checks, and audit logging of every outcome. Time is injected (`clock`)
  for deterministic tests.
- **`ApprovalRequest`** — everything the card shows (action, target, data
  used, exact content) plus the stored content hash and lifecycle state.
- **`KillSwitch`** — generation counter + job registry + cancel listeners;
  `cancelAll()` cancels every registered job and the pending approval.
- **`AuditLog` / `AuditEntry` / `AuditDecision` / `InMemoryAuditLog` /
  `AuditLogExporter`** — audit contract, in-memory impl (tests), CSV export
  with proper quoting.
- **`TrustedActionStore` / `InMemoryTrustedActionStore`** — trust/revoke
  contract and test impl.
- **`JarvisSettings.alwaysAskReadOnly`** — new setting (default off).

### Gate integration — `:tools`
- **`GatedToolExecutor`** — the single entry point for running tools:
  resolves the tool, previews it with `dryRun` (that preview is the exact
  content on the card), gates on the risk tier, executes only with the
  approved content. The agent (Phase 3), macros (Phase 6), and plugins
  (Phase 10) must all use it; `JarvisTool.execute` is never called directly.
- `ToolRegistry` is now a Hilt `@Singleton` with an `@Inject` constructor.
- Build fix: `:tools` now declares `serialization-json` (Phase 1 compiled
  only because the manual verification used one big classpath).

### Persistence — `:data`
- **`AuditLogEntryEntity` + `AuditLogDao`** — audit table with search
  (LIKE over tool/action/target/params/decision), get/delete/clear/mark-undone.
- **`JarvisDatabase` v1 → v2** with a real `MIGRATION_1_2` (creates the
  audit table; existing data kept).
- **`RoomAuditLog`** — Room-backed `AuditLog` (encrypted at rest via SQLCipher).
- **`TrustedActionRepository`** — DataStore-backed trusted-action store.
- **`SettingsRepository`** — persists `alwaysAskReadOnly`.
- `DataModule` provides `AuditLogDao`, `AuditLog`, `TrustedActionStore`.

### UI — `:ui` and `:app`
- **`ApprovalCard`** (`:ui`, reusable by the Phase 4 overlay) — tier chip,
  action, target, data used, exact content (monospace), expiry countdown,
  Allow / Edit / Deny, inline edit field, and "always allow this action"
  for tiers 0–1.
- **`ApprovalHost`** (`:app`) — mounts once over every screen; shows the
  card dialog, the Tier 2 second-confirm dialog, and launches the Tier 3
  biometric prompt. Dialogs cannot be dismissed by tapping outside or by
  Back — Zaid must Allow, Edit, or Deny.
- **`BiometricAuth`** (`:app`) — `BiometricPrompt` with
  `BIOMETRIC_STRONG | DEVICE_CREDENTIAL`, so devices without enrolled
  biometrics fall back to the device PIN/pattern/password. If neither is
  available, Tier 3 actions cannot proceed and Jarvis only assists.
  `MainActivity` now extends `FragmentActivity` (required by BiometricPrompt).
- **`AuditLogScreen` + `AuditLogViewModel`** — list, search, detail dialog,
  CSV export via share intent, delete, clear-all. Undo: entries record
  whether an action is undoable, but actually reversing an action needs the
  tool's inverse operation, which lands with the tools in Phase 5 — until
  then the Undo button says exactly that and marks nothing as undone.
- **Settings → Permissions & trust** — "always ask for read-only" toggle,
  trusted-actions list with per-action Revoke and Revoke-all, link to the
  audit log.
- **Kill switch UI** — red stop button in the top app bar (every screen)
  and a "Stop Jarvis" button on Home, with snackbar confirmation.
- **Navigation** — `AuditLogDestination` (reachable from Home and Settings),
  back button on the audit screen.
- `AppModule` provides the singleton `KillSwitch` and `ConfirmationEngine`.
- Version catalog adds `androidx.biometric:biometric:1.1.0` (POM verified);
  app version is now `0.2.0-phase2`.

## Verification

**Full Gradle build (`./gradlew assembleDebug`) still CANNOT run in this VM**
— same limitation as Phase 1. Tried once: `./gradlew --no-daemon -q help`
fails with SSL/proxy-tunnelling errors (Java HTTP clients cannot traverse
the egress proxy); the Gradle daemon dispatch failure from Phase 1 also
still applies. **Do not treat the project as build-verified until
`./gradlew assembleDebug` passes on a real machine or in Android Studio.**

What WAS verified (2026-11-15, in this VM):

- **`:core` compiles cleanly** with kotlinc 2.4.21 + serialization plugin:
  60 class files, 0 errors.
- **All 32 unit tests pass** (`JUnitCore`: `OK (32 tests)`) — 4 Phase 1
  settings tests + 28 new: 18 ConfirmationEngine tests (tier routing for
  all four tiers, second confirmation, biometric gating, 2-minute expiry,
  stale-request expiry, edit → new request, content-hash binding, single-use
  rejection, unknown-id rejection, trusted auto-allow + revoke, tiers 2–3
  never trusted, kill-switch cancel, deny auditing, stale second-confirm
  rejection, session end), 4 KillSwitch tests, 6 AuditLog tests
  (record/search/delete/clear/mark-undone, CSV quoting).
- **All Android module sources compile cleanly** with kotlinc 2.4.21 +
  Compose compiler plugin + serialization plugin, against the real
  dependency JARs (102 artifacts, incl. newly fetched biometric 1.1.0 and
  fragment 1.9.1) and `android.jar` (API 36): **127 class files, 0 errors**.
  Only the 2 pre-existing Phase 1 deprecation warnings remain (Chat icon,
  `menuAnchor()`); the one new warning introduced (ArrowBack) was fixed to
  the AutoMirrored icon and the compile re-run clean.
  This validates syntax and types across `app`, `data`, `ui`, `agent`,
  `tools`, `plugins` — but NOT annotation processing (Hilt/Room codegen,
  incl. the new Room migration and DAOs), resource merging, or the Android
  manifest. Those need a real Gradle run.
- **No secrets in code**: grep for API-key/password/token patterns across
  the project found nothing.

## Known risks for the first real Gradle run

- Hilt + KSP codegen and Room schema generation are untested here — this
  phase adds a Room migration, a new DAO, new Hilt providers, and a
  `FragmentActivity` base class, all of which only a real Gradle build
  exercises.
- AGP 9.5's built-in Kotlin support may interact with the explicitly
  applied `kotlin-android` plugin (carried over from Phase 1).

## What remains

- **Phase 3** — Agent core: real Claude/OpenAI/Gemini API clients, planner
  loop (must route every tool call through `GatedToolExecutor`), memory,
  token/cost reporting.
- **Phase 4** — Voice (SpeechRecognizer/TTS, wake word, "Jarvis stop"),
  chat UI, overlay bubble (reuses `ApprovalCard`), widget, Quick Settings tile.
- **Phase 5** — Phone tools (tiers 0–2) + real undo actions for reversible tools.
- **Phases 6–12** — Accessibility automation, research/content, social
  publishing, productivity, plugins, hardening, release build.
