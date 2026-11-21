# Phase 9 — Productivity, proactive & wellness

Jarvis becomes useful day-to-day: tasks and reminders with plain-word
due dates, a read-only view of the calendar, a morning briefing, quiet
focus sessions, and wellness features (screen time, break reminders,
sleep wind-down) — all gentle, all on-device, and never nagging. Every
proactive feature is off by default or quiet by default, every one has
a clear off switch, and quiet hours (default 22:00–07:00) silence the
nudges. Anything that changes state still goes through the Confirmation
Engine first — the notification alone never changes a setting.

## What was built

### Tasks & reminders — the productivity core

`Task` (`:core/productivity/TaskModels.kt`) is a Room entity
(`:data/db/TaskEntity.kt`, Room **v7**, real `MIGRATION_6_7`): title,
notes, nullable due date/time, priority (Low / Medium / High), status
(OPEN / DONE), created/updated timestamps, optional reminder time, and
an `overdueNotifiedDay` (epoch day, UTC) for the smart-reminder dedupe.
`TaskRepository` + `TaskDao` support search/filter (open, done,
overdue, by priority, due today).

Tools (all through `GatedToolExecutor`, like everything else):

| Tool | Tier | What it does |
|---|---|---|
| `create_task` | 1 | Creates a task; due date parsed from plain words (see below). |
| `list_tasks` | 0 | Lists tasks (open / done / overdue / all, by priority). |
| `complete_task` | 1 | Marks a task done. |
| `delete_task` | 1 | Deletes a task (cancels its reminder first). |
| `set_reminder` | 1 | Attaches a reminder time; a WorkManager one-time worker fires a notification at that time (tap opens the task). |

The reminder worker re-reads the task before notifying and fires only
when the task still exists, is still OPEN, and still has *this*
reminder attached; the reminder is cleared after firing (fires once).
Re-scheduling cancels the old job first.

**Natural-language due dates** (`DueDateParser`, pure Kotlin, tested):
"tomorrow", "next Monday", "in 3 days", "Friday at 5pm", "today",
"tonight" (20:00), "tomorrow morning" (09:00), "tomorrow evening"
(19:00), "this evening" (19:00), "end of day" (23:59), "in N
minutes/hours/days/weeks", "next week", "next month", "<day> at <time>"
(5pm, 9:30am, 14:00, hour-only), ISO dates with optional time. Bare
day references default to end of day (23:59); weekday names resolve to
the next occurrence strictly after today (if today is Monday, "monday"
is next Monday). Anything unparseable returns null — the planner asks
Zaid, the editor shows a hint — never guessed.

**Tasks screen**: list with filters (Open / Done / Overdue / All),
add/edit/complete/delete, due-date display with overdue highlighting
in red, priority chips. The editor parses due dates and reminders live
and previews the resolved date under each field. **Home** shows a
"Today" card: tasks due today + overdue count + today's calendar
events.

### Calendar awareness — read-only

`CalendarContractReader` (`:app/productivity/`) reads the device
calendar via CalendarContract. `READ_CALENDAR` is asked only when
first used (Home schedule, `list_events`, the briefing) — never at app
start — and the calendar is read-only: **no event creation in Phase 9**
(a documented future addition). `list_events` (Tier 0) lists events
today/tomorrow with time, title, and location.

### Daily briefing — never fails

`BriefingService` (`:agent/productivity/`) assembles today's events +
tasks due today + overdue tasks (`BriefingData` /
`BriefingFormat.formatPlain`, `:core/productivity/BriefingModels.kt`)
and, when an LLM is configured and reachable, polishes it into natural
language via the existing `LlmProviderClient` (usage recorded through
`UsageTracker`); offline or unconfigured, it falls back to the plain
templated summary. Even a throwing task store returns an honest
"couldn't load your day" message — the briefing never fails. Askable
in chat ("Morning briefing" via the `morning_briefing` tool, Tier 0)
or as the opt-in daily digest notification. **No weather** — there is
no weather API in Phase 9 (documented in the limitations below).

### Focus mode — WorkManager-driven, DND restored

