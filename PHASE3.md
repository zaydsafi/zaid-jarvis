# Phase 3 — Agent core (LLM clients, planner, memory) + OpenRouter & custom providers

**Status: complete, 2026-11-15.** App version `0.3.0-phase3`. 111 files total
(was 73 in Phase 2).

Phase 3 implements Master Prompt §4 (Agent Brain) plus the user's addition:
**OpenRouter and other AI-model API-key providers** — a custom provider can be
any server that speaks the OpenAI Chat Completions API.

## Scope

- Real LLM clients behind the `LlmProviderClient` contract: Claude, OpenAI,
  Gemini, OpenRouter, and generic OpenAI-compatible custom providers.
- Planner + executor loop: plan → confirm (Phase 2 gate) → execute, one step
  at a time, with retries, injection defense, offline fallback, and the kill
  switch.
- Tool registry with typed JSON schemas + a small set of safe built-in tools
  (Tier 0 only — phone control is Phase 5).
- Memory: short-term conversation context + long-term memory in Room, with a
  Memory screen for view/edit/delete.
- Context providers (plumbing + time/battery/network only).
- Real token/cost reporting, response caching, offline commands.
- Fake-LLM test harness + unit tests (see Verification).

## What was built

### `:core` — models

- `model/LlmProvider.kt` — `LlmProvider` gains `OPENROUTER` and `CUSTOM`;
  `LlmModel` gains OpenRouter presets (`anthropic/claude-sonnet-4.5`,
  `openai/gpt-5`, `google/gemini-3-pro`, `meta-llama/llama-4-scout` cheap)
  and a `CUSTOM_MODEL` placeholder; `defaultFor` / `cheapFor` updated.
- `model/CustomProviderConfig.kt` (new) — id, name, base URL, model ID;
  validation (blank fields, URL must start with `http://` or `https://` —
  `http://` allowed for local servers); `normalizedBaseUrl()`.
- `model/Pricing.kt` (new) — `Pricing` + `PricingTable`: approximate USD per
  1M input/output tokens per model, `DEFAULT_RATE` fallback for unknown and
  custom models, `estimateCostUsd`. Rates are clearly approximate — the UI
  says the cost is an estimate, not a bill.
- `model/ContextSnapshot.kt` (new) — local date/time, battery %, charging,
  online state; `formatForPrompt()` for the system prompt.
- `model/JarvisSettings.kt` — gains `customModelId` (OpenRouter free-text
  override), `customProviders: List<CustomProviderConfig>`,
  `selectedCustomProviderId`, plus `selectedCustomProvider()`,
  `resolvedModelApiId()`, `cheapModelForCurrentProvider()`.

### `:agent` — LLM clients, planner, safety (all new except the contract)

- `LlmTypes.kt` — provider-neutral types: `ChatMessage` (with toolCalls /
  toolCallId), `ToolDefinition`, `LlmToolCall`, `LlmRequest` (with
  `modelApiId`, `systemPrompt`, `tools`), `LlmResponse` (text / toolCalls /
  usage / finishReason), `LlmUsage`, `LlmProviderClient` interface.
- `clients/JsonHelpers.kt` — shared JSON parsing, `extractErrorMessage`.
- `clients/ClaudeClient.kt` — Anthropic `v1/messages`
  (`anthropic-version: 2023-06-01`), tool_use / tool_result blocks.
- `clients/OpenAiCompatibleClient.kt` — abstract base; `OpenAiClient`,
  `OpenRouterClient` (HTTP-Referer / X-Title headers), `CustomProviderClient`;
  `max_completion_tokens` for OpenAI/OpenRouter, `max_tokens` for custom.
- `clients/GeminiClient.kt` — `:generateContent`, `x-goog-api-key` header,
  functionCall / functionResponse (Gemini has no call ids — the tool name is
  the pairing key).
- The `parse*Response` functions are public so tests can parse recorded JSON
  fixtures — no real API calls in tests, ever.
- `Planner.kt` — `AgentPlanner` with `RunConfig` and the `AgentEvent` sealed
  interface (Thinking / ToolCallStarted / ToolCallFinished / InjectionFlagged
  / FinalAnswer / NeedsInput / Failed). Max 10 turns; max 3 attempts per LLM
  call and per tool step with 500ms / 1s / 2s backoff. Non-retryable errors:
  denials, expiry, 401/403, unknown tool, invalid params, biometric. The
  injection guard stops the run and asks Zaid. Offline path runs time /
  calculator commands through the gate, else an honest offline message.
  `isSimpleRequest` heuristic picks the cheap model for short plain questions.
- `InjectionGuard.kt` — regex patterns for override / extract / exfiltrate /
  money / delete / code-execution / hide / jailbreak attempts; `scan()`
  returns findings with descriptions and snippets.
- `ResponseCache.kt` — in-memory LRU (50 entries, 15-minute TTL, SHA-256
  keys); never caches tool-calling responses.
- `UsageTracker.kt` + `UsageRecorder` — cost via `PricingTable`, recorded
  into `SettingsRepository.recordUsage`.
