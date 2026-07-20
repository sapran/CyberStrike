# Multi-Provider LLM Layer (Provider / Model Resolution)

## Purpose
The provider layer is CyberStrike's model-agnostic LLM abstraction, inherited wholesale from opencode and re-skinned (openspec/project.md §2). It loads the entire models.dev catalog into a normalized in-memory database, detects which providers have credentials (env/auth.json/config/plugin/custom loader), constructs the matching AI-SDK provider — bundling 23 factories and dynamically npm-installing the rest — and resolves free-form `provider/model` strings to concrete language models with per-provider auth quirks (Anthropic OAuth subscription, Bedrock, Copilot, Vertex, GitLab, Cloudflare, and the cyberstrike/zenmux gateways). It exists so the whole pentest agent loop is decoupled from any single model or vendor.

## Requirements

### Requirement: Models.dev Catalog Acquisition and Air-Gap Overrides
The system SHALL load the model catalog from models.dev at startup, cache it locally, refresh it hourly, and honor air-gap override environment variables.

#### Scenario: Default online fetch and hourly refresh
- **WHEN** no override env var is set and the local cache is absent
- **THEN** Data() fetches `${url()}/api.json` where url() defaults to "https://models.dev" (models.ts:120,133); on module load ModelsDev.refresh() runs once immediately and again every 60 minutes via an unref'd setInterval (models.ts:161-168), writing to the cache and resetting the lazy (models.ts:142-158)

#### Scenario: CYBERSTRIKE_MODELS_PATH pre-seeded offline catalog
- **WHEN** CYBERSTRIKE_MODELS_PATH is set
- **THEN** Data() reads the catalog JSON from that path instead of the default cache file (models.ts:124), so an operator can supply a local catalog with no network access

#### Scenario: CYBERSTRIKE_MODELS_URL mirror
- **WHEN** CYBERSTRIKE_MODELS_URL is set
- **THEN** both the lazy fetch and the periodic refresh target `${CYBERSTRIKE_MODELS_URL}/api.json` (models.ts:120,133,144)

#### Scenario: CYBERSTRIKE_DISABLE_MODELS_FETCH air-gap
- **WHEN** CYBERSTRIKE_DISABLE_MODELS_FETCH is truthy and no cache file/snapshot resolves
- **THEN** Data() returns an empty catalog `{}` (models.ts:132) and the startup refresh plus hourly interval are skipped entirely (models.ts:161) — no outbound request is made

### Requirement: Model-Agnostic Catalog-to-Database Conversion
The system SHALL convert every models.dev provider and model into a normalized database with no model hardcoded into the catalog path, and resolve model identifiers as free-form provider/model strings.

#### Scenario: Whole catalog is normalized
- **WHEN** the provider state initializes
- **THEN** the entire models.dev map is converted via fromModelsDevProvider/fromModelsDevModel (provider.ts:749-758, 669-747), so any provider/model present in the catalog becomes selectable — capabilities, cost, limits, and modalities are copied through generically

#### Scenario: parseModel splits on first slash
- **WHEN** a model string like "amazon-bedrock/anthropic.claude-sonnet-4-5" is parsed
- **THEN** parseModel splits on the FIRST "/" into providerID + the remainder as modelID (provider.ts:1419-1425), so the provider prefix only disambiguates the same model across vendors; `model:` is optional everywhere and a free string (openspec/project.md §4.5)

#### Scenario: GitHub Copilot Enterprise is synthesized
- **WHEN** the catalog contains a github-copilot provider
- **THEN** a cloned github-copilot-enterprise provider is derived from it with re-pointed model providerIDs (provider.ts:787-798)

### Requirement: Provider Enablement and Credential Detection
The system SHALL enable a provider only when it has a usable credential (env var, auth.json entry, config api URL, plugin loader, or an autoloading custom loader) and SHALL respect enabled/disabled allowlists.