`start_focus` / `stop_focus` (Tier 1) run a focus session: 25/50/90
minute presets + custom (5–240 min). Starting enables Do Not Disturb
via the existing Phase 5 DND path (mode ALARMS_ONLY — alarms still
ring) after the Tier 1 approval card, starts the foreground
`FocusSessionService` (persistent notification with time remaining,
ticking every 30 s), and enqueues a `FocusEndWorker` for the end time
as the backstop if the service is killed. Ending restores DND to its
previous state and notifies. Ending is idempotent
(`FocusSessionRepository.finish()` — only the first finisher restores
DND / notifies), and if saving the session fails the DND change is
rolled back. Sessions are logged in Room (same v7 migration:
`focus_sessions` — start, duration, status RUNNING / COMPLETED /
CANCELLED) and shown in a history list on the Wellness screen.

### Smart reminders, daily digest, quiet hours

- **Smart reminders**: when a task becomes overdue, one gentle
  notification — once per task per day, never repeated (the
  `overdueNotifiedDay` dedupe, tested). The notification is a single
  summary (up to 3 named + count), capped per run. On by default, but
  quiet by design — with a clear off switch.
- **Daily digest** (opt-in, off by default, time configurable in
  Settings, default 08:00): a 24-hour WorkManager periodic worker
  (`DailyDigestWorker`) posts the morning briefing as a notification.
  Re-synced with `ExistingPeriodicWorkPolicy.UPDATE` so a changed
  digest time re-aligns the next run.
- **Quiet hours** (default 22:00–07:00, start == end disables):
  `QuietHours` (`:core/productivity/`) is the one shared pure function
  every Phase 9 notifier uses — wrap-around midnight handled. During
  quiet hours NO proactive notification is posted, except:
  - user-set task reminders (explicit requests, like an alarm),
  - focus-session results (explicit user requests),
  - the wind-down nudge (exempt by design — bedtime default 23:00 is
    normally *inside* quiet hours; Zaid opted in and chose the time),
  - Phase 8 publish/schedule results (unchanged).
  Overdue nudges, the digest, and break reminders all respect quiet
  hours.

The proactive checks (overdue / break / wind-down) run in one 15-minute
periodic worker (`ProactiveCheckWorker`), enqueued only when at least
one of the three features is enabled. `WellnessScheduler` keeps both
periodic jobs in sync with Settings — called from
`JarvisApplication.onCreate` (a force-stop or update can drop jobs)
and from the settings ViewModels whenever a toggle or time changes.

### Wellness — all on-device, all opt-in

- **Screen-time awareness**: `UsageStatsScreenTimeReader`
  (`:app/productivity/`) reads UsageStats (PACKAGE_USAGE_STATS —
  granted only in system Settings via the button in Settings →
  Productivity & wellness or on the Wellness screen; the feature is
  disabled until granted, with a clear explanation). Today's total
  screen time + top 5 apps on the Wellness screen, and a
  `get_screen_time` tool (Tier 0). Aggregation is pure Kotlin
  (`ScreenTime.aggregate`, tested with fakes). Data stays on-device —
  never stored, uploaded, or logged.
- **Break reminders** (opt-in, off by default, configurable interval,
  default 60 min of continuous screen time): checked by the periodic
  worker using UsageEvents — at most once per hour (`ProactiveRules`,
  tested), never during quiet hours, never while a focus session is
  running. Continuous time is computed with a 5-minute gap tolerance
  (a quick app switch doesn't break a session; putting the phone down
  does). UsageStats can lag, so reminders only ever fire late, never
  early.
- **Sleep wind-down** (opt-in, off by default, configurable bedtime,
  default 23:00): at bedtime a notification suggests winding down.
  Its "Wind down now" action opens the Wellness screen, where the DND
  + dim action goes through the normal gated path (in-app button →
  `GatedToolExecutor`: `set_do_not_disturb` alarms_only + `set_brightness`
  20%, Tier 1 card each). **The notification alone never changes any
  setting by itself.**

**Wellness screen**: today's screen time, top apps, focus-session
history, break/wind-down toggles + intervals, quiet-hours editor
(TimePicker dialogs), usage-access status + button, "Wind down now".

### Settings — "Productivity & wellness" section

Daily digest toggle + time, quiet hours start/end, break-reminder
toggle + interval, wind-down toggle + bedtime, usage-access status +
button, calendar permission status + button — plus links to the Tasks
and Wellness screens. New `JarvisSettings` fields (all with defaults):
`dailyDigestEnabled=false`, `dailyDigestHour=8`,
`dailyDigestMinute=0`, `quietHoursStartMinute=1320`,
`quietHoursEndMinute=420`, `overdueRemindersEnabled=true` (on by
default but quiet by design), `breakRemindersEnabled=false`,
`breakReminderIntervalMinutes=60`, `windDownEnabled=false`,
`windDownHour=23`, `windDownMinute=0`.

