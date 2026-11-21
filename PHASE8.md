# Phase 8 — Social publishing (OAuth + scheduler)

The second half of the publishing gate: research → draft → preview →
Allow → **publish → report**. Jarvis can now publish APPROVED drafts to
X, Threads, LinkedIn, and Instagram through their official APIs, and
schedule posts for later — always after Zaid approves the exact post on
a Tier 2 card. Nothing in this phase scrapes, automates a social app,
or uses an unofficial endpoint.

## What was built

### Platforms — one client per platform, official APIs only

`SocialPlatform` (`:core/social/`) carries each platform's OAuth
endpoints, scopes, character limit (re-used from the Phase 7
`PlatformPreset`, so the two phases can never disagree), docs URL, and
a plain-language account requirement. `PlatformClient`
(`:agent/social/`) is the interface; one implementation per platform.

| Platform | Auth | Publish flow | Char limit | Notes |
|---|---|---|---|---|
| X / Twitter | OAuth 2.0 + PKCE | `POST /2/tweets` (text only) | 280 | Post URL built from the handle + returned id. Media upload is a future addition. |
| Threads | OAuth 2.0 + PKCE | Two-step: `POST /v1.0/{user-id}/threads` (`media_type=TEXT`) → `POST /v1.0/{user-id}/threads_publish` → permalink lookup | 500 | Short-lived tokens exchanged for 60-day long-lived tokens at connect time. |
| LinkedIn | OAuth 2.0 + PKCE | **Posts API** `POST /v2/posts` (current recommended endpoint; the older `/v2/ugcPosts` is deprecated for new apps) | 3000 | Post URN comes back in the `x-restli-id` header. Author URN from `/v2/userinfo` `sub`. |
| Instagram | OAuth 2.0 (Meta dialog; client-secret authenticated, no PKCE params) | Two-step Graph API: `POST /{ig-user-id}/media` → `POST /{ig-user-id}/media_publish` → permalink lookup | 2200 | **Requires a Business/Creator account linked to a Facebook Page** + a Meta developer app. Every post needs a photo — a public image URL is required at publish time (the API has no text-only posts). |

> **Verify against current docs before first use.** Platform APIs
> change. Each endpoint above is the best-known documented flow as of
> the build date; the docs URLs are in `SocialPlatform.docsUrl` and in
> the setup notes below. The first live connect and first real publish
> on a real device are where these flows get proven — no real OAuth or
> publish call was made during this phase.

### OAuth — PKCE with Custom Tabs, no AppAuth

AppAuth-Android is not in the project's version catalog, and the flow
is small, so PKCE is implemented manually (`OAuthFlowManager` in
`:app/social/`): build the authorization URL with an S256 code
challenge (RFC 7636 — verified against the Appendix B test vector in
`PkceTest`), open it in a Custom Tab (falling back to the default
browser), handle the redirect in `OAuthRedirectActivity`
(`com.zaid.jarvis://oauth2redirect`, registered in the manifest),
validate the `state` parameter, exchange the code, and fetch the /me
profile. The verifier and state live only in memory for one flow —
if the process dies mid-sign-in, the flow simply fails and Zaid
retries. Meta's web OAuth dialog (Instagram) does not take PKCE
parameters; its token exchange is authenticated with the client secret
instead.

Tokens (access + refresh) are stored Keystore-encrypted in
`SecureKeyStore` under the `social_` prefix, never in Room, never in
DataStore, never logged. Account metadata (handle, display name, token
expiry) lives in its own DataStore file. Client IDs/secrets are stored
the same encrypted way, entered in Settings → Connected accounts.

**Token refresh** (`AuthenticatedApiCaller`): an expired token is
refreshed before the call; a 401 refreshes once and retries once; a
failed refresh — or a second 401 — clears the tokens (the account shows
as disconnected) and tells Zaid to reconnect. Threads and Instagram
refresh their long-lived access tokens directly (they have no separate
refresh token); X and LinkedIn use `grant_type=refresh_token`.

### Publishing gate — enforced in code

