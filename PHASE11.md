# Phase 11 — Hardening (security review, performance, Play-policy checklist)

**Status: complete (2026-11-16). Version `0.11.0-phase11`.**
**Project: 349 files (306 `.kt`). Tests: 704 passed (60 new).**

Phase 11 is a REVIEW + FIX phase, not a features phase. No new tools,
no new feature screens. It went over everything built in Phases 1–10 and
made it safe, tight, and ready for a real release build in Phase 12.

Two real bugs were found and fixed (a publishing-gate bypass and a
whole-table load in the LLM context), plus one manifest bug (the widget
receiver could never receive updates), one OAuth ordering fix, and the
DNS-rebinding SSRF hardening deferred from Phase 7. Everything else in
the audit was verified sound — the findings below say which is which.

## 1. Security audit — what was checked, found, and fixed

### 1.1 DNS-rebinding SSRF hardening (deferred from Phase 7) — FIXED

- New `agent/.../agent/research/ValidatingDns.kt`: a custom OkHttp `Dns`
  that resolves the hostname, validates **every** resolved address
  against the private/reserved ranges, and throws `UnknownHostException`
  if any address is non-public. Mixed answers (one public, one private)
  fail closed.
- `core/.../core/research/UrlSafety.kt` extended with
  `isNonPublicAddress(InetAddress)` — byte-level checks, not stdlib
  predicates (verified empirically: `isSiteLocalAddress()` is false for
  unique-local `fc00::/7` and CGNAT `100.64/10`, and both it and
  `isLoopbackAddress()` are false for IPv4-mapped/compatible forms).
  Covers IPv4 ranges incl. CGNAT, `192.0.0.0/24`, `198.18/15`, `0/8`,
  `240/4`; IPv6 loopback/unspecified/link-local/unique-local/multicast;
  IPv4-mapped `::ffff:a.b.c.d` and IPv4-compatible `::a.b.c.d` forms;
  unknown lengths fail closed. The literal check `isNonPublicIp` was
  fixed for mapped/compatible forms too (previously `::ffff:127.0.0.1`
  slipped through).
- Wired in `AgentModule.kt` — `.dns(ValidatingDns())` on the **shared**
  OkHttpClient, so it covers the page fetcher, both search clients, and
  (deliberately) the LLM/OAuth/social clients: those hit fixed public
  `https` endpoints, so the guard is a no-op in practice and
  defense-in-depth. Custom providers are `https`-only since this phase,
  so a custom provider resolving to a private IP is refused.
- Alternative spellings verified with a scratch program:
  `InetAddress.getByName("2130706433")` → `127.0.0.1` (caught; a test
  asserts this); `0x7f000001` throws `UnknownHostException` in Java
  (fails closed).
- **What this guarantees and what it does not** (documented in the
  `ValidatingDns` KDoc): OkHttp connects only to the `InetAddress`
  objects `lookup()` returns, so there is no TOCTOU gap between
  validation and connect for the addresses OkHttp uses; each lookup
  (including each redirect hop) is validated independently. It does NOT
  protect against a malicious server at a public IP, does not validate
  TLS, does not pin DNS answers across lookups, and does not unwrap
  6to4/Teredo. Note the redirect context: before this phase,
  `OkHttpPageFetcher` checked the final URL *after* OkHttp had already
  followed the redirect — the request to the private host went out and
  only the reading was refused. The validating Dns now blocks the
  connection itself.
- Tests: `ValidatingDnsTest` (10 tests: public allowed, private refused,
  mixed refused, loopback/link-local refused, unresolvable → clear
  error, empty answer refused, mapped forms refused/allowed as
  appropriate, integer form refused); `UrlSafetyTest` +9.

### 1.2 REAL BUG — scheduled-path publishing-gate bypass — FIXED

Editing an APPROVED draft keeps status APPROVED; only the approval hash
catches it. `publishDraftNow` checked `editedSinceApproval()`, but
`SchedulePostTool` did not, and `publishScheduled` only checked the
schedule-time content hash — which was recorded *from the edited text*
if the edit happened before scheduling. So an edited-after-approval
draft could be scheduled and published with text Zaid never approved.