### Planner — rule (9)

The system prompt gains rule (9): tasks / reminders / focus /
screen-time capabilities; proactive features are quiet and off by
default; never nag; never change a setting without approval.

### Tool table (10 new; 45 total)

| Tool | Tier | Module |
|---|---|---|
| `create_task` | 1 | `:tools/productivity/` |
| `list_tasks` | 0 | `:tools/productivity/` |
| `complete_task` | 1 | `:tools/productivity/` |
| `delete_task` | 1 | `:tools/productivity/` |
| `set_reminder` | 1 | `:tools/productivity/` |
| `list_events` | 0 | `:tools/productivity/` |
| `morning_briefing` | 0 | `:tools/productivity/` |
| `start_focus` | 1 | `:tools/productivity/` |
| `stop_focus` | 1 | `:tools/productivity/` |
| `get_screen_time` | 0 | `:tools/productivity/` |

`:tools` stays pure Kotlin behind interfaces (`TaskStore`,
`ReminderScheduler`, `CalendarReader`, `ScreenTimeReader`,
`FocusSessionManager`, `BriefingProvider` — with NoOp implementations
for tests); the Android implementations live in `:app/productivity/`
and are wired in `AppModule`.

### Screens & navigation

- **Tasks screen** (`TasksScreen` + `TasksViewModel`): see above.
- **Wellness screen** (`WellnessScreen` + `WellnessViewModel`): see
  above. Focus start/stop and "Wind down now" go through
  `GatedToolExecutor`, so approval cards appear for each change.
- **Home** (`HomeScreen` + `HomeViewModel`): "Today" card with tasks
  due today / overdue + today's schedule (or a hint to grant calendar
  access). "Daily briefing" chip now opens Chat; "Set a reminder" chip
  opens Tasks; new Tasks and Wellness chips.
- **Navigation**: `TasksDestination` and `WellnessDestination`
  (reachable from Home and Settings, not the bottom bar); back button
  shows on both. `MainActivity` handles `DESTINATION_TASKS` /
  `DESTINATION_WELLNESS` deep-links (cold start via startDestination,
  warm launch via `VoiceStateStore` counters) so notification taps
  open the right screen.
- **Permissions**: `READ_CALENDAR` added to
  `PermissionHelper.TOOL_PERMISSIONS` (so it appears in Settings → App
  permissions) + usage-access helpers (`hasUsageAccess`,
  `usageAccessSettingsIntent`).
- **Manifest**: `READ_CALENDAR`, `PACKAGE_USAGE_STATS`,
  `FocusSessionService` entry (`foregroundServiceType="specialUse"` +
  `PROPERTY_SPECIAL_USE_FGS_SUBTYPE`; FOREGROUND_SERVICE +
  FOREGROUND_SERVICE_SPECIAL_USE already declared).

### Files

New files (41, incl. tests):

- `:core` main (`core/productivity/`): `TaskModels.kt`,
  `DueDateParser.kt`, `QuietHours.kt`, `ProactiveRules.kt`,
  `ScreenTime.kt`, `BriefingModels.kt`
- `:core` tests (`core/productivity/`): `DueDateParserTest.kt`,
  `QuietHoursTest.kt`, `ProactiveRulesTest.kt`, `ScreenTimeTest.kt`,
  `TaskModelsTest.kt` (contains `TaskModelsTest` + `BriefingFormatTest`)
- `:data`: `db/TaskEntity.kt`, `db/FocusSessionEntity.kt`,
  `repository/TaskRepository.kt`,
  `repository/FocusSessionRepository.kt`,
  `datastore/ProactiveStateRepository.kt`; tests
  `repository/TaskRepositoryTest.kt`,
  `repository/FocusSessionRepositoryTest.kt`
- `:tools`: `tools/productivity/Productivity.kt` (interfaces + NoOps),
  `tools/productivity/ProductivityTools.kt` (10 tools); test
  `tools/productivity/ProductivityToolsTest.kt`
- `:agent`: `agent/productivity/BriefingService.kt`; test
  `agent/productivity/BriefingServiceTest.kt`