`PublishService` (`:tools/social/`) is the single place where a draft
may move to PUBLISHED, implemented by `PublishServiceImpl` (`:agent`)
and shared by the `publish_draft` tool and the WorkManager worker, so
both paths enforce exactly the same rules:

1. **Only APPROVED drafts publish.** DRAFT is refused with a plain
   message ("approve it in Drafts first"); PUBLISHED is refused too.
2. **Edited-after-approval is refused.** `approve()` records the
   SHA-256 of the body (`approvedBodyHash`, new Room column in v6);
   if the body no longer matches, the publish is refused until the
   edited text is approved again. The editor shows "edited since
   approval — approve again" and offers a re-approve button. Drafts
   approved before Phase 8 (no hash recorded) can still publish — the
   publish card itself shows the exact text.
3. **Character limits are re-checked at publish time.** Over-limit
   text is refused, never silently truncated.
4. **Instagram requires a public image URL.**
5. **Only after the platform confirms the post** does the draft become
   PUBLISHED, via `DraftRepository.markPublished()` — the only place
   PUBLISHED may be set. `save()` still refuses PUBLISHED.
6. The publish report (post URL, platform post id, time) is shown to
   Zaid ("Published to X — link") and stored in the audit log via the
   tool result.

The approval card is the layer above: `publish_draft` and
`schedule_post` are Tier 2 (card + second confirmation) showing the
exact text, platform, account, and — for scheduling — time. Edited
publish previews are refused outright (the card text *is* the post;
changing it on the card would publish unapproved text — Zaid edits the
draft instead, which re-requires approval).

### Scheduler — WorkManager + Room v6

- `ScheduledPost` Room entity (Room v6, real `MIGRATION_5_6`, which
  also adds the `approvedBodyHash` column to `content_drafts`): draft
  id, platform, account handle, scheduled time, content hash (SHA-256
  of the body at schedule time), status (SCHEDULED / PUBLISHED /
  FAILED / CANCELLED), failure reason, post URL, WorkManager job id,
  and the image URL for Instagram posts.
- Scheduling UI: from the draft editor, "Schedule…" picks a date +
  time (must be in the future, min 5 minutes ahead — validated in
  `ScheduleValidation` and again in the tool and the scheduler). The
  schedule itself needs a Tier 2 card.
- `PublishPostWorker` (one-time work, network constraint) re-checks
  the gate at run time: draft still APPROVED and byte-identical to
  schedule time. If the draft was edited or un-approved, the worker
  **cancels** the post and notifies Zaid instead of publishing stale
  content. On success it marks the draft PUBLISHED and notifies with
  the link; on failure it marks the post FAILED with the reason. The
  worker always returns success after handling the post — a failed
  post is retried by Zaid from the Scheduled screen with a fresh
  approval card, never silently retried by WorkManager.
- **Scheduled posts screen**: upcoming/sent/failed posts, cancel a
  scheduled post (Tier 1 card), retry a failed post (re-schedules for
  5 minutes from now through `schedule_post`, fresh Tier 2 card, then
  deletes the old FAILED row).
- Dependencies reach the worker through a Hilt entry point
  (`PublishWorkerEntryPoint`) — no `androidx.hilt:hilt-work`
  dependency needed.

### Notifications

One channel ("Publishing"): success (tap opens the published post in
the browser), failure, and cancellation (tap opens the app, where the
Scheduled screen shows the reason and offers a retry). Skipped silently
when the notification permission isn't granted — results are always
visible in-app too. `POST_NOTIFICATIONS` is declared in the manifest.

### Tool table (4 new; 35 total)

| Tool | Tier | What it does |
|---|---|---|
| `publish_draft` | 2 | Publishes an APPROVED draft now (optional `image_url` — required for Instagram). |
| `schedule_post` | 2 | Schedules an APPROVED draft (`delay_minutes`, min 5; optional `image_url`). |
| `list_scheduled_posts` | 0 | Lists scheduled posts with statuses and failure reasons. |
| `cancel_scheduled_post` | 1 | Cancels a scheduled post; the draft is untouched. |

The planner prompt gains rule (8): publishing tools exist, only for
APPROVED drafts, always with the approval card first. Publishing is
not undoable — a live post is deleted in the platform's own app.