#### Scenario: Environment variable key
- **WHEN** one of a provider's declared env vars is present in the environment
- **THEN** the provider is merged with source:"env" and the key captured when there is a single env var (provider.ts:897-906)

#### Scenario: auth.json API key
- **WHEN** Auth.all() returns an entry of type "api" for a provider
- **THEN** the provider is merged with source:"api" and that key (provider.ts:909-917); auth.json is a discriminated union of oauth/api/wellknown parsed per-entry (auth/index.ts:34-56)

#### Scenario: Allowlist / denylist filtering
- **WHEN** config.enabled_providers is set and omits a provider, or config.disabled_providers includes it
- **THEN** the provider is excluded from the database (provider.ts:766-773 isProviderAllowed, applied at 1002-1006)

#### Scenario: Empty provider dropped
- **WHEN** a provider has zero models remaining after all filtering
- **THEN** it is deleted from the provider map (provider.ts:1035-1038)

### Requirement: Default and Small Model Resolution
The system SHALL resolve the default model and the small (auxiliary) model from config first, then recency, then a hardcoded capability-priority list, with graceful fallbacks.

#### Scenario: defaultModel precedence
- **WHEN** defaultModel() is called
- **THEN** config.model wins if set (parseModel); otherwise the first still-available entry from state/model.json `recent`; otherwise the top-sorted model of the first configured/available provider, throwing if none exist (provider.ts:1392-1417)

#### Scenario: getSmallModel config override
- **WHEN** config.small_model is set
- **THEN** it is parsed and returned directly, bypassing the priority search (provider.ts:1319-1322)

#### Scenario: getSmallModel priority search
- **WHEN** no small_model is configured
- **THEN** the first model matching the priority list [claude-haiku-4-5, claude-haiku-4.5, 3-5-haiku, 3.5-haiku, gemini-3-flash, gemini-2.5-flash, gpt-5-nano] is chosen; github-copilot prepends free gpt-5-mini/claude-haiku-4.5, cyberstrike restricts to gpt-5-nano, and Bedrock applies cross-region-prefix selection; final fallback is cyberstrike/gpt-5-nano (provider.ts:1316-1379)

### Requirement: Model Lookup with Fuzzy Suggestions
The system SHALL look up a provider/model pair and, on a miss, raise ModelNotFoundError carrying up to three fuzzy-matched suggestions.

#### Scenario: Successful lookup
- **WHEN** both providerID and modelID exist in the database
- **THEN** getModel returns the resolved Model info object (provider.ts:1163-1181)

#### Scenario: Unknown provider
- **WHEN** the providerID is not in the database
- **THEN** ModelNotFoundError is thrown with up to 3 fuzzysort suggestions over the available provider ids (provider.ts:1166-1170)

#### Scenario: Unknown model for known provider
- **WHEN** the modelID does not exist under a known provider
- **THEN** ModelNotFoundError is thrown with up to 3 fuzzysort suggestions over that provider's model ids (provider.ts:1173-1179)

### Requirement: Bundled Providers and Dynamic NPM Install
The system SHALL construct each provider's SDK from a bundled factory keyed by api.npm, dynamically installing the npm package when it is not bundled, and memoize the constructed SDK.

#### Scenario: Bundled factory
- **WHEN** model.api.npm matches a key in BUNDLED_PROVIDERS (23 factories: anthropic, openai, google, bedrock, azure, vertex, openrouter, xai, mistral, groq, deepinfra, cerebras, cohere, gateway, togetherai, perplexity, vercel, gitlab, alibaba, venice, github-copilot, openai-compatible, vertex/anthropic)
- **THEN** the bundled factory builds the SDK in-process with no install (provider.ts:1126-1135; bundled-providers.ts:40-65)

#### Scenario: Dynamic install of unbundled provider
- **WHEN** api.npm is neither bundled nor a file:// path
- **THEN** BunProc.install(npm, "latest") installs it into the global cache node_modules (bun/index.ts:64-133), the module is imported, and the first export whose name starts with "create" is used as the factory (provider.ts:1137-1153)