- `:app` (`app/productivity/`): `RoomTaskStore.kt`,
  `CalendarContractReader.kt`, `UsageStatsScreenTimeReader.kt`,
  `WellnessNotifier.kt`, `WorkManagerReminderScheduler.kt`,
  `TaskReminderWorker.kt`, `FocusSessionManagerImpl.kt`,
  `FocusSessionService.kt`, `FocusEndWorker.kt`,
  `DailyDigestWorker.kt`, `ProactiveCheckWorker.kt`,
  `WellnessScheduler.kt`
- `:app` UI: `components/TimePickerDialog.kt`,
  `screens/TasksScreen.kt`, `screens/WellnessScreen.kt`,
  `viewmodel/TasksViewModel.kt`, `viewmodel/WellnessViewModel.kt`,
  `viewmodel/HomeViewModel.kt`

Edited: `core/model/JarvisSettings.kt` (Phase 9 fields),
`data/datastore/SettingsRepository.kt` (persistence),
`data/db/JarvisDatabase.kt` (v7 + `MIGRATION_6_7`),
`data/di/DataModule.kt` (DAO + repository providers),
`tools/ToolsModule.kt` (registration), `agent/di/AgentModule.kt`
(`provideBriefingProvider`), `agent/Planner.kt` (rule 9),
`app/di/AppModule.kt` (providers),
`app/JarvisApplication.kt` (sync periodic work on start),
`app/MainActivity.kt` (deep-links),
`app/navigation/JarvisNavGraph.kt` (destinations),
`app/voice/VoiceStateStore.kt` (counters),
`app/permissions/PermissionHelper.kt` (calendar + usage access),
`app/viewmodel/SettingsViewModel.kt` (Phase 9 setters +
`WellnessScheduler.sync()`), `app/screens/HomeScreen.kt`,
`app/screens/SettingsScreen.kt`, `app/AndroidManifest.xml`,
`app/build.gradle.kts` (version `0.9.0-phase9`).

## Verification

Command: `bash ~/opt/verify-pom/verify-phase9.sh` (kotlinc 2.4.21 +
JUnitCore — `./gradlew` still cannot run in this VM; the script tries
it once briefly, as in Phases 1–8). No new verification deps needed
(WorkManager/Room already fetched; Calendar/UsageStats are android.jar
APIs).

- **560 unit tests pass, 0 failures** (475 in Phase 8 + 85 new):
  - `:core` 223 (44 new: DueDateParser — all supported phrases +
    unparseable input; QuietHours — incl. wrap-around midnight and
    start == end disabled; ProactiveRules — overdue dedupe, break
    throttling, wind-down once-per-day, next-daily-time; ScreenTime —
    aggregation incl. top-5 + tie-break, continuous-session incl.
    gap tolerance; TaskModels — overdue/due-today; BriefingFormat —
    plain template assembly)
  - `:tools` 166 (19 new: create/list/complete/delete task,
    set-reminder validation, list-events incl. no-access, morning
    briefing, start/stop focus validation, get-screen-time incl.
    no-access; registry count 45)
  - `:agent` 108 (6 new: BriefingService — LLM polish + plain
    fallback, usage recorded, never fails on a throwing store)
  - `:data` 63 (16 new: task repository with fake DAO — filters,
    overdue, reminder attach/clear, overdue-notified marking;
    focus-session repository — running session, idempotent finish,
    history)
- **All Android sources compile via kotlinc** (Compose +
  serialization plugins, android.jar API 36): **989 class files,
  0 errors**; only the 2 pre-existing Phase 1 deprecation warnings
  remain.
- Secret scan clean; no real network calls in any test (recorded
  fixtures / fakes only); no hard-coded keys or secrets.

Bugs found and fixed during verification:
- `ProductivityToolsTest` passed a `now` parameter to
  `ListEventsTool` — the tool's clock parameter is named `nowMillis`
  (same as `ListTasksTool`). Fixed the three call sites.
- `TasksScreen` smart-cast errors: `task.dueAtMillis` /
  `task.reminderAtMillis` are public properties declared in another
  module, so the compiler can't smart-cast them after a null check.
  Captured them into locals first (same class of fix as Phase 8's
  cross-module smart casts).
- `WellnessScreen` used `ScreenTime.formatDuration` and
  `UsageEntry.label` / `totalMillis` — the real API is the top-level
  `formatDuration()` function, and `UsageEntry` carries `packageName`
  + `totalTimeInForegroundMillis`. Fixed; app labels are now resolved
  via PackageManager in the screen (falling back to the last package
  segment).
- `BriefingServiceTest` used `UsageRecorder` without importing it
  (it lives in `com.zaid.jarvis.agent`, the test in
  `com.zaid.jarvis.agent.productivity`). Added the import.