### Screens & navigation

- **Connected accounts** (Settings → Social publishing): one card per
  platform — status, handle, token expiry, client ID/secret fields
  (Keystore-encrypted), Connect (Custom Tab sign-in) / Disconnect.
- **Scheduled posts** (Home chip, Settings, Drafts): list with status,
  failure reasons, post links, Cancel / Retry.
- **Draft editor**: Publish now + Schedule… for APPROVED drafts, an
  image-URL dialog for Instagram, date/time pickers for scheduling,
  and the edited-since-approval warning with re-approve.
- **Settings**: new "Social publishing" section. Home gains a
  "Scheduled posts" chip. Two new nav destinations; "Delete all
  secrets" now also clears social tokens and credentials.

### Files

New this phase (main): `:core/social/` — `SocialPlatform.kt`,
`SocialModels.kt`, `Pkce.kt`, `ScheduleValidation.kt`;
`:tools/social/` — `SocialPublishing.kt`, `PublishingTools.kt`;
`:data/db/ScheduledPostEntity.kt`,
`:data/repository/ScheduledPostRepository.kt`,
`:data/repository/SocialAccountRepository.kt`;
`:agent/social/` — `OAuthClient.kt`, `AuthenticatedApiCaller.kt`,
`PlatformClients.kt`, `SocialPublisherImpl.kt`,
`PublishServiceImpl.kt`; `:agent/HttpBody.kt` (safe capped body
reader — see "Bugs found" below); `:app/social/` — `OAuthFlowManager.kt`,
`OAuthRedirectActivity.kt`, `PublishNotifier.kt`,
`PublishPostWorker.kt`, `WorkManagerSocialScheduler.kt`,
`SocialAccountStoreImpl.kt`; `:app/screens/ConnectedAccountsScreen.kt`,
`:app/screens/ScheduledPostsScreen.kt`;
`:app/viewmodel/SocialAccountsViewModel.kt`,
`:app/viewmodel/ScheduledPostsViewModel.kt`.
Changed: `ContentDraft` + `ContentDraftEntity` (`approvedBodyHash`),
`DraftRepository` (`markPublished()`, approval hash),
`JarvisDatabase` (v6 + `MIGRATION_5_6`), `DataModule`,
`SecureKeyStore` (social secret keys), `ToolsModule` (4 tools),
`AgentModule` (social providers), `Planner` (rule 8), `AppModule`
(social providers), `DraftsViewModel` (publish/schedule),
`SettingsViewModel` (delete-all-secrets covers social),
`WebSearchClients` + `OkHttpPageFetcher` (the `readUtf8` fix below),
`DraftEditorScreen`, `DraftsScreen`, `SettingsScreen`, `HomeScreen`,
`JarvisNavGraph` (2 destinations), `AndroidManifest.xml`
(`POST_NOTIFICATIONS` + OAuth redirect activity),
`gradle/libs.versions.toml` (androidx.browser 1.8.0),
`app/build.gradle.kts` (workmanager + browser deps, version
`0.8.0-phase8`). Tests: 8 new test classes + `DraftRepositoryTest`
extensions (see below). Project now 251 files (214 `.kt`).

## Verification

Command: `bash ~/opt/verify-pom/verify-phase8.sh` (kotlinc 2.4.21 +
JUnitCore — `./gradlew` still cannot run in this VM; the script tries
it once briefly, as in Phases 1–7). New verification deps fetched for
this phase: `androidx.work:work-runtime` + `work-runtime-ktx` 2.9.0,
`androidx.browser:browser` 1.8.0, `com.google.guava:listenablefuture`
1.0, `androidx.concurrent:concurrent-futures` 1.2.0 (added to
`~/opt/verify-pom/fetch-deps.sh`).

