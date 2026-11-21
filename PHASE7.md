# Phase 7 — Research & content engine

**Status: complete.** This is the research → draft → preview half of the
publishing gate (Master Prompt §9 item 7). **Nothing is published in this
phase** — publishing is Phase 8.

## What was built

### Web search — two backends, one interface

`search_web` (Tier 0) answers through a `WebSearch` interface
(`:tools/research/WebResearch.kt`), so more backends can be added later
without touching the tools or the planner:

- **DuckDuckGo (default, free, no key)** — the lite HTML endpoint
  (`lite.duckduckgo.com/lite/`), parsed by a minimal dependency-free
  parser (`SearchResultParser.parseDuckDuckGoLite`, incl. unwrapping
  DuckDuckGo's `uddg` redirect links). No Jsoup: it isn't in the
  version catalog, and the parser is ~60 lines.
- **Tavily (optional premium backend)** — chosen over Brave Search
  because it is built for AI research (one JSON call, relevance-scored
  results, generous free tier). Key from <https://app.tavily.com>,
  stored Keystore-encrypted like every other key, entered in
  Settings → Research. Parsed with kotlinx.serialization
  (`SearchResultParser.parseTavilyJson`).

Results carry title, URL, snippet, source domain, and fetched
timestamp; capped at `maxSearchResults` (1–10, default 5, in Settings).

### Page fetching — with an SSRF guard

`fetch_page` (Tier 0) downloads one URL with OkHttp and returns its
readable text via `HtmlTextExtractor` (strips scripts/styles/nav/footer,
keeps headings + paragraphs + list items, decodes entities, truncates
to 8,000 chars). Guard rails, enforced in the tool AND in the fetcher
(defense in depth):

- http(s) only; `UrlSafety.validateUrl` refuses anything else.
- Refuses localhost, `*.localhost`, loopback, private, link-local,
  carrier-grade NAT, and reserved IP ranges (IPv4 + IPv6), plus
  leading-zero octets (ambiguous octal — fail closed).
- Re-validates the final URL after redirects; refuses non-`text/html` /
  `text/plain` content; 200 KB download cap.
- **Honest limit:** a hostname that *resolves* to a private IP (DNS
  rebinding) is not caught — that needs a custom OkHttp `Dns`, a
  Phase 11 hardening item.

### Research pipeline — `ResearchEngine` (`:agent/research/`)

`research_topic` (Tier 0) runs search → fetch → synthesize:

1. **Search** — up to 3 template-derived queries (the topic, an
   "overview" variant, a "recent developments" variant), deduplicated
   by URL. One failing query doesn't kill the job.
2. **Fetch** — the top 5 pages. Every page's text is scanned by the
   Phase 3 `InjectionGuard` BEFORE it reaches the synthesis prompt: a
   page with instruction-like text is excluded, and the brief says so.
   Search snippets are never fed to the LLM — only fetched page text,
   and only after scanning.
3. **Synthesize** — the configured LLM (via the existing
   `LlmProviderClient` contract, usage recorded through the normal
   `UsageTracker`) writes a structured brief: summary, key facts with
   `[n]` citations, open questions, confidence notes. The engine parses
   the response and **drops any fact it cannot bind to a fetched
   source** — a brief never carries an unsourced claim.

Jobs report progress (`Searching` → `Fetching` → `Synthesizing` →
`Done`) to the UI and are cancellable via the kill switch (checked
before every step). The finished brief is saved in the encrypted Room
DB (see below).

### Storage — Room v5

`JarvisDatabase` is now version 5, with a real `MIGRATION_4_5` (creates
both new tables):

- `research_records` — topic + brief JSON + created date
  (`ResearchEntity`/`ResearchDao`/`ResearchRepository`). A record whose
  brief JSON is unreadable is skipped on read (never crashes the
  history screen) and can still be deleted.
- `content_drafts` — title, body, platform, status, optional research
  id, created/updated dates (`ContentDraftEntity`/`ContentDraftDao`/
  `DraftRepository`).

### Content drafting — the "draft" half of the gate

- `ContentDraft` model + `PlatformPreset` (X / Twitter 280, Threads
  500, Instagram 2200, LinkedIn 3000, Blog 10000, Other 2000 chars)
  + `DraftLimits.enforce` (hard-truncate with a visible "…" and a
  `truncated` flag).
- `DraftComposer` (`:agent/research/`): given topic + optional research
  context + platform + tone (from the Personality setting), the LLM
  writes the draft. The prompt states the limit; if the model exceeds
  it, Jarvis asks for **one** rewrite, then hard-truncates and flags
  it. Code fences and surrounding quotes are stripped; quotes inside
  the post survive. The draft is saved as **DRAFT**.
- **Status rules** (enforced in `DraftRepository`, the publishing
  gate): `save` accepts DRAFT and APPROVED only — **PUBLISHED is
  refused**, because publishing is Phase 8 and a draft must never
  claim to be published when it isn't. `approve()` moves DRAFT →
  APPROVED only, and only via the deliberate "Approve for publishing"
  button. An unknown status string read back from the DB maps to DRAFT
  (fail-safe).
- **`draft_post` flow (Tier 1):** the approval card shows topic +
  platform + tone — NOT the generated text, because the text doesn't
  exist yet (the LLM writes it after approval). The generated draft is
  then shown in the Drafts screen, where Zaid reviews, edits, and
  approves it. Nothing is published from any screen in this phase.

### Screens & navigation

- **Research screen** (Home → Research, Settings → Research): run a
  topic, watch live progress, browse history with expandable full
  briefs (citations + sources), delete.
- **Drafts screen** (Home → Drafts, Settings → Research): list with
  status + platform filter chips, status label, char count, date;
  tap to edit, delete, or start a blank draft.
- **Draft editor**: title / platform (preset dropdown with limits) /
  body editing, live character count vs the platform limit (red when
  over), Save, and "Approve for publishing" (only when DRAFT, disabled
  when over the limit), with a note that nothing is published from
  this screen.
- **Settings → Research**: backend selection (DuckDuckGo free / Tavily
  with masked key field), max-results slider (1–10), a note that
  fetched web content is treated as untrusted data, and links to
  Research history and Drafts.
- Home: new "Research" and "Drafts" chips; the "Trending topics" chip
  (whose TODO said Phase 7) now opens Research.
- Planner system prompt gained rule (7): the research/draft
  capability, and never to publish.

### Tool table (4 new; 31 total)

| Tool | Tier | What it does |
|---|---|---|
| `search_web` | 0 | Search the web (DuckDuckGo free / Tavily with key) |
| `fetch_page` | 0 | Fetch one page as readable text (SSRF-guarded) |
| `research_topic` | 0 | Full pipeline: search → fetch → sourced brief |
| `draft_post` | 1 | Generate a draft and save it for review (never publishes) |

### Files

New this phase (main): `:core/core/research/` — `ResearchModels.kt`,
`ContentDraft.kt`, `HtmlTextExtractor.kt`, `UrlSafety.kt`,
`SearchResultParser.kt`, `DraftLimits.kt`; `:data/db/ResearchEntity.kt`,
`:data/db/ContentDraftEntity.kt`, `:data/repository/ResearchRepository.kt`,
`:data/repository/DraftRepository.kt`; `:tools/research/WebResearch.kt`,
`:tools/research/ResearchTools.kt`; `:agent/research/WebSearchClients.kt`,
`:agent/research/OkHttpPageFetcher.kt`, `:agent/research/ResearchEngine.kt`,
`:agent/research/DraftComposer.kt`; `:app/screens/ResearchScreen.kt`,
`:app/screens/DraftsScreen.kt`, `:app/screens/DraftEditorScreen.kt`,
`:app/viewmodel/ResearchViewModel.kt`, `:app/viewmodel/DraftsViewModel.kt`.
Changed: `JarvisSettings` + `SettingsRepository` (backend + max results),
`SecureKeyStore` + `ApiKeyRepository` (Tavily key), `JarvisDatabase` (v5 +
`MIGRATION_4_5`), `DataModule`, `ToolsModule` (4 tools), `AgentModule`
(research providers), `Planner` (rule 7), `SettingsViewModel`,
`SettingsScreen` (Research section + generic key field), `JarvisNavGraph`
(3 destinations), `HomeScreen` (chips), `app/build.gradle.kts`
(version `0.7.0-phase7`). Tests: 10 new test classes (see below).
Project now 216 files (181 `.kt`).

## Verification

Command: `bash ~/opt/verify-pom/verify-phase7.sh` (kotlinc 2.4.21 +
JUnitCore — `./gradlew` still cannot run in this VM; the script tries
it once briefly, as in Phases 1–6).

- **373 unit tests pass, 0 failures** (300 in Phase 6 + 73 new):
  - `:core` 155 (32 new: HTML extraction, SSRF guard, DDG/Tavily
    parsing from recorded fixtures, draft limits, brief formatting)
  - `:tools` 130 (13 new: tool schemas/dry-runs/execution with fake
    network, URL validation, registry count 31)
  - `:agent` 58 (15 new: pipeline orchestration with fake
    search/fetch/LLM — progress order, kill-switch cancel, search
    failure, injection-flagged page excluded, citation binding;
    draft length enforcement, rewrite-once, truncation, fence/quote
    stripping)
  - `:data` 30 (13 new: repository round-trips with fake DAOs,
    brief-JSON handling, draft status transitions incl. PUBLISHED
    refused and unknown status → DRAFT)
- **All Android sources compile via kotlinc** (Compose +
  serialization plugins, android.jar API 36): **637 class files,
  0 errors**; only the 2 pre-existing Phase 1 deprecation warnings
  remain.
- Secret scan clean; no real network calls in any test (recorded
  fixtures / fakes only); no hard-coded keys.

Bugs found and fixed during verification: the Tavily parser is named
`parseTavilyJson` (not `parseTavily`) and takes no query argument;
tool params are `JsonObject` (not `Map`); risk tiers are
`READ_ONLY`/`REVERSIBLE` (not `TIER_0`/`TIER_1`); `Done` carries a
`ResearchBrief` (not a `ResearchRecord`); `ResearchBrief.confidenceNotes`
is a `String`; `KillSwitch.cancelAll()` takes no argument; the
pre-existing `GatedUndoTest` registry count moved 27 → 31. Two
unnecessary `!!` warnings in `ResearchTools.kt` were also cleaned.

## What was NOT verified

Same standing limitation as Phases 1–6: the real Gradle build
(`./gradlew assembleDebug`) cannot run in this VM, so Hilt/KSP codegen,
the Room v5 migration (`MIGRATION_4_5`), resource merging, and the
manifest are unverified until the first real build on Salahuddin's
machine / Android Studio. **No real search, fetch, or LLM call was
ever made** — the DuckDuckGo endpoint, Tavily, and the page fetcher
are verified only against recorded fixtures and fakes. No on-device
behaviour tested.

## Setup notes

- DuckDuckGo works out of the box — no key, no setup.
- For Tavily: sign up at <https://app.tavily.com>, copy the API key,
  then in Jarvis go to Settings → Research, pick "Tavily (needs API
  key)", and paste the key. The free tier is generous for personal
  research. Switching back to DuckDuckGo needs no key change.