#### Scenario: Local file provider
- **WHEN** api.npm starts with "file://"
- **THEN** the module is imported directly with no npm install (provider.ts:1138-1143)

#### Scenario: SDK memoization
- **WHEN** the same providerID + npm + options are requested again
- **THEN** the xxHash32-keyed cache returns the existing SDK instance instead of reconstructing it (provider.ts:1076-1078,1132-1133)

### Requirement: Per-Provider Custom Auth Loaders
The system SHALL apply provider-specific authentication and options through CUSTOM_LOADERS (18 providers) that decide autoload eligibility and inject options and per-provider model-loader adapters.

#### Scenario: Anthropic OAuth subscription path
- **WHEN** stored anthropic auth is type oauth, or an sk-ant-oat* key is present
- **THEN** a custom @anthropic-ai/sdk LanguageModelV2 is used instead of @ai-sdk/anthropic — Bearer OAuth with auto-refresh, subscription betas, and JSON metadata.user_id (device/account/session) for included-quota parity (provider.ts:84-131; anthropic-oauth.ts:161-234); plain sk-ant-api keys fall through to the x-api-key path with claude-code betas (provider.ts:122-130)

#### Scenario: Amazon Bedrock credential chain and region prefixing
- **WHEN** a profile, access key, bearer token, web-identity file, or container credential is available
- **THEN** the provider autoloads with region precedence (config > AWS_REGION > us-east-1) and per-region cross-inference model prefixing (us./eu./apac./jp./au.); with no credential it does not autoload (provider.ts:212-361)

#### Scenario: Gateway providers (cyberstrike / zenmux / openrouter / vercel / cerebras)
- **WHEN** the cyberstrike gateway has no key
- **THEN** only zero-cost models are retained and apiKey "public" is used (provider.ts:132-153); zenmux/openrouter/vercel inject cyberstrike.io Referer + X-Title headers and cerebras a 3rd-party-integration header (provider.ts:362-383,448-458,562-571)

#### Scenario: Vertex / GitLab / Cloudflare conditional autoload
- **WHEN** GOOGLE_CLOUD_PROJECT (Vertex), a GitLab token/OAuth, or CLOUDFLARE_ACCOUNT_ID plus gateway id are present
- **THEN** the provider autoloads with instance-specific options and a getModel adapter (Vertex regional endpoints, GitLab agenticChat with Duo feature flags, Cloudflare Workers-AI/AI-Gateway); Cloudflare AI Gateway throws if no API token is supplied (provider.ts:384-561)

### Requirement: Model Status and Config Filtering
The system SHALL filter models out of the exposed catalog by lifecycle status, config blacklist/whitelist, and experimental flags.

#### Scenario: Status gating
- **WHEN** a model's status is "deprecated"
- **THEN** it is removed; "alpha" models are removed unless CYBERSTRIKE_ENABLE_EXPERIMENTAL_MODELS is set (provider.ts:1014-1015; flag.ts:18)

#### Scenario: Config blacklist / whitelist
- **WHEN** a provider config has a blacklist containing the modelID, or a whitelist excluding it
- **THEN** the model is deleted from that provider (provider.ts:1016-1020)

#### Scenario: Hardcoded special-case removals
- **WHEN** the modelID is gpt-5-chat-latest, or openrouter's openai/gpt-5-chat
- **THEN** it is deleted regardless of other config (provider.ts:1012-1013)

