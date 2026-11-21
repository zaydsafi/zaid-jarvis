# Zaid Jarvis

A professional, advanced, voice-first AI assistant for Android — built for its
owner, Zaid. Jarvis understands natural-language requests, plans multi-step
tasks, controls the phone and other apps, researches topics, writes posts, and
publishes them to social media — **only after Zaid explicitly approves each
operation**.

> **Status: Phase 11 of 12 complete — hardening.**
> Jarvis has been through a full security audit (DNS-rebinding SSRF guard,
> network security config with cleartext off, a publishing-gate bypass
> fixed, OAuth ordering fixed, secret audit clean), a performance pass
> (bounded list queries everywhere, weekly maintenance worker pruning
> analytics older than 90 days and audit entries beyond 10,000, a full
> "Delete all data" flow), and a Play-policy checklist with a
> plain-language privacy policy — see [PHASE11.md](PHASE11.md),
> [PLAY_POLICY_CHECKLIST.md](PLAY_POLICY_CHECKLIST.md), and
> [PRIVACY_POLICY.md](PRIVACY_POLICY.md).
> Earlier notes: [PHASE1.md](PHASE1.md), [PHASE2.md](PHASE2.md),
> [PHASE3.md](PHASE3.md), [PHASE4.md](PHASE4.md), [PHASE5.md](PHASE5.md),
> [PHASE6.md](PHASE6.md), [PHASE7.md](PHASE7.md), [PHASE8.md](PHASE8.md),
> [PHASE9.md](PHASE9.md), [PHASE10.md](PHASE10.md).

## Non-negotiable rules (from the master prompt)

1. **Permission-first** — no operation without Zaid's approval (Confirmation
   Engine, ✅ Phase 2).
2. **Publishing gate** — research → draft → preview → Allow → publish → report.
3. **Transparency** — no hidden or silent background actions.
4. **Kill switch** — one tap (or "Jarvis stop") cancels everything (✅ Phase 2;
   voice command ✅ Phase 4).
5. **Least privilege** — each Android permission requested only when needed.
6. **Honesty** — never claims success without verifying it.
7. **Privacy** — sensitive data stays on-device by default.

## Tech stack

- Kotlin 2.3.21, Jetpack Compose (Material 3), single activity, Navigation Compose
- MVVM + Clean Architecture, multi-module (`app`, `core`, `agent`, `tools`,
  `data`, `ui`, `plugins`)
- Hilt, Coroutines/Flow, Room + SQLCipher (encrypted), DataStore, WorkManager,
  Retrofit/OkHttp, Kotlin Serialization
- LLM providers (selectable in Settings): **Claude** (default, Sonnet 5.5 /
  Haiku 5.5), **OpenAI** (GPT-5 / GPT-5 mini), **Gemini** (Gemini 3 Pro /
  Flash), **OpenRouter** (any model ID, e.g. `anthropic/claude-sonnet-4.5`),
  and **custom OpenAI-compatible providers** (name + base URL + API key +
  model ID — local servers, other vendors, corporate proxies)
- Min SDK 26, target SDK 36 (Android 16), compile SDK 36, R8 for release

## Modules