Fixed in three layers:
- `tools/.../tools/social/PublishingTools.kt` — `SchedulePostTool.execute`
  refuses edited-after-approval drafts; `SchedulePostTool.dryRun` and
  `PublishDraftTool.dryRun` show the refusal on the card (publish
  execute already refused; its card used to offer the publish anyway).
- `agent/.../agent/social/PublishServiceImpl.kt` — `publishScheduled`
  now cancels (with reason) when `draft.editedSinceApproval()`, covering
  posts scheduled before this fix.
- Defense-in-depth: `data/.../repository/DraftRepository.kt` —
  `markPublished()` now also refuses when `draft.editedSinceApproval()`
  (the gate in `PublishServiceImpl` remains the primary enforcement).
- 4 + 2 regression tests (`PublishingToolsTest` +3, `PublishServiceTest`
  +1, `DraftRepositoryTest` +2).

Publishing gate otherwise verified: `markPublished` has exactly 2 call
sites, both in `PublishServiceImpl` after gate checks;
`SocialPublisher.publish` only from `PublishServiceImpl`;
`publishScheduled` only from `PublishPostWorker`; the worker handles
deleted drafts (FAILED), un-approved/edited drafts (CANCELLED +
notification), missing rows (returns success, nothing to do). Draft
editor + retry flows all go through gated `schedule_post` /
`publish_draft` tools.

### 1.3 Confirmation Engine / GatedToolExecutor — verified, one bound added