- **475 unit tests pass, 0 failures** (373 in Phase 7 + 102 new):
  - `:core` 179 (24 new: PKCE S256 against the RFC 7636 test vector,
    verifier/state format, schedule validation incl. the 5-minute
    floor, platform resolution + char limits, approval content binding,
    token-expiry margin)
  - `:tools` 147 (17 new: publish/schedule tool previews, parameter
    validation, content-hash recording, list/cancel behaviour, edited
    preview refusal; registry count 35)
  - `:agent` 102 (44 new: platform response parsing from recorded
    fixtures for all four platforms + token responses + error mapping;
    publish-gate enforcement — DRAFT refused, APPROVED allowed,
    edited-after-approval refused, over-limit refused, PUBLISHED only
    via markPublished, scheduled-post cancellation on stale drafts;
    token refresh — 401 → refresh → retry once, proactive refresh,
    refresh failure → disconnected, second 401 → disconnected)
  - `:data` 47 (17 new: scheduled-post repository round-trips with a
    fake DAO incl. unknown status → CANCELLED and unknown platform
    skipped; markPublished transitions; approval-hash recording,
    invalidation on edit, and re-approval)
- **All Android sources compile via kotlinc** (Compose +
  serialization plugins, android.jar API 36): **780 class files,
  0 errors**; only the 2 pre-existing Phase 1 deprecation warnings
  remain.
- Secret scan clean; no real network calls in any test (recorded
  fixtures / fakes only); no hard-coded keys, client secrets, or
  tokens.

Bugs found and fixed during verification:
- **Okio `readUtf8(n)` throws EOFException on small bodies** — the
  Phase 7 search/fetcher code and the new social caller all read
  bodies with `source().readUtf8(MAX_BYTES)`, which demands exactly
  that many bytes; every small API response would have failed with
  "EOFException". The AuthenticatedApiCaller test caught it (its fake
  responses are small). Fixed with a shared `readUtf8Capped()`
  helper (`:agent/HttpBody.kt`) that requests up to the cap and reads
  only what arrived; applied to all four call sites (the two Phase 7
  ones included).
- Member extensions declared inside `object SocialJson` aren't
  visible outside it — moved `string`/`obj` to top-level internal
  extensions.
- Smart casts on cross-module public properties fail — `OAuthTokens`
  and `OAuthClientCredentials` fields are now copied to locals before
  null-checking (3 sites).
- The pre-existing registry-count assertions moved 31 → 35 in both
  `GatedUndoTest` and `ResearchToolsTest`.
- `MIGRATION_5_6`'s `ALTER TABLE` initially landed in `MIGRATION_4_5`
  during editing; moved to the correct migration and re-checked.

## What was NOT verified

- The real Gradle build (`./gradlew assembleDebug`) — still blocked
  in this VM (Gradle daemon dispatch + egress proxy, same as Phases
  1–7). First real build on Salahuddin's machine / Android Studio.
- Hilt codegen (including the new providers and the worker entry
  point), Room codegen (including the v6 migration), resource merging,
  and the manifest — kotlinc checks the sources, not the generated
  code or the merged manifest.
- Any real OAuth flow, token refresh, or publish call — no real social
  API call was made. The first live connect per platform, the first
  real publish, and the first scheduled publish must happen on a real
  device with Salahuddin's own developer accounts and credentials.
- WorkManager scheduling on a device (timing under Doze/battery
  optimization, exact-delay behaviour), Custom Tabs behaviour,
  notification delivery, and the OAuth redirect round-trip.
- The `approvedBodyHash` migration on a database that already has
  drafts (the ALTER TABLE is straightforward, but it has not run
  against a real v5 database).

## Setup notes

How Salahuddin sets each platform up. The redirect URI for **every**
platform is:

```
com.zaid.jarvis://oauth2redirect
```

1. **X** — <https://developer.x.com/en/portal/dashboard>
   - Create a free developer account and a Project + App. In the app's
     settings, enable OAuth 2.0, set the app type to Native App (public
     client, PKCE), add the redirect URI above, and set permissions to
     Read and Write. Copy the **Client ID** (the client secret is
     optional for a native app — leave it blank in Jarvis unless X
     requires it for your app type).
   - In Jarvis: Settings → Social publishing → Connected accounts → X
     → Add credentials → paste the Client ID → Save → Connect → sign
     in on X's page.
   - Docs: <https://developer.x.com/en/docs/x-api/tweets/manage-tweets/introduction>