| Module | Contents |
|---|---|
| `app` | Single activity, navigation graph, Home/Chat/Settings/Audit Log/**Memory**/**Macros**/**Research**/**Drafts**/**Connected accounts**/**Scheduled posts**/**Tasks**/**Wellness** screens, ViewModels, biometric auth, approval host, **voice engine, overlay bubble, widget, Quick Settings tile, shared chat session**, **Android phone-device implementation, permission helper**, **AccessibilityService + macro recorder**, **OAuth redirect activity + Custom Tabs connect flow, WorkManager scheduler, publish notifications**, **productivity implementations (tasks, calendar, UsageStats, focus service, reminder/digest/proactive workers, wellness notifier/scheduler)** |
| `core` | Pure-Kotlin models (`LlmProvider`, `JarvisSettings`, …), `JarvisResult`, dispatchers, **Confirmation Engine, KillSwitch, audit-log + trusted-action contracts**, **pricing table, custom-provider config, context snapshot**, **voice logic (wake-word matcher, stop-command parser, TTS sanitizer, overlay state machine)**, **undo info on audit entries**, **automation model (screen snapshots, selectors, macros, blocklist)**, **research models (search results, briefs, drafts, SSRF guard, HTML extractor, draft limits)**, **social models (platforms, OAuth tokens, scheduled posts, PKCE, schedule validation)**, **productivity models (tasks, focus sessions, due-date parser, quiet hours, proactive rules, screen time, briefing models)** |
| `data` | DataStore settings, encrypted Room DB (SQLCipher, incl. audit log), Keystore-backed API-key storage, trusted-action store, **MemoryRepository, DeviceContextProvider, LastAnswerRepository**, **MacroRepository**, **ResearchRepository, DraftRepository**, **ScheduledPostRepository, SocialAccountRepository**, **TaskRepository, FocusSessionRepository, ProactiveStateRepository** |
| `ui` | Material 3 theme (Light / Dark / AMOLED), animated Jarvis orb, **ApprovalCard** |
| `agent` | **LLM clients (Claude, OpenAI, Gemini, OpenRouter, custom), planner loop, injection guard, response cache, usage tracker, conversation context**, **research engine + draft composer, DuckDuckGo/Tavily search, page fetcher**, **social platform clients (X, Threads, LinkedIn, Instagram), OAuth client, token refresh, publish gate**, **briefing service (LLM polish + plain fallback)** |
| `tools` | Tool contract + risk tiers, **GatedToolExecutor (the only way to run a tool)**, **tool JSON schemas, built-in tools (time, calculator, echo)**, **phone tools (tiers 0–2), device abstraction, contact resolver, undo contract**, **automation tools + macro player**, **research tools (search/fetch/research/draft)**, **social publishing tools (publish/schedule/list/cancel)**, **productivity tools (tasks, reminders, calendar, briefing, focus, screen time)** |
| `plugins` | **Phase 1 stub** — plugin contract (plugin system in Phase 10) |

## Confirmation Engine (Phase 2)

Every tool call passes through the Confirmation Engine — tools are never
executed directly, only via `GatedToolExecutor`. Risk tiers (Master Prompt §6):

| Tier | Examples | Approval |
|---|---|---|
| 0 Read-only | time, weather, search | Ask once per session, then allow (or always ask, in Settings) |
| 1 Reversible | toggles, open app, set alarm | Approval card each time |
| 2 External / visible | SMS, call, post, email, delete | Approval card + second confirm |
| 3 Sensitive | payments, banking, passwords | Approval card + biometric (or device PIN) |

Approvals expire after **2 minutes**, are bound to the exact content shown
(SHA-256), and can never be reused. "Always ask" is the default; low-risk
actions (tiers 0–1) can be trusted from their approval card and revoked
any time in Settings → Permissions & trust. Every request and outcome is
recorded in the encrypted audit log (Home → Audit log), with search, CSV
export, and real undo for reversible actions (Phase 5): the Undo button
runs the tool's inverse through the Confirmation Engine and marks the
entry undone only after the inverse succeeds. The kill switch (red stop
button in the top bar, on Home, and the voice command "Jarvis stop")
cancels all running and queued automation immediately.

## Agent core (Phase 3)

The planner (`AgentPlanner`) runs one user request through a loop: the LLM
receives the request, the available tools (with JSON schemas), the current
context (date/time, battery, network), and a short summary of long-term
memory — then either answers, asks a clarifying question, or calls a tool.
**Every tool call routes through the Phase 2 `GatedToolExecutor`** — the
planner never executes a tool directly.

- **Providers**: Claude, OpenAI, Gemini, OpenRouter, and custom
  OpenAI-compatible providers. Simple intents (short, plain questions) use
  the cheaper model of the selected provider when enabled in Settings.
- **Tools in Phase 3** (all Tier 0, no permissions, offline-safe): `get_time`,
  `calculator` (hand-written parser — arithmetic only, never a script engine),
  `echo` (dry-run demo). Phone-control tools landed in Phase 5 (see below).
- **Memory**: short-term conversation context (last 20 messages) + long-term
  memory in the encrypted Room DB. Zaid can view, edit, and delete any memory
  item in Settings → View memory. The planner only sees a short summary.