- `ConversationContext.kt` — bounded to 20 messages; `addUser` /
  `addAssistant` / `addToolResult` (with a `toolName: result` prefix so
  providers without tool-call ids like Gemini can recover the tool name),
  `snapshot()`, `size()`, `clear()`.
- `ToolExecutor.kt` — interface + `GatedToolExecutorAdapter`, the production
  implementation delegating to `GatedToolExecutor`. **This is the only path
  from the planner to tool execution.**
- `LlmClientFactory.kt` — `clientFor(settings)` with plain-language failures
  for missing keys / invalid custom config; `modelApiIdFor(settings,
  simpleIntent)` with cheap-tier logic.
- `di/AgentModule.kt` — OkHttpClient with **no logging interceptor** (keys
  must never be logged), 30s connect / 120s read / 30s write timeouts;
  provides ResponseCache, UsageTracker, ToolExecutor adapter, AgentPlanner
  (with `isOnline` from DeviceContextProvider).
- `AgentStub.kt` deleted.

### `:tools` — schemas + built-in tools

- `ToolSchema.kt` (new) — `noParams()` / `singleStringParam()` builders,
  `validateParams` for required fields and string / number / boolean /
  integer types.
- `builtin/GetTimeTool.kt`, `builtin/CalculatorTool.kt`,
  `builtin/EchoTool.kt` (new) — all Tier 0, no permissions. The calculator
  is a hand-written parser (+ - * / % and parentheses only), never a script
  engine.
- `ToolsModule.kt` — Hilt provider for `ToolRegistry` pre-loaded with the
  built-ins (`BuiltInTools.registerAll`).
- `ToolStub.kt` — `JarvisTool` gains `inputSchema` (default
  `ToolSchema.noParams()`); `ToolRegistry` no longer has an `@Inject`
  constructor (ToolsModule is the only binding).
- `GatedToolExecutor.kt` — validates params against the schema before the
  dry-run preview, so a malformed call never reaches Zaid's approval card.

### `:data` — memory, keys, context

- `db/MemoryItem.kt` — `MemoryDao` gains `search` (LIKE over kind/key/value)
  and `getByKind`.
- `repository/MemoryRepository.kt` (new) — observe / search / save / delete /
  deleteAll, `summaryForPrompt` for the planner's system prompt.
- `security/SecureKeyStore.kt` — gains `allSecretKeys()`,
  `customApiKeyPreferenceKey(id)` (`api_key_custom_<id>`), `API_KEY_PREFIX`.
- `repository/ApiKeyRepository.kt` — rewritten to support custom-provider
  keys (`setCustomApiKey`, `hasCustomApiKey`, `getCustomApiKey`,
  `clearCustomApiKey`, `customKeyPresence` flow); `clearAllApiKeys` now
  clears all `api_key_*` keys.
- `device/DeviceContextProvider.kt` (new) — battery via
  `ACTION_BATTERY_CHANGED`, online check via ConnectivityManager,
  `snapshot()`.
- `datastore/SettingsRepository.kt` — refactored with a `readSettings(prefs)`
  helper; persists `customModelId`, `customProviders` (JSON string),
  `selectedCustomProviderId`.
- `di/DataModule.kt` — provides `MemoryRepository` and `DeviceContextProvider`.

### `:app` — chat, memory, settings UI

- `viewmodel/ChatViewModel.kt` + `screens/ChatScreen.kt` (new) — typed chat
  → planner; resolves client/model via the factory; gathers memory summary +
  device context; registers the run job with the kill switch; message list,
  input, Send, thinking indicator, Clear.
- `viewmodel/MemoryViewModel.kt` + `screens/MemoryScreen.kt` (new) —
  list/add/edit/delete/delete-all memory items with an editor dialog.
- `viewmodel/SettingsViewModel.kt` — exposes `customKeyPresence`; adds
  `addCustomProvider` / `updateCustomProvider` / `deleteCustomProvider`
  (clears its key too) / `setSelectedCustomProvider` / `setCustomModelId` /
  `setCustomApiKey`.
- `screens/SettingsScreen.kt` — provider dropdown now includes OpenRouter +
  Custom; OpenRouter free-text model field; custom-provider selector when
  CUSTOM; new "Custom providers" section (add/edit/delete cards with
  name/base URL/model + per-provider key field); API-keys section loops over
  providers **except CUSTOM** (custom keys live in the Custom providers
  section); Usage section now reports real usage with the approximate-rates
  note; "View memory" button.
- `navigation/JarvisNavGraph.kt` — `MemoryDestination`, Memory screen wired,
  `onNavigateToMemory` passed to Settings.
- `build.gradle.kts` — version `0.3.0-phase3`.
- `agent/build.gradle.kts`, `tools/build.gradle.kts`, `data/build.gradle.kts`
  — `testImplementation(libs.junit)`.

## Provider setup