- Research and drafts need a configured LLM provider (Settings → API
  keys) — the synthesis and drafting calls go through the same
  provider as chat.

## Honest limitations

- Search quality depends on the backend: DuckDuckGo's lite endpoint
  is a free HTML scrape and can be thin or slow; Tavily is the better
  backend when it matters.
- Research synthesis quality depends on the LLM and the pages it
  reads; briefs cite sources, but a wrong page still yields a wrong
  brief — the citations are there so Zaid can check.
- JavaScript-rendered pages return little or no readable text (the
  fetcher doesn't run a browser); `fetch_page` says so honestly.
- The SSRF guard blocks literal private IPs but not DNS rebinding
  (Phase 11 hardening item).
- Publishing is Phase 8 — approved drafts wait until then.

## What remains

- **Phase 8** — Social publishing (OAuth + scheduler). Needs
  Salahuddin's explicit approval; will require manual steps from him
  (create social developer accounts, paste OAuth credentials).
- **Phase 9** — Productivity, proactive & wellness features.
- **Phase 10** — Plugin system, backup, analytics.
- **Phase 11** — Hardening (security review, performance, Play-policy
  checklist; incl. the DNS-rebinding SSRF hardening from this phase).
- **Phase 12** — Release build (signed APK/AAB).
- First real `./gradlew assembleDebug` + on-device test still pending
  on his machine / Android Studio (all phases).