- The only `JarvisTool.execute()` call site in the project is
  `GatedToolExecutor` (inside the engine's approved-content block);
  every caller (planner, all ViewModels, macros) goes through
  `GatedToolExecutor.execute`. `MacroPlayer` is constructed only in
  `ToolsModule` for `RunMacroTool`. Plugin tools only reach the planner
  via `PluginRegistry` → `ToolRegistry` and execute only via the
  executor; disabling unregisters immediately; name collisions are
  skipped, never overwrite built-ins. `executeUndo` is itself gated.
- **Fixed:** `consumedApprovalIds` in `ConfirmationEngine` was an
  unbounded set growing once per executed/edited approval. Now capped
  at `MAX_CONSUMED_APPROVAL_IDS` = 1,000 with eldest eviction
  (`rememberConsumed()`). Safe because ids are UUIDs and single-use by
  construction: an id can only be resolved while its request is the
  currently pending one, so a completed request's id can never be
  presented again. Test with a bound of 3: 5 approvals run, set stays
  at 3. (`sessionApprovedTier0` is bounded by the number of tools; the
  `awaiting` map holds at most one entry per pending request.)
- **Accepted limitation (not fixed):** `_pendingRequest` is a single
  slot — two concurrent `execute()` calls could overwrite each other's
  pending card (the first request's deferred stays in `awaiting` but
  `resolve()` only resolves the current pending request, so it could
  hang until expiry). In practice single-threaded (one `ChatSession` /
  planner run; workers don't use the engine). A Mutex serialization of
  `execute()` is a possible future hardening.

### 1.4 OAuth — verified, one ordering fix

- State validated, single-use in-memory pending flow, process death →
  clean "interrupted, retry" failure (code matches docs). Refresh: 401
  → refresh once → retry once; second 401 or failed refresh → tokens
  cleared, account disconnected.
- **Fixed** (`app/.../social/OAuthFlowManager.kt`): the `error` branch
  used to run *before* state validation, so a forged redirect to the
  exported activity could surface attacker-controlled `error_description`
  text; state is now validated first, for error responses too.
- Notes: Meta's token endpoints require client secret / access token in
  GET query strings — Meta's required API shape over https, not fixable
  here. `OAuthRedirectActivity` has no `onNewIntent` override — a second
  redirect while the first is processing is dropped; safe because the
  pending flow is single-use, and the user retries.

### 1.5 Secret audit — clean

- Zero `android.util.Log` calls in main sources (only `auditLog.`
  false positives), zero `println` / `printStackTrace`, zero hard-coded
  keys/tokens/passwords. Keys flow only into client `Authorization`
  headers via call-time lambdas; tokens only into
  `AuthenticatedApiCaller`. No logging interceptor anywhere.
- `SecureKeyStore`: MasterKey AES256_GCM + EncryptedSharedPreferences
  (AES256_SIV keys / AES256_GCM values); the DB passphrase is a random
  32-byte hex value generated once. `SocialAccountRepository`: tokens +
  client secrets only in SecureKeyStore; only handle/expiry metadata in
  plaintext DataStore. `JarvisSettings` holds no secrets.
- Audit log: params redacted for key-like fields, results capped at 200
  chars, encrypted at rest (SQLCipher), CSV export user-initiated —
  acceptable by design. Note: `summarizeResult` includes the first 200
  chars of tool success messages in the (encrypted) audit log.
  Analytics payloads verified token-validated (free text structurally
  impossible). `BackupService`: export contains no secrets; import
  merges by id, restores SCHEDULED posts as CANCELLED and RUNNING focus
  sessions as CANCELLED, all in one Room transaction.
- Observations (not bugs): import restores trusted actions from the
  file (preview shows the count; the user confirms on a Tier-2-style
  dialog); imported APPROVED drafts keep their approval hash but still
  need a Tier 2 card to publish; imported macros are validated at play
  time.
- The verify script's secret scan was extended (Phase 11): GitHub
  tokens, AWS keys, Google OAuth tokens, Slack tokens, and private-key
  headers, and it now scans `.xml` and `.md` too. One false positive
  found by the extended scan: `PHASE9.md` quoted the old hyphenated
  `sk-…` word from the Phase 9 false positive — reworded (the code
  comment itself was already fixed in Phase 9).

### 1.6 Network security — config added, http custom providers refused

- New `app/src/main/res/xml/network_security_config.xml`
  (`cleartextTrafficPermitted="false"`, no exceptions), wired via
  `android:networkSecurityConfig` in the manifest. Verified no built-in
  flow needs http (DuckDuckGo lite, Tavily, Claude/OpenAI/Gemini/
  OpenRouter, all social API + OAuth endpoints are https).
- **Behaviour change (deliberate):** `CustomProviderConfig
  .validationErrors()` now refuses `http://` with exactly: *"Base URL
  must use https:// — plain HTTP would send your API key in clear
  text"*. This removes the Phase 3 "http:// allowed for local servers"
  behaviour — a local Ollama/llama.cpp/vLLM server now needs an https
  endpoint (e.g. a reverse proxy) to be usable. The README was updated
  to match. Wiring already existed: `SettingsScreen` calls
  `validationErrors()`, displays errors, disables Add/Save when
  non-empty; `LlmClientFactory.clientFor` already refuses invalid
  configs, so a pre-existing http:// provider (or one imported from a
  backup) can never be used. Optional hardening not taken:
  `SettingsViewModel.addCustomProvider` / `updateCustomProvider` don't
  validate themselves — could check `config.validationErrors().isEmpty()`
  before saving.
- Tests: `CustomProviderConfigTest` (http refusal incl. localhost, https
  still valid; net +2).

### 1.7 Deep-link / intent safety — verified

- `MainActivity` extras only select a destination (chat/tasks/wellness)
  and optionally start voice listening — no extras trigger any action;
  all actions still go through the normal gated path.
- Widget/tile/notifications all use `FLAG_IMMUTABLE` PendingIntents
  (verified: every `PendingIntent` in the app); no `FLAG_MUTABLE`, no
  `sendBroadcast`, no WebView / `JavascriptInterface` anywhere.
- Residual (documented, no fix available): any app can launch the
  exported `MainActivity` with `EXTRA_START_VOICE=true` — but the app
  comes to the foreground visibly, the system mic indicator shows, and
  it requires the already-granted mic permission; there is no reliable
  sender check for explicit intents.

### 1.8 Exported components + manifest — two fixes

- **Widget receiver fixed (real bug):** `JarvisWidgetProvider` was
  `android:exported="false"` — `APPWIDGET_UPDATE` is broadcast by
  `system_server`, so `exported=false` blocked widget updates entirely.
  Now `exported="true"` with a justifying comment. The custom
  `ACTION_REFRESH` action only refreshes widget UI, so exporting it is
  safe.
- **`android:allowBackup="false"`** with an explanatory comment: the app
  has its own backup/export (Phase 10); Auto Backup would copy the
  SQLCipher DB (useless without Keystore keys, which don't transfer)
  and plaintext DataStore settings to cloud backup. Confirmed no code
  depends on Auto Backup (no `BackupAgent` / `fullBackupContent`).
- Verified: `android:exported` explicit on every component; MainActivity
  (launcher), OAuth redirect (browser return, state-validated),
  AccessibilityService (`BIND_ACCESSIBILITY_SERVICE`), QS tile
  (`BIND_QUICK_SETTINGS_TILE`) all have real reasons and minimal intent
  filters. **QUERY_ALL_PACKAGES, CALL_PHONE, SEND_SMS are NOT declared**
  (only comments mention them); installed-apps listing uses
  `queryIntentActivities` with a launcher intent.
- Manifest + `network_security_config.xml` validated well-formed with
  Python's XML parser (manifest merge itself is unverified until a real
  Gradle build).

### 1.9 InjectionGuard coverage — no gap found

The planner scans *every* tool result before it reaches history / the
next turn (notes, screen content, web content, calendar, contacts,
briefing all flow through tool results). `ResearchEngine` additionally
scans each fetched page *before* the synthesis prompt; search snippets
never reach the LLM. Briefing LLM-polish output goes to the
user/notification, never to tool calls except via the planner (scanned).
The guard is heuristic by design (documented).

### 1.10 Accessibility — verified, no fix

Password nodes skipped in snapshots, `typeText` refuses password fields,
`actionGuard` refuses tap/type/scroll/gesture on password screens and
in blocked apps (Back/Home only navigate away), recording drops password
text and skips blocked apps, no screen content logged. `BlockedApps` is
segment-based and explicitly documented as best-effort with the
password-field rule as the backstop — the documentation holds.

### 1.11 "No LLM configured" — verified graceful in all four paths

Chat shows the factory's plain message as a system message; Briefing
uses its plain templated fallback and never fails; Research emits a
`Failed` event with the message (note: it searches/fetches *before*
resolving the client, so it does wasted web work before failing — minor
inefficiency, left alone); Drafting returns the failure. No crashes, no
silent failures.

## 2. Performance review — what was checked, found, and fixed

No profiler is available in this VM — the performance audit is code
review only; no CPU/memory/battery measurements were taken. Stated
honestly in every place it matters.

### 2.1 DAO query LIMITs / bounded lists — three real bugs fixed

Audited all 10 DAOs + repositories. Every list query is now bounded:
explicit `LIMIT` params on DAOs (no Kotlin defaults, to avoid any
Room-codegen question), repositories pass caps from the new
`data/.../db/ListLimits.kt` (audit list 1,000 / search 500, tasks 2,000
list / 5,000 bulk, notes 500 / 5,000 bulk, memory 500, research 200,
drafts 500, macros 500, scheduled posts 500 / 5,000 bulk, focus sessions
500 / 5,000 bulk, analytics 10,000). `getAll()` deliberately stays
uncapped (backup export/merge must see the whole table); fast-growing
tables are bounded by the retention policy instead.

- **Real bugs fixed:** `MemoryRepository.summaryForPrompt()` loaded ALL
  memory rows then `.take(maxItems)` — now `dao.search("", maxItems)`;
  `FocusSessionRepository.getRecent()` loaded ALL sessions then
  `.take(limit)` — now `dao.getRecent(limit)`; `SettingsViewModel
  .macroCount` loaded all macros for a count — now `dao.observeCount()`
  (`SELECT COUNT(*)`).
- Task filters (`getOpen` / `getOverdue` / `getDueToday`) now filter in
  SQL (`status != 'DONE'` — preserves the repo's fail-safe mapping of
  unknown statuses to OPEN; day boundaries via `java.time` in device
  zone).
- New DAO methods: `getAllIds()` (all tables, for backup planning),
  `AuditLogDao.count()` / `deleteBeyondNewest(keepCount)`,
  `AnalyticsEventDao.deleteOlderThan(cutoff)`, `TaskDao.getOpen` /
  `getOverdue` / `getDueBetween`, `FocusSessionDao.getRecent`,
  `MacroDao.observeCount`.
- **Tool-context caps (coordinator fix):** `ListTasksTool` and
  `ListScheduledPostsTool` formatted *all* rows into the LLM context.
  Both now cap the formatted output at 50 items with an "…and N more"
  line (`MAX_LISTED_TASKS` / `MAX_LISTED_POSTS`). The underlying stores
  are bulk-capped at 5,000, so memory was already bounded; this caps
  the LLM context. Tests: one per tool.
- **Audit Log search (coordinator fix):** `AuditLogViewModel` searched
  in memory over the capped (1,000) list, so older entries were
  unfindable. It now runs the SQL search (`AuditLog.search`, DAO search
  capped at 500) via `flatMapLatest`, re-running whenever the log
  changes.

### 2.2 Maintenance worker + retention policy — new

- `data/.../db/RetentionPolicy.kt`: analytics 90 days, audit log keeps
  newest 10,000; pure functions `analyticsCutoffMillis(now)`,
  `auditLogDeletionsFor(count)`.
- `data/.../repository/MaintenanceService.kt`: prunes via
  `AnalyticsPruner` / `AuditLogPruner` interfaces (implemented by the
  real repos), never throws, each step independent, returns a
  `MaintenanceReport`.
- `app/.../backup/MaintenanceWorker.kt` +
  `app/.../backup/MaintenanceScheduler.kt`: weekly,
  `ExistingPeriodicWorkPolicy.KEEP`, synced from `JarvisApplication`
  on start, always returns success.
- Approvals: verified in-memory only (`ConfirmationEngine` — pending
  StateFlow + `awaiting` map, expiry checked at use). No persistence →
  no prune step; documented in `MaintenanceService` / `MaintenanceWorker`
  KDoc.
- ResponseCache: cap already enforced (50-entry access-ordered LRU +
  15-min TTL, covered by existing `ResponseCacheTest`). No change made.

### 2.3 Performance audit observations (no change needed)

- `Planner.kt` (read-only): bounded — `MAX_TURNS = 10`,
  `MAX_ATTEMPTS = 3`, backoff 500ms/1s/2s, kill-switch checked every
  turn and tool call. Minor observation: no per-turn cap on tool calls
  in one LLM response (each still needs approval except auto-allowed
  Tier 0 repeats / trusted).
- `WakeWordService.kt` (read-only): documented battery claims match
  code — platform recognizer loop, continuous mic, off by default,
  foreground service, restart backoff 500ms doubling to 5s max, 10s
  wake cooldown. Minor: `restartListening` launches a new coroutine per
  call without cancelling pending restarts (mostly harmless —
  `SpeechInput.start()` cancels first).
- WorkManager intervals: ProactiveCheck 15 min (the minimum, OK),
  DailyDigest 24h, AutoBackup 7d, Maintenance 7d. Nothing runs more
  often than needed.
- Bitmaps: zero `Bitmap` / `BitmapFactory` usage in the project.

### 2.4 Worker robustness — five workers hardened

All 5 workers now fail silent (return success) on unexpected exceptions
instead of retrying: `TaskReminderWorker` (retry after a successful
notify would double-notify), `FocusEndWorker` (KDoc already promised
never-retry; an exception would have violated it), `DailyDigestWorker`,
`ProactiveCheckWorker` (Settings read was the one unwrapped path),
`AutoBackupWorker` (Settings read).

### 2.5 Backup import re-verified — one O(n²) fixed

Strict parsing, SHA-256 checksum, 10 MB cap, single-transaction writes,
merge-by-id all present. **Fixed O(n²) bug:** `newScheduled.map { it.id
}.toSet()` was rebuilt inside the `newHistory` filter lambda per row —
hoisted to `newScheduledIds`. Preview/import merge planning now uses
`getAllIds()` instead of loading full rows (macros still use `getAll()`
— needs names for dedupe).

## 3. Robustness pass

### 3.1 Migrations re-read — no discrepancies

`MIGRATION_1_2`–`MIGRATION_7_8` re-read against all entity classes: SQL
syntax, column names/types match; new tables via `CREATE TABLE`; added
columns nullable with no default needed (`undoData TEXT`,
`approvedBodyHash TEXT`); no data loss. **Nothing changed.** Still
unverified until a real Gradle build + on-device run (see §6).

### 3.2 "Delete all data" flow — new, complete

New `data/.../repository/DataWipeService.kt`: an ordered `List<WipeStep>`
assembled in `AppModule` (single place listing what "all data" means);
runs every step, collects failures, `Failure` names failed stores.
Steps: Room DB (clearAllTables + close + `clearInstance()` + delete
`jarvis.db` / `-wal` / `-shm` / `-journal`) → Settings → Proactive
state → Plugin state → Trusted actions → Last answer → Social accounts
→ API keys → Secrets (Keystore passphrase last, so leftover encrypted
bytes are unrecoverable) → Files (auto-backups + cache via new
`AppFileWipe.kt`).

Verified all 6 DataStore files (`jarvis_settings`,
`jarvis_proactive_state`, `jarvis_plugin_state`,
`jarvis_trusted_actions`, `jarvis_last_answer`,
`jarvis_social_accounts`); no other SharedPreferences/DataStore usage;
`VoiceStateStore` is in-memory only; only `AutoBackupFiles` uses
`filesDir`. Added `clearAll()` / `clear()` to `SettingsRepository`,
`ProactiveStateRepository`, `PluginStateRepository`,
`SocialAccountRepository`.

UI: "Danger zone" section in `SettingsScreen.kt` +
`DeleteAllDataDialog`; biometric/device-credential via `BiometricAuth`
when available, typed `DELETE` fallback; wiping state disables
dismissal; failure shows a message. `SettingsViewModel` drives it
(guards double-tap, runs on `Dispatchers.IO`, cancels all WorkManager
work, stops WakeWordService/BubbleService/FocusSessionService, restarts
the app via `MainActivity` + `Process.killProcess` on success).

### 3.3 Hilt DI pattern — verified empirically, no change

The codebase's `@Inject constructor` + `@Provides`-for-the-same-type
pattern (14 repos/services in `DataModule`) was suspected to be a
Dagger duplicate-binding error. Tested with the real Dagger 2.52
compiler: **no error — the `@Provides` method wins and the `@Inject`
constructor is shadowed**, including when the shadowed constructor has
an unsatisfiable dependency (covers `MaintenanceService`'s default
`clock` and `AnalyticsRepository`'s default `isEnabled`). Control case
(inject-only) also passes. **No DI changes made.** Full sweep confirmed
no duplicates anywhere: `AppModule` provides interfaces + `KillSwitch` /
`ConfirmationEngine` / `DataWipeService` (none `@Inject`-constructed);
`AgentModule` / `ToolsModule` likewise clean. This validates the DI
*pattern* with real Dagger 2.52, not this project's full graph — the
full graph still needs a real Gradle build.

## 4. Play-policy checklist + privacy policy

- New standalone `PLAY_POLICY_CHECKLIST.md`: per sensitive
  permission/API — the policy requirement, what Jarvis does, and what
  still needs a human step (Play Console Accessibility API declaration,
  foreground-service declarations, data-safety form draft, content
  rating, advertising-ID declaration, privacy-policy URL). Verifies:
  Accessibility prominent disclosure text is still accurate and
  complete; QUERY_ALL_PACKAGES not declared; SYSTEM_ALERT_WINDOW
  justification + in-app disclosure; FOREGROUND_SERVICE types match
  actual use (microphone: WakeWordService; specialUse: BubbleService,
  FocusSessionService) with matching permission declarations;
  PACKAGE_USAGE_STATS justification (screen time, opt-in, on-device);
  READ_CALENDAR / READ_CONTACTS / RECORD_AUDIO / CAMERA /
  BLUETOOTH_CONNECT each with in-app justification and first-use
  runtime-permission flow; POST_NOTIFICATIONS runtime permission;
  target SDK 36 + Play target-API requirement.
- New `PRIVACY_POLICY.md` in the project root: plain-language, accurate
  to the code — what data Jarvis stores and where, what leaves the
  device and to which services (with links to each third party's
  privacy policy), retention (audit log 10,000 / analytics 90 days),
  and how to delete everything. **Human step:** fill in the developer
  contact details (marked `[…]` in the file), publish it at a public
  URL, and link it in Play Console (and in-app Settings once the URL
  exists).

## 5. Tests

**704 tests passed (60 new this phase).** New test classes:

| Class | Tests | Covers |
|---|---|---|
| `agent.research.ValidatingDnsTest` | 10 | DNS-rebinding guard (fake Dns) |
| `data.repository.RetentionPolicyTest` | 5 | 90-day cutoff / 10,000-entry cap math |
| `data.repository.MaintenanceServiceTest` | 6 | Prune steps, never-throws, report |
| `data.repository.DataWipeServiceTest` | 4 | Every store's clear called via fakes |
| `data.repository.ListLimitsTest` | 12 | Recording-fake cap tests, incl. regressions for the three whole-table-load bugs |

Extended: `UrlSafetyTest` (+9), `CustomProviderConfigTest` (+2),
`PublishServiceTest` (+1), `PublishingToolsTest` (+4: 3 gate regressions
+ 1 output cap), `ProductivityToolsTest` (+1: output cap),
`ConfirmationEngineTest` (+1: bounded consumed-ids set),
`DraftRepositoryTest` (+2: markPublished refusal),
`RoomAuditLogTest` (+2: prune), `AnalyticsRepositoryTest` (+1: prune).
All fake DAOs updated to the new `LIMIT` signatures.

## 6. Verification

Command: `bash ~/opt/verify-pom/verify-phase11.sh`
(extends `verify-phase10.sh` with the 5 new test classes above, and an
extended secret scan: GitHub/AWS/Google-OAuth/Slack token patterns and
private-key headers, now also scanning `.xml` and `.md`).
Full log: `/tmp/verify-p11-final.log`.

| Check | Outcome |
|---|---|
| Gradle step | Failed as expected in this VM (SSL/proxy — known limitation) |
| :core tests | **258 OK** |
| :tools tests | **171 OK** |
| :plugins tests | **35 OK** |
| :agent tests | **124 OK** |
| :data tests | **116 OK** |
| **Total** | **704 tests, 0 failures** |
| Android sources compile | **1,212 class files, 0 errors** |
| Secret scan (extended) | No secrets found |

Only warnings are the 2 pre-existing Phase 1 deprecations
(`Icons.Filled.Chat` in `JarvisNavGraph.kt`, `menuAnchor()` in
`SettingsScreen.kt`). One new warning found during verification —
`flatMapLatest` in `AuditLogViewModel` needs
`@OptIn(ExperimentalCoroutinesApi::class)` — was fixed in this phase.

### What was NOT verified (honest limits)

- `./gradlew assembleDebug` **cannot run in this VM** (sandbox intercepts
  localhost TCP so the Gradle daemon never receives dispatches; Java HTTP
  clients cannot traverse the egress proxy). First real build must happen
  on Salahuddin's machine / Android Studio.
- Hilt/Room codegen (incl. all migrations), resource merging, and the
  manifest merge are unverified until that real build. The network
  security config is well-formed XML but untested at runtime.
- `ValidatingDns` is tested with a fake Dns only — never against a real
  rebinding DNS.
- No on-device behaviour tested (delete-all-data wipe + restart,
  maintenance worker timing, widget updates after the export fix,
  OAuth redirect flow, Accessibility on a real device).
- Workers have no JVM-testable surface (no `:app` test source set in the
  verify script); the worker robustness changes are verified by compile
  + code review only.
- The performance audit is code review only — no profiler in this VM, no
  CPU/memory/battery measurements.
- InjectionGuard remains heuristic by design; the banking blocklist
  remains best-effort by design; the single-slot pending approval card
  (concurrency edge, §1.3) and the `EXTRA_START_VOICE` residual (§1.7)
  are accepted limitations.

## 7. Changed files

**New files (16):**
`agent/.../agent/research/ValidatingDns.kt`,
`app/src/main/res/xml/network_security_config.xml`,
`data/.../db/ListLimits.kt`, `data/.../db/RetentionPolicy.kt`,
`data/.../repository/MaintenanceService.kt`,
`data/.../repository/DataWipeService.kt`,
`app/.../backup/MaintenanceWorker.kt`,
`app/.../backup/MaintenanceScheduler.kt`,
`app/.../backup/AppFileWipe.kt`,
`PLAY_POLICY_CHECKLIST.md`, `PRIVACY_POLICY.md`, `PHASE11.md`,
tests: `ValidatingDnsTest`, `RetentionPolicyTest`,
`MaintenanceServiceTest`, `DataWipeServiceTest`, `ListLimitsTest`.

**Modified (main, ~50):** all 10 DAO files + `JarvisDatabase.kt`
(`DATABASE_NAME`, `clearInstance()`); repositories `RoomAuditLog`,
`AnalyticsRepository`, `TaskRepository`, `NoteRepository`,
`MemoryRepository`, `ResearchRepository`, `DraftRepository`,
`MacroRepository`, `ScheduledPostRepository`, `FocusSessionRepository`,
`BackupService`, `SettingsRepository`, `ProactiveStateRepository`,
`PluginStateRepository`, `SocialAccountRepository`; `UrlSafety.kt`,
`CustomProviderConfig.kt`, `AgentModule.kt`, `OkHttpPageFetcher.kt`
(KDoc only), `PublishServiceImpl.kt`, `PublishingTools.kt`,
`ProductivityTools.kt`, `ConfirmationEngine.kt`, `OAuthFlowManager.kt`,
`DataModule.kt`, `AppModule.kt`, `JarvisApplication.kt`;
workers `TaskReminderWorker`, `FocusEndWorker`, `DailyDigestWorker`,
`ProactiveCheckWorker`, `AutoBackupWorker`; `SettingsViewModel.kt`,
`SettingsScreen.kt`, `AuditLogViewModel.kt`, `AndroidManifest.xml`,
`app/build.gradle.kts` (version `0.11.0-phase11`), `README.md`,
`PHASE9.md` (secret-scan false-positive reword only).

**Modified (tests, ~14):** all fake DAOs updated to new signatures;
`UrlSafetyTest`, `CustomProviderConfigTest`, `PublishServiceTest`,
`PublishingToolsTest`, `ProductivityToolsTest`,
`ConfirmationEngineTest`, `DraftRepositoryTest`, `RoomAuditLogTest`,
`AnalyticsRepositoryTest`, `BackupServiceTest`, `TaskRepositoryTest`,
`NoteRepositoryTest`, `MemoryRepositoryTest`, `MacroRepositoryTest`,
`ResearchRepositoryTest`, `ScheduledPostRepositoryTest`,
`FocusSessionRepositoryTest`, `DraftComposerTest`,
`ResearchEngineTest` (fake-DAO compat only).

## 8. What remains for Phase 12 (release build)

- First real `./gradlew assembleDebug` + on-device test on
  Salahuddin's machine / Android Studio (standing item since Phase 1).
- Signed APK/AAB + release signing (manual: keystore via
  `scripts/make_keystore.sh`).
- First live OAuth connect + first real publish per platform.
- Play Console human steps in `PLAY_POLICY_CHECKLIST.md` §8
  (declarations, data-safety form, privacy-policy URL, content rating).
- Optional hardening noted but not taken: validate custom-provider
  configs in `SettingsViewModel.addCustomProvider` /
  `updateCustomProvider` before saving; Mutex serialization of
  `ConfirmationEngine.execute()`; per-turn tool-call cap in the
  planner; `WakeWordService` restart-coroutine cancellation.