See README.md ("API keys"). In short: Settings → API keys for Claude, OpenAI,
Gemini, OpenRouter (one key reaches every OpenRouter model; pick a preset or
type any model ID). Custom providers: Settings → Custom providers — name,
base URL, model ID, then the key in the provider's card. The app calls
`[base URL]/chat/completions`. Keys are never hard-coded and are stored
Keystore-encrypted; deleting a custom provider deletes its key.

## Verification

**Full Gradle build (`./gradlew assembleDebug`) still CANNOT run in this VM**
— same limitation as Phases 1–2. Tried once this phase:
`./gradlew --no-daemon -q help` fails with the same SSL/proxy-tunnelling
errors (Java HTTP clients cannot traverse the egress proxy); the Gradle
daemon dispatch failure also still applies. **Do not treat the project as
build-verified until `./gradlew assembleDebug` passes on a real machine or
in Android Studio, and no real API call has been made yet — the first live
API test must happen on Salahuddin's machine with his own API keys.**

What WAS verified (2026-11-15, in this VM) via
`~/opt/verify-pom/verify-phase3.sh` (kotlinc 2.4.21, `-jvm-target 17`,
+ Compose + serialization plugins, against the real dependency JARs and
`android.jar` API 36):

- **`:core` compiles cleanly** (67 class files, 0 errors); **49/49 unit
  tests pass** — 32 from Phases 1–2 + 17 new: 5 PricingTest, 5
  CustomProviderConfigTest, 7 LlmProviderTest (defaults/cheap tiers,
  OpenRouter free-text override, custom-provider resolution).
- **`:tools` compiles cleanly on the JVM** (15 class files, 0 errors);
  **13/13 unit tests pass** — 5 ToolSchemaTest, 8 BuiltInToolsTest
  (calculator precedence/parentheses/division-by-zero/invalid input,
  get_time formatting with a fixed clock, echo, tier-0/no-permission check).
- **All Android module sources compile cleanly** (`app`, `data`, `ui`,
  `agent`, `tools`, `plugins`): **231 class files, 0 errors**. Only the 2
  pre-existing Phase 1 deprecation warnings remain (Chat icon, `menuAnchor()`
  in EnumDropdown); the new custom-provider dropdown uses the
  non-deprecated `menuAnchor(ExposedDropdownMenuAnchorType, enabled)`
  overload. This validates syntax and types — but NOT annotation processing
  (Hilt/Room codegen), resource merging, or the Android manifest.
- **`:agent` tests: 43/43 pass** — 17 PlannerTest (simple answer, tool-call
  loop, LLM retry then success, give-up after 3 attempts, auth errors not
  retried, tool retry then success, denial never retried, injection stops the
  run, clarifying question → NeedsInput, offline time/calculator, offline
  fallback message, kill-switch cancel mid-loop, cache hit, usage recording,
  invalid tool arguments, `isSimpleRequest`), 7 InjectionGuardTest, 5
  ResponseCacheTest, 8 ProviderParsingTest (recorded JSON fixtures for
  Claude/OpenAI/Gemini: text, tool calls, missing usage, Gemini blocked
  response), 3 ConversationContextTest, 3 UsageTrackerTest. All LLM
  behaviour is driven by the deterministic `FakeLlmClient` harness — no real
  API calls, only fake keys (`"test-key"`).
- **`:data` tests: 6/6 pass** — MemoryRepositoryTest with an in-memory fake
  DAO (save/observe, update, search, delete/deleteAll, summaryForPrompt
  empty/non-empty).
- **Total: 111 unit tests pass** (49 + 13 + 43 + 6).
- **No secrets in code**: grep for API-key/password/token patterns across
  the project found nothing.

Bugs found and fixed during verification (all re-verified in the final run):
`ToolSchema.kt` was missing the `import kotlinx.serialization.json.put`
extension import; `CalculatorTool.execute` only caught
`IllegalArgumentException` while its parser throws `IllegalStateException`;
`Planner.kt` called the suspend `currentCoroutineContext()` inside a
non-suspend lambda (context now captured once outside it);
`SettingsRepository.kt` relied on a smart cast of a cross-module public
property (now a local variable).

## Known risks for the first real Gradle run

- Hilt + KSP codegen and Room schema generation are untested here — this
  phase adds a new Hilt module (`AgentModule`), new providers
  (`MemoryRepository`, `DeviceContextProvider`), and new DAOs, all of which
  only a real Gradle build exercises.
- AGP 9.5's built-in Kotlin support may interact with the explicitly applied
  `kotlin-android` plugin (carried over from Phase 1).
- The first live API call (any provider) is untested — provider response
  parsing is verified only against recorded fixtures.

## What remains

- **Phase 4** — Voice (SpeechRecognizer/TTS, wake word, "Jarvis stop"),
  chat UI polish, overlay bubble (reuses `ApprovalCard`), widget, Quick
  Settings tile.
- **Phase 5** — Phone tools (tiers 0–2) + real undo actions for reversible
  tools.
- **Phases 6–12** — Accessibility automation, research/content, social
  publishing, productivity, plugins, hardening, release build.