- **Cost**: real token counts from API usage responses, persisted in Settings →
  Usage, with an approximate per-model pricing table (clearly marked as an
  estimate; unknown/custom models use a fallback rate). Identical pure-text
  prompts are served from an in-memory response cache (15-minute TTL, never
  cached when a tool is involved).
- **Offline**: with no network, the LLM can't be reached — but time,
  calculator, battery, and network-status commands still work through
  the same Confirmation Engine. Anything else gets an honest "you're
  offline" message.
- **Prompt-injection defense**: tool results are data, never instructions.
  The `InjectionGuard` scans every tool result for instruction-like patterns
  (override attempts, secret extraction, exfiltration, money moves, jailbreaks)
  and, when it finds one, the planner stops and asks Zaid instead of acting.
- **Retries**: at most 3 attempts per LLM call and per tool step, with
  500ms / 1s / 2s backoff. Denials, expired approvals, auth errors (401/403),
  unknown tools, and invalid parameters are never retried.
- **Kill switch**: the planner checks the Phase 2 kill switch every turn —
  "Jarvis stop" or the stop button cancels a run mid-loop.

## Voice & UI (Phase 4)

- **Voice input**: tap the mic in Chat (or the Home orb, the widget mic,
  or the Quick Settings tile). SpeechRecognizer with live partial
  transcripts, in the language from Settings (English / Urdu / Arabic).
  Microphone permission is asked only on the first mic tap, never at
  app start.
- **Voice replies**: after a voice request, Jarvis reads its answer aloud
  (TextToSpeech at the Settings speed; markdown/links/emoji are stripped
  before speaking). Toggle in Settings → Voice. Every Jarvis message
  also has a speak button.
- **"Jarvis stop"**: saying it — even mid-sentence, even while Jarvis is
  speaking — cancels the run and silences TTS immediately.
- **Wake word**: "Hey Jarvis" listening is a foreground service, **off
  by default** (Settings → Voice). Honest limit: Android has no public
  always-on hotword API for third-party apps, so this loops the
  platform recognizer — it holds the mic continuously and uses more
  battery than a true keyword spotter would.
- **Chat**: timestamps, tool-progress lines ("Running get_time…"),
  suggested prompts, auto-scroll. The chat screen and the overlay
  bubble share one conversation session.
- **Overlay bubble**: a floating orb over other apps (needs "Display
  over other apps", granted in system Settings). Tap to open a mini
  chat, drag to move, long-press to dismiss. Approval cards appear
  inside the bubble; second confirmations and biometric checks open
  the app.
- **Widget**: home-screen widget with the orb, a mic button, and the
  latest Jarvis answer. Add it from the launcher widget picker.
- **Quick Settings tile**: "Talk to Jarvis" — tap to open Chat and
  start listening; the tile shows ACTIVE while the mic is live. Add
  it from the Quick Settings edit panel.

## Phone tools (Phase 5)

20 tools in total (3 pure tools from Phase 3 + 17 phone tools), all
gated by the Confirmation Engine. Ask Jarvis in chat or by voice —
"turn the flashlight on", "set an alarm for 7:30", "text Alice I'll be
late" — and approve the card it shows.

- **Tier 0 (read-only)**: battery level, network type/status, contact
  search, installed-apps list.
- **Tier 1 (reversible)**: flashlight, volume (media/alarm/ring/
  notification/call/system), brightness, Wi-Fi, Bluetooth, alarms,
  timers, opening apps, Do Not Disturb. Every one records the previous
  device state, so **Undo in the Audit Log really reverses it** (and
  the undo itself shows its own approval card first).
- **Tier 2 (external/visible)**: calls, texts, emails, sharing. These
  are **intent-only**: Jarvis opens the dialer / messaging app / email
  app with everything pre-filled, and you tap Call or Send yourself.
  Jarvis never dials or sends on its own — the app deliberately does
  not declare CALL_PHONE or SEND_SMS.
- **Honest limits**: Android 10+ won't let apps toggle Wi-Fi directly,
  so Jarvis opens the Wi-Fi settings panel instead; Android 13+ shows
  its own dialog when turning Bluetooth on; Android has no way to
  cancel an alarm or timer programmatically, so alarm/timer undo opens
  the Clock app for you to delete it there. Brightness needs "Modify
  system settings" and DND needs Do Not Disturb access — both granted
  in system Settings.
- **Permissions**: each permission is requested only when its tool is
  first used, never at app start. Grant any of them ahead of time in
  Settings → App permissions, which also lists the special accesses
  (Modify system settings, Do Not Disturb access, Display over other
  apps) with their grant status.
- **Edited previews**: if you edit an SMS or email on the approval
  card, the edited text is what actually runs — the audit log records
  the edited parameters too. A tool that can't make sense of an edit
  refuses to run rather than running something you didn't approve.

## Accessibility automation & macros (Phase 6)

27 tools in total now (20 from Phases 3–5 + 7 automation tools). Ask
Jarvis in chat or by voice — "open WhatsApp and tap the search icon",
"scroll down", "run my morning news macro" — and approve the card.

- **Setup (two steps, both by hand)**: first enable "Zaid Jarvis
  automation" in Android Settings → Accessibility (Jarvis cannot
  enable it itself — Android forbids it); then turn Automation on in
  Jarvis Settings → Automation. The toggle stays greyed out until the
  service is enabled, and automation is off by default.
