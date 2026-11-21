# Zaid Jarvis — Google Play policy checklist

**Status: prepared in Phase 11 (2026-11-16).** This checklist covers every
sensitive permission and API the app uses, what Play requires for it, what
Jarvis already does, and what still needs a human step in Play Console.
Nothing here is a substitute for reading the current Play policy text at
submission time — policies change, and the first real submission is where
each declaration gets tested.

App facts: package `com.zaid.jarvis`, min SDK 26, **target SDK 36**,
version `0.11.0-phase11`. No ads, no in-app purchases, no account system
of its own, no third-party analytics or crash SDK, no advertising ID.

## 1. Target API level

| Item | Status |
|---|---|
| Target SDK | 36 (Android 16) — current at the time of writing |
| Min SDK | 26 (Android 8.0) |

Play requires new apps to target a recent API level (currently within one
year of the latest Android release). Target 36 satisfies that today.
**Human step:** re-check the required target level at submission time; if
a newer Android version has shipped, bump `targetSdk` (and `compileSdk`)
and re-test before submitting.

## 2. Accessibility API (AccessibilityService) — declaration required

**What Jarvis does:** the optional automation feature (Phase 6) uses an
`AccessibilityService` to see the screen and tap / type / scroll in other
apps — only after the user enables it by hand in Android Settings →
Accessibility, turns Automation on in Jarvis Settings, and approves each
individual action on an approval card. Macros are recorded by the user
and replayed only after a card listing every step.

**Policy requirement:** apps using the Accessibility API must submit an
Accessibility API declaration in Play Console, show a prominent
disclosure before the user enables the service, and use the API only for
its declared purpose. Play does not allow Accessibility for general
automation that could be done another way — the justification must be
that the feature exists to help the user operate their own device
hands-free / by voice, which is exactly Jarvis's case (voice-first
assistant; the user asks, approves, and the service carries out the
approved taps).

**In the app (verified Phase 11):**
- Prominent disclosure text is the service description shown by Android
  when the user enables it (`accessibility_service_description`):
  > "Zaid Jarvis uses Accessibility to see the screen and to tap buttons,
  > type text, and scroll in other apps — only after you approve each
  > action, and only while Automation is turned on in Settings. It never
  > reads or stores passwords, never acts on screens with password
  > fields, and never automates banking or payment apps."
- Service config is minimal: `canRetrieveWindowContent`,
  `canPerformGestures`, `flagReportViewIds`,
  `flagRetrieveInteractiveWindows`; only `typeViewClicked` /
  `typeViewTextChanged` / `typeViewScrolled` events; no touch
  exploration, no key-event interception.
- The in-app toggle is greyed out until the service is enabled, and
  automation is off by default.
- Password fields are never read/stored/typed into; banking/payment apps
  are blocked (best-effort blocklist + password-field backstop).

**Human steps:**
- [ ] Complete the Accessibility API declaration in Play Console, using
      the justification above.
- [ ] Keep the disclosure text accurate — if automation gains a
      capability, update the string first.

## 3. Permissions — one by one

### 3.1 Declared permissions and their justification