## Notes
- README '150+ AI providers / 5,300+ models' claim (README.md:40,95,144) is ACCURATE and actually conservative: the live catalog cache (~/.cache/cyberstrike/models.json) contains 167 providers and 5,690 models. These counts are NOT hardcoded in-repo — they are a property of the external models.dev catalog fetched at runtime, so source alone cannot 'prove' them; the cache confirms them. The repo itself only bundles 23 SDK factories (bundled-providers.ts) and 18 CUSTOM_LOADERS (provider.ts:84-572). README.md:97's '23 bundled SDK providers' figure matches exactly.
- No models-snapshot.ts file is committed anywhere in the repo, so the Data() build-time-snapshot fallback branch (models.ts:127-131) is dead in this checkout — resolution falls through path>cache>fetch. A production build is expected to generate it.
- Provenance: this whole layer is inherited wholesale from sst/opencode and re-skinned (openspec/project.md §1–§2). CyberStrike-specific additions on top of opencode's design: the OPENCODE_ -> CYBERSTRIKE_ env-var prefix rename (flag.ts), the metered 'cyberstrike' SaaS gateway provider with public/zero-cost-model fallback (provider.ts:132-153), and cyberstrike.io Referer/X-Title headers on the gateway providers.
- The 'cyberstrike' provider is the SaaS control-plane gateway (OpenAuth/Stripe-metered, openspec/project.md §2); with no key it exposes only free (cost.input===0) models via apiKey "public". 'zenmux' is the ZenMux gateway. Neither is a distinct model family — both are OpenAI-compatible routing gateways that just inject attribution headers.
- getModelDescriptor (provider.ts:1215-1299) serializes model access (npm/apiKey/authToken/baseURL/headers) so the standalone hackbrowser worker subprocess can reconstruct a LanguageModel without importing the Provider system — reinforcing the model-agnostic, factory-by-api.npm design.
- Model-agnostic design CONFIRMED: parseModel is a dumb split-on-first-slash (provider.ts:1419-1425), ModelId is a free string, `model:` is optional and inherited everywhere; catalog->Model conversion (fromModelsDevModel) copies capabilities generically with no per-model branching. The only model-name string matching is in soft ranking/selection heuristics (getSmallModel priority, sort()) and a couple of hardcoded removals (gpt-5-chat-latest), not in the core resolution path.

## Key Files
- `packages/cyberstrike/src/provider/provider.ts` — Core provider system: CUSTOM_LOADERS (18 per-provider auth loaders), state() database assembly, getSDK dynamic install + bundled dispatch, getModel/getSmallModel/defaultModel/parseModel resolution, model filtering
- `packages/cyberstrike/src/provider/models.ts` — ModelsDev catalog: Data() lazy load order (path>cache>snapshot>fetch), url()/CYBERSTRIKE_MODELS_URL, startup + hourly refresh, air-gap disable
- `packages/cyberstrike/src/provider/bundled-providers.ts` — BUNDLED_PROVIDERS: the 23 api.npm->createX SDK factories bundled into the binary (single source of truth, also used by hackbrowser worker)
- `packages/cyberstrike/src/flag/flag.ts` — Env override flags: CYBERSTRIKE_MODELS_URL/_PATH (58-59), CYBERSTRIKE_DISABLE_MODELS_FETCH (20), CYBERSTRIKE_ENABLE_EXPERIMENTAL_MODELS (18)
- `packages/cyberstrike/src/auth/index.ts` — auth.json store: discriminated union oauth/api/wellknown, get/all/set/remove with 0600 mode
- `packages/cyberstrike/src/auth/anthropic-oauth.ts` — Anthropic Pro/Max OAuth: PKCE authorize/exchange, getValidAnthropicToken auto-refresh, sk-ant-oat handling, account-uuid profile fetch
- `packages/cyberstrike/src/provider/anthropic-subscription-model.ts` — Custom LanguageModelV2 wrapping the official @anthropic-ai/sdk for subscription parity (SUBSCRIPTION_BETAS, AGENT_SDK_PREFIX) — invoked by the anthropic custom loader
- `packages/cyberstrike/src/bun/index.ts` — BunProc.install: locked `bun add --force --exact` into the global cache, version resolution/outdated check — backs dynamic non-bundled provider install
- `packages/cyberstrike/src/provider/local.ts` — Runtime local-provider discovery: hits an OpenAI-compatible /models endpoint and writes a config provider (Ollama/LM Studio/vLLM)