- **Tools**: `find_on_screen` (Tier 0 — describes the tappable elements
  on screen), `tap_on_screen`, `type_text`, `scroll_screen`, `press_back`,
  `press_home` (Tier 1), and `run_macro` (Tier 1 for scroll/navigate-only
  macros, Tier 2 — card + second confirm — for macros that tap or type).
- **Safety**: Jarvis **never reads or stores passwords** — password
  fields are invisible to it and it refuses to act on any screen that
  has one. It **never automates banking or payment apps** (a blocklist
  of banking/payment packages, plus the password-field rule as the
  backstop). Screen text is treated as data: the planner's injection
  guard scans it before acting on it.
- **Macros**: record your own taps, typing, and scrolling from the
  Macros screen (Home → Macros), name the recording, and replay it
  later — one approval card lists every step. Playback verifies each
  step, stops at the first failure and tells you exactly which step
  failed and what was on screen instead, and the kill switch stops it
  immediately. Macros are capped at 50 steps with a 10-second timeout
  per step.
- **Honest limits**: automation is inherently fragile — if an app
  updates its layout, taps miss and macros stop at the broken step.
  Nothing here has been tested on a real device yet (no emulator in
  this environment), so the first on-device run is where the service
  connection, gestures, and recording get proven. Play Store policy
  requires the prominent Accessibility disclosure Jarvis already shows
  when you enable the service; the full policy checklist is Phase 11.

## Research & content engine (Phase 7)

31 tools in total now (27 from Phases 3–6 + 4 research tools). This is the
research → draft → preview half of the publishing gate. Ask Jarvis in chat
or by voice — "research quantum computing", "draft a post about my new
collection for Instagram" — and approve the card.

- **Tools**: `search_web` (Tier 0 — web search), `fetch_page` (Tier 0 —
  read one page as text), `research_topic` (Tier 0 — the full pipeline:
  search 1–3 queries, read up to 5 pages, synthesize a structured brief
  with every factual claim cited to its source), `draft_post` (Tier 1 —
  generates a draft and saves it for review; the approval card shows the
  topic, platform, and tone — not the generated text, which doesn't
  exist until after approval).
- **Search backends** (Settings → Research): **DuckDuckGo** is the
  default — free, no key, via the lite HTML endpoint. **Tavily** is the
  optional premium backend (built for AI research, one JSON call,
  generous free tier) — paste a key from <https://app.tavily.com> in
  Settings → Research. Keys are stored encrypted like all other keys.
- **Safety**: web content is untrusted data. Every fetched page is
  scanned by the Phase 3 InjectionGuard *before* it reaches the
  synthesis prompt — a page with instruction-like text is excluded and
  the brief says so. Search snippets are never fed to the LLM. Facts
  the engine cannot bind to a fetched source are dropped, never kept
  unsourced. `fetch_page` refuses non-http(s) URLs, localhost, and
  private/link-local IPs (SSRF guard), refuses binary content, and caps
  downloads at 200 KB.
- **Research**: jobs report progress (searching → fetching →
  synthesizing), are cancellable via the kill switch, and the finished
  brief is saved in the encrypted Room DB (Room v5). Research History
  (Home → Research) lists past briefs with view/delete.
