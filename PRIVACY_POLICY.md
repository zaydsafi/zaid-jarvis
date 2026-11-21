# Privacy Policy — Zaid Jarvis

**Last updated: 16 November 2026 (app version 0.11.0-phase11)**

> **[Developer contact — fill in before publishing.]** This policy is
> published with the Zaid Jarvis app. To contact the developer about
> privacy, write to [email / website — add before release].

Zaid Jarvis ("Jarvis", "the app") is a personal AI assistant for Android.
This policy explains, in plain language, what data the app stores, what
leaves your phone, and how to delete everything. Short version: **your
data lives on your phone, encrypted. Nothing is sent anywhere except to
services you explicitly chose, for things you explicitly asked for.
There are no ads, no analytics SDKs, and no Jarvis account or server.**

## 1. Data that stays on your phone (never leaves it)

Everything Jarvis stores locally is kept in an encrypted database
(SQLCipher) on your device, or in Android's Keystore-backed encrypted
storage. None of it is uploaded, synced, or shared:

- **Your content:** tasks, notes, macros, drafts, research briefs,
  memory entries, scheduled posts, focus-session history, trusted
  actions, and the audit log (the record of what Jarvis did and what
  you approved).
- **Your settings:** theme, language, provider/model choices, quiet
  hours, digest/wind-down preferences, and plugin states.
- **Your secrets:** LLM API keys, the Tavily search key, social OAuth
  client IDs/secrets and access/refresh tokens, and the database
  encryption passphrase. These live in Keystore-backed encrypted
  storage and are never written to the database, the backup file, the
  audit log, or any log output.
- **Usage insights (on-device analytics):** counts only — which tools
  were used, publish outcomes, focus-session duration buckets. No
  message text, note text, URLs, or searches are ever recorded; free
  text is structurally impossible to store here. You can turn insights
  off in Settings ("Usage insights") and wipe the stored events with
  "Clear analytics data".
- **Screen-time data** (if you grant usage access), **calendar events**
  (read-only), and **contact details** are read on your device to answer
  your requests and are never uploaded anywhere.
- **Screen content** seen by the optional Accessibility automation is
  processed in memory to find buttons and text you asked about. It is
  never stored, never logged, and never sent anywhere. Jarvis never
  reads or stores passwords, and never acts on screens with password
  fields.

Android Auto Backup is disabled for this app, so none of the above is
copied to a cloud backup by the system. The app's own backup
(Settings → Backup) exports a JSON file **you** choose where to save;
it never includes API keys, OAuth tokens, or the database passphrase.

## 2. Data that leaves your phone — only to services you chose

Jarvis has no server of its own. Your content is sent only to the
services you configured, and only when you ask for the feature:

| When you… | What is sent | To whom |
|---|---|---|
| Chat with Jarvis, research a topic, draft a post, or ask for a morning briefing | Your message, the conversation context, and the tool descriptions Jarvis may use | Your chosen LLM provider: Anthropic (Claude), OpenAI, Google (Gemini), OpenRouter, or a custom OpenAI-compatible provider you configured |
| Search the web | Your search query | DuckDuckGo (default, no key) or Tavily (if you entered a key) |
| Fetch a web page | The page URL (and a normal browser user-agent) | The website hosting that page |
| Publish or schedule a post | The post text (and image URL, for Instagram) | X, Threads, LinkedIn, or Instagram, via their official APIs — only after you approve the exact post |
| Connect a social account | OAuth sign-in happens on the platform's own page | X / Threads / LinkedIn / Instagram (Meta). Jarvis never sees your platform password |

All of these connections use HTTPS only. The app refuses plain-HTTP
connections, and custom LLM providers must use `https://` URLs.

**Third-party privacy policies** (these services process your data under
their own policies once you send it to them):
- Anthropic: https://www.anthropic.com/legal/privacy
- OpenAI: https://openai.com/policies/privacy-policy
- Google (Gemini): https://policies.google.com/privacy
- OpenRouter: https://openrouter.ai/privacy
- Tavily: https://tavily.com/privacy-policy
- DuckDuckGo: https://duckduckgo.com/privacy
- X: https://x.com/en/privacy
- Meta (Threads / Instagram): https://www.facebook.com/privacy/policy
- LinkedIn: https://www.linkedin.com/legal/privacy-policy

## 3. Data Jarvis does NOT collect

- No location data (the app has no location feature and requests no
  location permission).
- No photos, videos, or audio recordings (the camera permission is used
  only for the flashlight; voice input is transcribed by Android's
  speech recognizer and the audio is not stored by Jarvis).
- No advertising ID, no device fingerprinting, no third-party analytics
  or crash-reporting SDKs.
- No data about other people, except contact names you ask about (used
  on-device only, never uploaded) and calendar event details from your
  own calendar (read-only, on-device only).

## 4. Permissions and why each is needed

Each permission is requested only when its feature is first used, never
at app start. Denying a permission only disables that feature.

- **Microphone** — voice input and the optional "Hey Jarvis" wake word.
- **Camera** — flashlight only. Jarvis never takes photos or video.
- **Contacts** — resolving names ("text Alice") for calls and messages.
- **Calendar (read-only)** — showing today's schedule and the morning
  briefing. Jarvis never creates or changes events.
- **Usage access** — the optional screen-time feature (opt-in, granted
  in system Settings). All usage data stays on-device.
- **Notifications** — reminders, the daily digest, focus results, and
  publish results.
- **Display over other apps** — the optional floating bubble.
- **Accessibility** — the optional automation feature (tap/type/scroll
  in other apps after your approval). See the in-app disclosure shown
  when you enable it; it is off by default and you enable it yourself
  in Android Settings.
- **Do Not Disturb access / Modify system settings** — focus mode and
  wind-down (DND) and screen brightness, only after your approval.

## 5. How long data is kept

- **Audit log:** the most recent 10,000 entries are kept; older entries
  are pruned automatically (weekly). You can delete entries or clear
  the log any time in the Audit Log screen.
- **Usage-insights events:** events older than 90 days are pruned
  automatically. "Clear analytics data" wipes them immediately.
- **Everything else** (tasks, notes, drafts, research, memory, macros)
  is kept until you delete it.
- **Backups you export** are files you own — Jarvis cannot delete a
  file you saved elsewhere.

## 6. Deleting your data

- **Settings → "Delete all data"** wipes the local database, all
  settings, stored API keys and OAuth tokens, cached files, and
  analytics — after a biometric (or typed) confirmation. This is the
  complete wipe.
- **Settings → "Delete all secrets"** removes just the stored API keys,
  OAuth tokens, and credentials.
- **Uninstalling the app** removes all local data (there is no cloud
  copy, because Auto Backup is disabled and Jarvis has no server).
- Data already sent to a third party (an LLM provider, a social
  platform) is governed by that service's own policy — delete it there
  (for example, revoke Jarvis's access in the provider's settings, or
  delete a published post in the platform's own app).

## 7. Children

Jarvis is not directed at children under 13, and does not knowingly
collect data from children.

## 8. Changes to this policy

If a future version changes what data Jarvis stores or sends, this
policy will be updated with the new version and the "Last updated" date
above will change. Material changes will be noted in the release notes.

## 9. Contact

[Developer contact — add an email address or website before release.]