2. **Threads** — <https://developers.facebook.com/apps/>
   - Create a Meta developer app, add the **Threads API** product
     (Threads use cases → Threads API). In the Threads API settings,
     add the redirect URI above. Copy the **App ID** (client ID) and
     **App Secret** (client secret — required for Threads).
   - In Jarvis: same flow as X, in the Threads card (both ID and
     secret). The token is exchanged for a 60-day long-lived token at
     connect time.
   - Docs: <https://developers.facebook.com/docs/threads/posts>
3. **LinkedIn** — <https://www.linkedin.com/developers/apps>
   - Create an app, add the **Share on LinkedIn** and **Sign In with
     LinkedIn using OpenID Connect** products. In Auth settings, add
     the redirect URI above. Copy the **Client ID** and **Client
     Secret** (both required).
   - In Jarvis: same flow, in the LinkedIn card.
   - Docs: <https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api>
4. **Instagram** — <https://developers.facebook.com/apps/>
   - Prerequisites first: the Instagram account must be a **Business
     or Creator account linked to a Facebook Page** (Instagram app →
     switch to a professional account, then Accounts Centre → link a
     Page). A personal Instagram account cannot publish via the API.
   - Create a Meta developer app, add the **Instagram API** product
     (with Facebook Login). Add the redirect URI above in the Facebook
     Login settings. Copy the **App ID** and **App Secret**.
   - In Jarvis: same flow, in the Instagram card. At connect time
     Jarvis lists your Pages and finds the linked Instagram account;
     if none is linked, Connect fails with instructions.
   - Every Instagram post needs a photo: when publishing or
     scheduling, paste a **public image URL** (the image must be
     reachable by Meta's servers — a private or slow URL is rejected).
   - Docs: <https://developers.facebook.com/docs/instagram-platform/content-publishing>

If a platform rejects the redirect URI: some portals require the URI
to be an exact match — check for typos, and note that Meta's portals
sometimes require the redirect URI to be added under both the app's
Facebook Login settings and the product's own settings. If a connect
fails, the error message names the platform's reason.

## Honest limitations

- **No real OAuth or publish call was made.** The clients were verified
  against recorded fixtures only. Endpoints, response shapes, and
  OAuth details are the best-known documented flows as of the build
  date and are marked "verify against current docs before first use".
  Meta's redirect-URI rules for native apps are the most likely place
  reality differs (the custom scheme is standard for X and LinkedIn;
  for Threads/Instagram, verify that the Meta portal accepts
  `com.zaid.jarvis://oauth2redirect` — an https App Link may be
  required in practice).
- **Media upload is not in Phase 8.** X/Threads/LinkedIn publish text
  only; Instagram takes a public image URL. Attaching a photo from the
  device (upload to the platform's media endpoints) is a future
  addition.
- **WorkManager timing is best-effort.** A scheduled post runs when
  the system allows it — Doze and battery optimization can delay a
  job, so a post scheduled for 09:00 may publish a few minutes late.
  The worker never publishes early.
- **Threads/Instagram long-lived tokens last 60 days** and refresh
  automatically; if a refresh fails (e.g. the app's permissions were
  revoked on the platform), the account disconnects and must be
  reconnected.
- **Publishing is not undoable.** Jarvis cannot delete a live post —
  that happens in the platform's own app.
- **Instagram's container step can fail for slow images**: if the
  image URL is slow to load, the publish may fail with a "media not
  ready" style error — retrying usually works once the image is cached.

## What remains

- **Phase 9** — Productivity, proactive & wellness features.
- **Phase 10** — Plugin system, backup, analytics.
- **Phase 11** — Hardening (security review, performance, Play-policy
  checklist; incl. the DNS-rebinding SSRF hardening from Phase 7).
- **Phase 12** — Release build (signed APK/AAB — will require manual
  steps from Salahuddin: enable Accessibility, create social developer
  accounts, paste API keys/OAuth credentials).
- First real `./gradlew assembleDebug` + on-device test still pending
  on his machine / Android Studio (all phases), plus the first live
  OAuth connect and first real publish per platform.