- **Drafts**: each draft has a title, body, platform (X / Twitter 280,
  Threads 500, Instagram 2200, LinkedIn 3000, Blog 10000, Other 2000
  characters), status (DRAFT / APPROVED / PUBLISHED), and an optional
  link to the research it came from. The LLM is asked to stay under the
  limit; if it doesn't, Jarvis asks for one rewrite, then hard-truncates
  (flagged, never silent). The Drafts screen lists, filters, edits, and
  deletes drafts; the editor shows the character count against the
  limit and has an **Approve for publishing** button — a deliberate tap,
  never automatic. Phase 8 publishes APPROVED drafts (see below); until
  then PUBLISHED was refused by the repository.
- **Honest limits**: search quality depends on the backend —
  DuckDuckGo's lite endpoint is a free HTML scrape and can be thin or
  slow; Tavily is the better backend when it matters. No real search,
  fetch, or LLM call has been made in tests — everything was verified
  against recorded fixtures and fakes, so the first live research run
  on a real device is where the backends get proven. The SSRF guard
  blocks literal private IPs and, since Phase 11, hostnames that
  *resolve* to one — a custom OkHttp `Dns` validates every resolved
  address before connecting (DNS-rebinding hardening, see
  [PHASE11.md](PHASE11.md) §1.1).

## Social publishing (Phase 8)

35 tools in total now (31 from Phases 3–7 + 4 publishing tools). This is
the publish → report half of the publishing gate: research → draft →
preview → Allow → publish → report. Ask Jarvis in chat or by voice —
"publish my approved draft about the new collection", "schedule it for
tomorrow morning" — and approve the card. Or publish/schedule from the
draft editor itself.

- **Platforms** (official APIs only — no scraping, no automation of
  social apps): **X** (POST /2/tweets, text only), **Threads** (two-step
  container flow: create container, then publish it), **LinkedIn** (the
  current Posts API, POST /v2/posts), **Instagram** (Instagram Graph API
  container + publish flow; requires a Business/Creator account linked
  to a Facebook Page, and every post needs a photo — a public image URL
  is required at publish time). Each client enforces its platform's
  character limit again at publish time and returns the post URL +
  platform post id.
- **OAuth**: Authorization Code + PKCE (S256), implemented manually
  with Custom Tabs (no AppAuth dependency). Sign-in happens on the
  platform's own page in the browser — Jarvis never sees your platform
  password. Tokens (access + refresh) are stored Keystore-encrypted,
  never in Room, never logged. A 401 refreshes the token once and
  retries; if the refresh fails, the account is marked disconnected and
  you're asked to reconnect. Threads and Instagram tokens are exchanged
  for 60-day long-lived tokens at connect time, so scheduled posts keep
  working. **Connected accounts** (Settings → Social publishing) shows
  per-platform status, handle, and token expiry, and is where you paste
  each platform's OAuth client ID/secret — see [PHASE8.md](PHASE8.md)
  for the step-by-step setup for each platform.
- **Publishing gate** (enforced in code, not just in the UI): only
  APPROVED drafts can be published — the publish path refuses DRAFT
  outright. Publishing needs a fresh Tier 2 approval card (card + second
  confirmation) showing the exact text, platform, and account. If a
  draft was edited after approval, the publish is refused until the
  edited text is approved again (SHA-256 content binding — the approval
  records the body hash). After the platform confirms the post, the
  draft becomes PUBLISHED — `markPublished()` is the only place
  PUBLISHED may be set — and the publish report (post URL, platform
  post id, time) is shown to you and stored in the audit log.
- **Scheduler**: schedule an APPROVED draft from the draft editor (or
  via `schedule_post`) — at least 5 minutes ahead, Tier 2 card with the
  exact text + platform + time. A WorkManager worker runs at the
  scheduled time (network constraint), re-checks that the draft is
  still APPROVED and byte-identical to schedule time, publishes, marks
  the draft PUBLISHED, and notifies you with the link. If the draft was
  edited or un-approved since scheduling, the worker **cancels** the
  post and notifies you instead of publishing stale content. The
  **Scheduled posts** screen lists upcoming/sent/failed posts, cancels
  a scheduled post, and retries a failed one (fresh approval card).