| Permission | Why Jarvis needs it | Runtime flow |
|---|---|---|
| `INTERNET` | LLM API calls, web search/fetch, social publishing (normal permission, granted at install) | — |
| `RECORD_AUDIO` | Voice input + wake-word listening | Asked only on first mic tap / when wake word is enabled, never at app start |
| `CAMERA` | Flashlight only (`setTorchMode`). Jarvis never takes a photo or video | Asked only when the flashlight tool is first used |
| `READ_CONTACTS` | Resolve contact names for calls/messages ("text Alice") | Asked only when a contact tool is first used; contacts are never uploaded |
| `BLUETOOTH_CONNECT` (+ legacy `BLUETOOTH`/`BLUETOOTH_ADMIN` maxSdk 30) | Turn Bluetooth on/off | Asked only when the Bluetooth tool is first used |
| `READ_CALENDAR` | Read today's schedule for Home + briefing (read-only; Jarvis never creates events) | Asked only when the calendar is first used |
| `PACKAGE_USAGE_STATS` | Screen-time feature (today's total + top apps), opt-in, on-device only | Granted only in system Settings via an in-app button; feature disabled until granted |
| `POST_NOTIFICATIONS` | Reminders, digest, focus results, publish results | System runtime permission dialog on API 33+ |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_MICROPHONE` + `FOREGROUND_SERVICE_SPECIAL_USE` | Foreground services (see §4) | Normal permissions |
| `SYSTEM_ALERT_WINDOW` | Floating overlay bubble over other apps | Granted only in system Settings; Settings toggle is greyed until granted, with a grant button and an explanation |
| `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`, `CHANGE_WIFI_STATE` | Network status + Wi-Fi toggle/panel | Normal permissions |
| `MODIFY_AUDIO_SETTINGS` | Volume control | Normal permission |
| `SET_ALARM` | Alarms/timers via the Clock app intent | Normal permission |
| `ACCESS_NOTIFICATION_POLICY` | Do Not Disturb control (also needs Notification Policy access in system Settings) | Normal permission + special access |
| `WRITE_SETTINGS` | Brightness ("Modify system settings", granted in system Settings) | Special access |

### 3.2 Permissions deliberately NOT declared (verified Phase 11)

| Permission | Why not |
|---|---|
| `CALL_PHONE`, `SEND_SMS` | Tier 2 tools are intent-only: Jarvis opens the dialer / messaging app pre-filled and the user taps Call / Send. Jarvis never dials or sends itself |
| `QUERY_ALL_PACKAGES` | The installed-apps list uses `queryIntentActivities` with a launcher intent, which needs no such permission |
| Location (`ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION`) | Jarvis has no location feature at all |
| `READ_MEDIA_*`, `READ_EXTERNAL_STORAGE` | No photo/media access; backups use the Storage Access Framework picker (user-chosen file) |
| Exact alarms (`SCHEDULE_EXACT_ALARM`) | All scheduling uses WorkManager (best-effort timing, documented) |
| `REQUEST_INSTALL_PACKAGES`, `BIND_DEVICE_ADMIN`, etc. | Not used |

### 3.3 Data safety — permissions summary

The Play data-safety form must list the sensitive permissions above
(microphone, camera, contacts, calendar, usage stats, notifications).
Each is optional and feature-gated: denying any of them only disables
the feature that needs it, never the app.

## 4. Foreground services

| Service | Type | Permission declared | Use |
|---|---|---|---|
| `WakeWordService` | `microphone` | `FOREGROUND_SERVICE_MICROPHONE` | "Hey Jarvis" listening; off by default; holds the mic continuously (documented honestly in-app and in README) |
| `BubbleService` | `specialUse` | `FOREGROUND_SERVICE_SPECIAL_USE` + `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` ("Floating Jarvis bubble overlay") | Keeps the overlay bubble alive |
| `FocusSessionService` | `specialUse` | `FOREGROUND_SERVICE_SPECIAL_USE` + `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` ("Focus session countdown timer") | Focus-session countdown notification |

Verified Phase 11: each declared type matches the service's actual use,
and each has its matching permission declaration. **Human step:** Play
Console requires a foreground-service declaration naming each type and
its purpose — use the table above.

## 5. Data safety section (draft answers)

Fill the Play Console data-safety form with these answers (verify each
against the current form at submission time):

- **Data collected and shared:** Jarvis collects no data for itself and
  shares nothing with advertisers or analytics providers. The user's own
  content is sent **only** to services the user explicitly chose, for the
  feature they asked for:
  - Chat / research / drafting text → the user's chosen LLM provider
    (Claude / OpenAI / Gemini / OpenRouter / a custom provider the user
    configured). Only when the user asks Jarvis something.
  - Post text → X / Threads / LinkedIn / Instagram, only when the user
    approves a publish (official APIs).
  - Search queries → DuckDuckGo or Tavily (user's choice in Settings);
    page URLs → the site being fetched.
  - OAuth sign-in → the platform's own sign-in page (Jarvis never sees
    the platform password).
- **Data stored on-device only (never leaves the phone):** tasks, notes,
  macros, drafts, research briefs, memory entries, scheduled posts,
  focus history, trusted actions, audit log, screen-time data, calendar
  events, contacts, usage-insights events. The local database is
  encrypted (SQLCipher); API keys and OAuth tokens are Keystore-backed.
- **Encryption in transit:** HTTPS only, enforced by a network security
  config (`cleartextTrafficPermitted=false`). Custom LLM providers must
  use `https://` — plain HTTP is refused.
- **Data deletion:** Settings → "Delete all data" wipes the local
  database, all settings, stored secrets, cached files, and analytics.
  Uninstalling also removes all local data. There is no server-side
  account to delete (Jarvis has no backend).
- **Children:** the app is not directed at children.
- **Account deletion:** not applicable — no Jarvis account exists. (The
  user can revoke Jarvis's access at each LLM/social provider directly.)

## 6. Privacy policy

`PRIVACY_POLICY.md` in the project root is the plain-language policy,
accurate to the code as of Phase 11. **Human steps:**
- [ ] Fill in the developer contact details (marked `[…]` in the file).
- [ ] Publish it at a public URL and link it in Play Console (required:
      the store listing needs a privacy-policy URL, and the in-app
      Settings screen should link it too — add the link when the URL
      exists).
- [ ] Re-check it on every release that changes data handling.

## 7. Other Play requirements

- **App content / content rating:** complete the rating questionnaire.
  The app contains no user-generated content from other people (it
  publishes the user's own posts to their own accounts) and no
  user-to-user communication. Target audience: adults / general.
- **App access:** no login is needed to use Jarvis (LLM/social accounts
  are the user's own, entered by them). If Play review asks for test
  credentials, note that all features work without an account except
  LLM chat (needs any provider key) and social publishing (needs the
  user's own developer apps).
- **Advertising ID:** the app does not use the advertising ID — declare
  that in Play Console.
- **News / health / financial claims:** none. The app makes no health or
  financial claims.
- **Backup:** `android:allowBackup="false"` (Phase 11) — Android Auto
  Backup is off because the app's own export (Settings → Backup) is the
  supported path; Auto Backup would copy the encrypted database without
  its Keystore-held passphrase.
- **64-bit / App Bundle:** release is an AAB built in Phase 12; no native
  libraries are bundled by Jarvis itself (SQLCipher ships its own
  native libs via the dependency).
- **Permissions removed before release:** none outstanding — the
  declared set is the minimal set (see §3).

## 8. Pre-submission checklist (human steps)

- [ ] Re-check the required target API level (§1).
- [ ] Accessibility API declaration (§2).
- [ ] Foreground-service declarations (§4).
- [ ] Data-safety form (§5).
- [ ] Privacy policy published + linked (§6).
- [ ] Content-rating questionnaire (§7).
- [ ] Advertising-ID declaration: not used (§7).
- [ ] First real `./gradlew assembleRelease` + on-device test (Phase 12)
      — including the first live OAuth connect and first real publish
      per platform, and the Accessibility service on a real device.
- [ ] Store listing: screenshots, feature graphic, description, and the
      prominent-disclosure wording for accessibility in the listing
      text if Play asks for it.