- `HomeScreen` had an unnecessary safe call (`event.location` is a
  non-null String) — replaced with an `isNotBlank()` check.
- `UsageEvents.Event.MOVE_TO_FOREGROUND` /
  `MOVE_TO_BACKGROUND` are deprecated on API 34+ (in favour of
  `ACTIVITY_RESUMED` / `ACTIVITY_PAUSED`, which only exist on API
  29+) — kept with a documented `@Suppress("DEPRECATION")`, since
  they are the only events on API 26–28 (minSdk 26) and are still
  delivered on API 36 (same pattern as Phase 6's `recycle()` /
  `obtain()` suppressions).
- Secret scan false positive: a comment containing the words "task"
  and "reminder" joined by a hyphen produced a string matching the
  `sk-` API-key pattern. Reworded the comment.

## What was NOT verified

- The real Gradle build (`./gradlew assembleDebug`) — cannot run in
  this VM (same as Phases 1–8). Hilt/Room codegen (incl. the v7
  migration), resource merging, and the manifest are therefore
  unverified until the first real build on Salahuddin's machine /
  Android Studio.
- All on-device behaviour: task reminders firing, focus sessions
  (DND change/restore, service countdown, end worker), digest and
  proactive notifications, quiet-hours gating in practice, UsageStats
  reads, CalendarContract reads, wind-down flow, permission flows.
- WorkManager timing is best-effort under Doze/battery optimization
  (see limitations).

## Setup notes

Permissions Zaid grants, where, and why:

| Permission / access | Where | Why |
|---|---|---|
| Calendar (`READ_CALENDAR`) | Asked when Home/briefing/`list_events` first needs it, or Settings → Productivity & wellness / App permissions | Read today's schedule (read-only). Without it, Home shows a hint and `list_events` says so. |
| Usage access (`PACKAGE_USAGE_STATS`) | Settings → Productivity & wellness → Usage access, or the Wellness screen button (opens system Settings) | Read today's screen time + top apps. Android requires a manual grant; without it the feature is disabled, not broken. |
| Do Not Disturb access | Settings → App permissions (already there since Phase 5) | Focus mode and wind-down change DND mode. Without it, starting focus fails with a clear message. |
| Modify system settings | Settings → App permissions (Phase 5) | Wind-down dims the screen (brightness). |
| Notifications (`POST_NOTIFICATIONS`) | System permission dialog (API 33+) | Reminders, digest, focus results, break/wind-down nudges. |

Nothing else to set up: tasks, focus, and the briefing work with no
account and no key. The digest needs an LLM key only for the polished
version — without one it posts the plain summary.

## Honest limitations

- **WorkManager timing is best-effort**: under Doze and battery
  optimization, the digest, proactive checks, and reminders may run
  late (never early). The 15-minute proactive check is WorkManager's
  minimum interval, so break reminders and overdue nudges can lag by
  up to ~15 minutes plus Doze delays.
- **UsageStats requires a manual grant** and can lag a few minutes
  behind real usage; the "continuous session" length is computed from
  UsageEvents, which the system may coalesce — break reminders only
  ever fire late, never early.
- **Calendar is read-only**: no event creation in Phase 9 (a future
  addition — needs a write permission and a gated Tier 1 tool).
- **No weather**: the morning briefing has no weather section — there
  is no weather API in Phase 9.
- **Focus DND**: the previous DND mode is captured at session start;
  if Zaid changes DND manually mid-session, the restore puts back the
  captured mode, not the manual one. (Edge case, documented here.)
- **Room v7 migration untested** (schema mismatch would crash on
  start — check logcat on the first real run, as in prior phases).
- First real on-device test on Salahuddin's machine / Android Studio
  (all phases).

## What remains

- **Phase 10**: Plugin system + backup + analytics (plugins must
  route through `GatedToolExecutor`).
- **Phase 11**: Hardening — security review, performance, Play-policy
  checklist (Accessibility prominent disclosure already in place
  since Phase 6; DNS-rebinding SSRF hardening from Phase 7).
- **Phase 12**: Release build — signed APK/AAB (needs manual steps
  from Salahuddin: enable Accessibility, create social developer
  accounts, paste API keys/OAuth credentials).
- First real `./gradlew assembleDebug` + on-device test still pending
  on his machine / Android Studio (all phases), plus the first live
  OAuth connect and first real publish per platform.