- **Tools**: `publish_draft` (Tier 2 — publishes an APPROVED draft
  now), `schedule_post` (Tier 2 — schedules an APPROVED draft, delay in
  minutes, min 5), `list_scheduled_posts` (Tier 0), `cancel_scheduled_post`
  (Tier 1). The planner can never publish without the approval card —
  the tools route through GatedToolExecutor like everything else.
- **Honest limits**: no real OAuth or publish call has been made — the
  clients were verified against recorded fixtures only, so the first
  live connect and first real publish on a real device are where the
  flows get proven, and every endpoint is marked "verify against
  current docs before first use" in PHASE8.md. Media upload (attaching
  a photo from the device) is not in Phase 8 — Instagram takes a public
  image URL instead. WorkManager scheduling is untested on a device;
  exact timing depends on the system (Doze/battery optimization can
  delay a job). Publishing is not undoable — a live post is deleted in
  the platform's own app.

## Productivity, proactive & wellness (Phase 9)

45 tools in total now (35 from Phases 3–8 + 10 productivity tools).
Jarvis is now useful day-to-day — and gently proactive, never nagging:
every proactive feature is off by default or quiet by default, every
one has a clear off switch, and quiet hours (default 22:00–07:00)
silence the nudges. Ask in chat or by voice — "remind me to buy milk
tomorrow at 9am", "what's on my calendar today?", "give me my morning
briefing", "start a 25-minute focus session", "how much screen time
have I had today?" — or use the Tasks and Wellness screens.

- **Tasks & reminders**: tasks with title, notes, due date/time,
  priority (Low/Medium/High), and status — stored in the encrypted
  Room DB (v7). Due dates and reminders are typed in plain words
  ("tomorrow", "Friday at 5pm", "in 3 days") and parsed by a
  pure-Kotlin parser; anything it can't understand is asked about,
  never guessed. `set_reminder` attaches a reminder time; a WorkManager
  worker fires a notification at that time (tap opens the task).
  Overdue tasks get one gentle nudge per task per day — never
  repeated. The Tasks screen lists open/done/overdue tasks with
  overdue highlighting; Home shows a "Today" card with tasks due
  today + today's schedule.
- **Calendar awareness (read-only)**: Home shows today's schedule and
  `list_events` lists today/tomorrow's events (time, title,
  location). `READ_CALENDAR` is asked only when first used — never at
  app start. No event creation in Phase 9.
- **Morning briefing**: today's events + tasks due today + overdue
  tasks, polished into natural language by the LLM when one is
  configured, falling back to a plain summary when offline or
  unconfigured — the briefing never fails. Ask for it in chat, or
  turn on the **daily digest** (opt-in, off by default, default
  08:00) for a notification. No weather — there is no weather API in
  Phase 9.
- **Focus mode**: start a 25/50/90-minute session (or custom, 5–240
  min) — after a Tier 1 approval card, Jarvis turns on Do Not
  Disturb (alarms still ring) and shows a persistent countdown
  notification; at the end it restores DND and notifies. A WorkManager
  end worker is the backstop if the service is killed. Sessions are
  logged and shown in a history list.
- **Quiet hours** (default 22:00–07:00, set start = end to turn off):
  no proactive notification is posted during quiet hours — except
  your own reminders, focus results, the wind-down nudge, and Phase 8
  publish results.
- **Wellness** (all on-device, all opt-in): screen-time awareness via
  UsageStats — today's total + top 5 apps on the Wellness screen and
  via `get_screen_time` (usage access is granted in system Settings;
  the feature is disabled until granted). Break reminders (off by
  default, default 60 min of continuous screen time): at most once an
  hour, never during quiet hours or focus. Sleep wind-down (off by
  default, default 23:00): a bedtime nudge offering to turn on DND
  and dim the screen — only after you approve in-app; the
  notification alone never changes a setting.
- **Tools**: `create_task` (Tier 1), `list_tasks` (Tier 0),
  `complete_task` (Tier 1), `delete_task` (Tier 1), `set_reminder`
  (Tier 1), `list_events` (Tier 0), `morning_briefing` (Tier 0),
  `start_focus` (Tier 1), `stop_focus` (Tier 1), `get_screen_time`
  (Tier 0).
- **Honest limits**: WorkManager timing is best-effort under
  Doze/battery optimization (digest, reminders, and proactive checks
  may run late, never early). UsageStats requires a manual grant and
  can lag a few minutes. The calendar is read-only and there is no
  weather. No on-device behaviour has been tested — the first real
  build + device test happens on your machine / Android Studio.

## Plugin system, backup & analytics (Phase 10)

49 tools in total now (45 built-in + 4 plugin tools). Jarvis is
extensible, recoverable, and self-aware — without trusting anyone new:

- **Plugins**: built-in compiled plugins that register tools with the
  planner — every plugin tool is gated by the Confirmation Engine
  exactly like a built-in tool, and disabling a plugin unregisters its
  tools immediately. No dynamic code loading by design (no
  DexClassLoader, no downloading code). Two ship with the app: **Notes**
  (`save_note`, `list_notes`, `search_notes` — notes in the encrypted
  Room DB, with a Notes screen) and **Unit converter** (`convert_units`
  — length/mass/temperature, pure Kotlin, no permissions; no currency —
  that needs an exchange-rate API). Trust is per-plugin and off by
  default: only a trusted plugin's Tier 0/1 tools may auto-allow. Note
  content is data, never instructions — it is scanned by the
  InjectionGuard like any tool result. Settings → Plugins.
- **Backup & restore**: export everything (settings, tasks, notes,
  macros, drafts, research, memory, scheduled posts, focus history,
  trusted actions, audit log) to one JSON file — never API keys, OAuth
  tokens, or the database passphrase. Import validates the file
  (format version + SHA-256 checksum, 10 MB cap), shows a preview, and
  only writes after you confirm. Import merges by id: existing rows
  are kept, never overwritten (no "replace all" mode). Scheduled posts
  come back as cancelled (publishing never auto-resumes from a file);
  reminders on imported tasks are re-scheduled. An opt-in weekly
  automatic backup (off by default) keeps the newest 3 in Jarvis's
  private storage, restorable from the Backup screen. Settings →
  Backup.
- **Analytics (on-device only)**: local usage events — tool name +
  tier + outcome, publish platform + success/failure, focus duration
  bucket. No third-party SDK, nothing leaves the device, and free text
  is structurally impossible to record (payloads are validated tokens
  only — no message text, note text, URLs, or searches, ever). The
  Insights screen (Settings → Insights) shows 7/30-day aggregates:
  most-used tools, tasks created vs completed, focus minutes, posts
  published per platform, and lifetime LLM usage. The "Usage insights"
  toggle defaults to ON (local, non-personal data) — turning it off
  stops recording at once and offers to clear existing events.
- **Tools**: `save_note` (Tier 1, undo deletes the note), `list_notes`
  (Tier 0), `search_notes` (Tier 0), `convert_units` (Tier 0).
- **Honest limits**: no dynamic code loading by design; no cloud
  backup; no "replace all" import; analytics are local only. The Room
  v8 migration, SAF pickers, and the auto-backup worker are unverified
  until the first real build on your machine / Android Studio.

## Hardening (Phase 11)

A review + fix phase — no new tools, no new feature screens. Full audit
in [PHASE11.md](PHASE11.md); the Play checklist and privacy policy are
[PLAY_POLICY_CHECKLIST.md](PLAY_POLICY_CHECKLIST.md) and
[PRIVACY_POLICY.md](PRIVACY_POLICY.md).

- **Security**: a custom OkHttp `Dns` validates every resolved address
  before connecting (DNS-rebinding SSRF hardening); a network security
  config forbids cleartext traffic, and custom LLM providers must now
  use `https://` (plain `http://`, including local servers, is refused).
  A real publishing-gate bypass was found and fixed — an edited-after-
  approval draft could previously be scheduled and published with text
  you never approved. The OAuth error path now validates state before
  showing anything. The secret audit found nothing: no `Log.` calls in
  main sources, no hard-coded keys, keys/tokens only in Keystore-backed
  storage. The widget receiver was fixed (it was `exported="false"`,
  which blocked widget updates entirely) and Android Auto Backup is now
  off (`allowBackup="false"` — the app's own export is the supported
  backup path).
- **Performance**: every list query is bounded (SQL `LIMIT`s via
  `ListLimits`); three whole-table loads were fixed (memory summary,
  focus history, macro count), and `list_tasks` / `list_scheduled_posts`
  now cap what they put in the LLM's context at 50 items. A weekly
  maintenance worker prunes analytics events older than 90 days and
  audit-log entries beyond the newest 10,000. The Audit Log search now
  runs against the database instead of the capped in-memory list.
- **Robustness**: all five WorkManager workers fail silent instead of
  retrying; migrations 1_2–7_8 re-read and verified correct; and a full
  **Delete all data** flow (Settings → Danger zone) wipes the database,
  all settings, stored secrets, cached files, and analytics, after a
  biometric or typed `DELETE` confirmation.
- **Honest limits**: the DNS guard is tested with a fake Dns only, never
  against a real rebinding DNS; the performance audit is code review
  only (no profiler in the build VM); the delete-all-data wipe, the
  maintenance worker, and the widget fix are unverified on a real
  device until the first build on your machine / Android Studio.

## Build

Requirements: JDK 17, Android SDK (API 36 platform + build-tools 36.0.0).

```bash
# Debug APK
./gradlew assembleDebug

# Unit tests
./gradlew testDebugUnitTest

# Release (R8 enabled; signing wired in Phase 12)
./gradlew assembleRelease
```

Install on a connected device:

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

## API keys

API keys are **never hard-coded**. In the app, go to **Settings → API keys**,
paste each provider's key, and tap Save. Keys are stored encrypted via the
Android Keystore + EncryptedSharedPreferences. The database passphrase is a
random value generated on first run and stored the same way.

To get keys:

- **Claude** — <https://console.anthropic.com/> (API keys)
- **OpenAI** — <https://platform.openai.com/api-keys>
- **Gemini** — <https://aistudio.google.com/apikey>
- **OpenRouter** — <https://openrouter.ai/keys> (one key reaches every model
  on OpenRouter; pick a preset or type any model ID in Settings)
- **Tavily** (optional, for the premium search backend) —
  <https://app.tavily.com> (sign up, copy the API key, paste it in
  Settings → Research; the free tier is generous for personal research)

**Social OAuth credentials** (Settings → Social publishing → Connected
accounts): each platform needs its own free developer app — paste its
client ID (and client secret, where the platform needs one) in the
platform's card, then tap Connect and sign in on the platform's page.
The redirect URI to register in every developer portal is
`com.zaid.jarvis://oauth2redirect`. Full step-by-step setup for X,
Threads, LinkedIn, and Instagram is in [PHASE8.md](PHASE8.md).
Credentials and tokens are stored Keystore-encrypted, like API keys.

**Custom providers** (Settings → Custom providers): add any server that speaks
the OpenAI Chat Completions API — a local server (Ollama, llama.cpp, vLLM),
another vendor, or a corporate proxy. Enter a name, the base URL (e.g.
`https://api.example.com/v1`), and the model ID, then paste its API key in the
provider's card. The app calls `[base URL]/chat/completions`. Keys for custom
providers are stored encrypted the same way; deleting a provider deletes its
key too. Since Phase 11, base URLs must use `https://` — plain `http://`
(including local servers) is refused, because a plain-HTTP key can be read
by anyone on that network. A local server (Ollama, llama.cpp, vLLM) needs an
https endpoint (e.g. behind a reverse proxy) to be usable.

## Release keystore

```bash
./scripts/make_keystore.sh   # prompts for passwords; never commits them
```

See the script header for the `keystore.properties` setup. Release signing is
wired in Phase 12.

## Build phases

1. ✅ Skeleton
2. ✅ Confirmation Engine + audit log
3. ✅ Agent core (Claude/OpenAI/Gemini/OpenRouter/custom clients, planner, memory)
4. ✅ Voice & chat UI, overlay bubble, widget, Quick Settings tile
5. ✅ Phone tools (tiers 0–2) + real undo
6. ✅ AccessibilityService automation + macro recorder
7. ✅ Research & content engine
8. ✅ Social publishing (OAuth + scheduler)
9. ✅ Productivity, proactive & wellness features
10. ✅ Plugin system, backup, analytics
11. ✅ Hardening (security review, performance, Play-policy checklist) (this phase)
12. Release build
