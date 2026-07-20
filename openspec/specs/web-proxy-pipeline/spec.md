# Browser-Driven Web-Testing Pipeline (hackbrowser crawl → ingest → proxy orchestrator → analyzer/testers)

## Purpose
An end-to-end web-application testing pipeline that autonomously drives a real browser, captures live HTTP traffic with UI context, and feeds each unique endpoint through an LLM agent fan-out. `hackbrowser` (packages/hackbrowser) is a Playwright-based security browser + AI crawler that BFS-navigates a target, intercepts every in-scope request/response, and POSTs it to CyberStrike's `/session/ingest`. The server normalizes and de-duplicates each request, then dispatches it to a `proxy-agent` orchestrator which delegates architecture extraction to a `proxy-analyzer` and vulnerability testing to 8 specialist `proxy-tester-*` subagents. The design enforces separation of concerns via per-agent permission rulesets, a static WSTG-skill embedding step, and a mandatory 3-gate confirmation protocol before any finding is recorded.

## Requirements

### Requirement: Autonomous Browser Crawl Engine
The system SHALL drive a Playwright/Chromium browser through an LLM-guided BFS crawl of an in-scope target, intercepting every non-static in-scope HTTP request/response as a CapturedRequest.

#### Scenario: BFS page loop
- **WHEN** runCrawl invokes run(config) for a single-credential target
- **THEN** the engine launches Chromium, seeds pageQueue with the target URL, and loops while pageQueue is non-empty and pagesExplored < maxSteps (default 50), exploring each page via explorePageWithAI and enqueueing newly discovered in-scope links (packages/hackbrowser/src/agent.ts:2082-2200)

#### Scenario: Request interception + scope filter
- **WHEN** the browser emits a 'request' event
- **THEN** setupRequestInterceptor drops static assets (SKIP_EXTENSIONS) and out-of-scope hosts, then builds a raw HTTP request string, attaches the captured response (body truncated at 500KB) and pending UI/trigger context, and forwards a CapturedRequest to onCapture (packages/hackbrowser/src/agent.ts:387-465, capture.ts:289-314)

#### Scenario: DOM/accessibility scan feeds navigation
- **WHEN** a page is loaded
- **THEN** scanner.ts collectElements enumerates up to MAX_ELEMENTS=50 accessibility-role elements (deduped to MAX_PER_TEMPLATE=5 per numbered cluster) and generateFingerprint lets unchanged re-visited pages skip LLM calls (packages/hackbrowser/src/scanner.ts:1-40, agent.ts:2283-2293)

### Requirement: Capture Enrichment and Ingest Transport
The system SHALL enrich each captured request with UI-form context, trigger element, and page/role signals, then transmit it to the CyberStrike server over HTTP loopback to /session/ingest.

#### Scenario: UI-context snapshot on mutating requests
- **WHEN** a mutating (non-GET/HEAD/OPTIONS) request fires during an action window
- **THEN** snapshotPageUI captures form fields (name/type/readonly/disabled/hidden/validation) scoped to the trigger's form/dialog, and correlateWithUI marks request params absent from the UI as hiddenParams (packages/hackbrowser/src/capture.ts:17-283, 324-335, agent.ts:405-415)

#### Scenario: Ingest POST with enrichment fields
- **WHEN** the 500ms drain loop processes a queued capture (non-dry-run)
- **THEN** buildIngestPayload assembles {text: raw, scheme, response, ui_context, trigger_element, element_roles, page_url, page_visited_by} and sendIngest POSTs it to `${serverUrl}/session/ingest` with Basic auth (packages/hackbrowser/src/ingest.ts:61-118, agent.ts:2121-2131)

#### Scenario: Credential registration + header sync
- **WHEN** auth headers change across captures in a credentialed crawl
- **THEN** registerCredential POSTs the label to get a DB UUID and syncCredentialHeaders PATCHes the credential record so captured cookies/tokens reach the server (packages/hackbrowser/src/ingest.ts:173-214, agent.ts:2110-2116, 2155-2160)

### Requirement: Server Ingest Normalization and Dispatch
The system SHALL normalize each ingested HTTP request, de-duplicate it against prior captures, record per-credential observations, and enqueue new endpoint shapes for LLM processing defaulting to the proxy-agent orchestrator.

#### Scenario: Normalize + dedup
- **WHEN** POST /session/ingest receives a parseable HTTP request
- **THEN** runNormalize (Normalize.run) parses/templatizes it; if Request.exists matches an existing key it records an Observation and returns 202 {skipped:true} without any LLM call, otherwise Request.add inserts a new row (packages/cyberstrike/src/server/routes/session.ts:1157-1235)

#### Scenario: Enqueue to orchestrator
- **WHEN** a new endpoint shape is inserted
- **THEN** IngestQueue.enqueue schedules SessionPrompt.prompt with agent = body.agent ?? "proxy-agent" and excludeHistory:true, so each endpoint is processed in isolation by the orchestrator (packages/cyberstrike/src/server/routes/session.ts:1279-1310)

#### Scenario: Per-session serialization
- **WHEN** multiple ingests arrive for one session
- **THEN** IngestQueue chains tasks on a per-session promise chain (s.chain = next) with pause/resume, serializing LLM dispatch one request at a time per session (packages/cyberstrike/src/session/ingest-queue.ts:63-83)

### Requirement: Subprocess Launcher and Run Lifecycle
The system SHALL run hackbrowser as an isolated worker subprocess launched/stopped/observed via session HTTP routes, keeping Playwright out of the main binary and streaming crawl state into HackbrowserStatus.

#### Scenario: Launch spawns worker, returns immediately
- **WHEN** POST /:sessionID/hackbrowser/launch is called
- **THEN** launchHackbrowser guards re-entrance (one run per session), prepares WorkerOptions, Bun.spawns hackbrowser-worker.js with stdin/stdout IPC pipes, sets HackbrowserStatus phase="starting", and returns a KickOffResult without waiting (packages/cyberstrike/src/tool/hackbrowser-launcher.ts:482-540, server/routes/session.ts:422-478)

#### Scenario: IPC relay updates status
- **WHEN** the worker emits log/event/result/error JSON lines
- **THEN** backgroundRun relays logs, forwards CSEvents to HackbrowserStatus.handle (incrementing pagesExplored/capturedEndpoints), and on result/crash sets phase completed/failed, computes cost, and releases the activeRuns slot (packages/cyberstrike/src/tool/hackbrowser-launcher.ts:290-464)

#### Scenario: Stop is a graceful abort
- **WHEN** POST /:sessionID/hackbrowser/stop is called
- **THEN** stopHackbrowser writes {type:"abort"} to the worker's stdin (returns false if no active run); the crawl loop checks config.signal.aborted at the next page boundary and exits gracefully to phase=completed (packages/cyberstrike/src/tool/hackbrowser-launcher.ts:552-558, agent.ts:2204-2207)

#### Scenario: Status bootstrap route
- **WHEN** GET /hackbrowser/status is called
- **THEN** HackbrowserStatus.list returns the per-session run-state map (phase, counters, errors) for the TUI sidebar; sessions with no run are omitted (packages/cyberstrike/src/server/routes/session.ts:511-534)

### Requirement: Orchestrator Delegation-Only Permission Posture
The system SHALL run proxy-agent as a pure router whose permission ruleset denies everything except task delegation and read-only context tools, structurally preventing it from testing, writing findings, or recording intel itself.

#### Scenario: Deny-all-then-allowlist
- **WHEN** the proxy-agent ruleset is built
- **THEN** it is `"*": "deny"` plus explicit allows for only task, question, and read-only tools (web_get_session_context, web_get_detail, web_get_vulnerabilities, web_get_vuln_detail, get_coverage_notes, methodology_status, scope_check) — report_vulnerability/add_intel/web_write are never offered to the model (packages/cyberstrike/src/agent/agent.ts:462-484)

#### Scenario: Analyzer-first blocking dispatch
- **WHEN** the orchestrator receives a request
- **THEN** its prompt mandates dispatching proxy-analyzer ALONE and waiting for completion before any tester (testers depend on the analyzer's extracted objects/roles/IDs), then re-reading web_get_session_context and launching testers in parallel (packages/cyberstrike/src/agent/prompt/orchestrator/web-proxy-agent/prompt.txt:124-128, 297-309, 497-499)

#### Scenario: Runs on full model tier
- **WHEN** the orchestrator is instantiated
- **THEN** it deliberately omits useSmallModel (routing quality matters), while only proxy-analyzer sets useSmallModel:true to run cheap on the provider's small tier via Provider.getSmallModel (packages/cyberstrike/src/agent/agent.ts:456-459, 493, session/prompt.ts:372-373)

### Requirement: Static WSTG Skill Embedding for Offensive Testers
The system SHALL statically embed per-specialty WSTG skill content into each vulnerability tester's prompt at agent-registry build time, stripping defensive sections so testers need no runtime skill-tool access.

#### Scenario: loadVulnAgent embeds skills
- **WHEN** the agent registry initializes a proxy-tester
- **THEN** loadVulnAgent resolves each named skill via Skill.get, wraps the content in <skill name=...> blocks under an '## Embedded Skill References' section, and prepends the shared common-prompt.txt (packages/cyberstrike/src/agent/agent.ts:82-116, 526-560)

#### Scenario: stripDefensiveSections filters content
- **WHEN** skill content is embedded
- **THEN** stripDefensiveSections removes the Remediation, Risk Assessment, CWE Categories, References, and Checklist H2 sections via regex, keeping only offensive methodology (packages/cyberstrike/src/agent/agent.ts:70-78)

#### Scenario: Testers have no skill tool
- **WHEN** a tester's permission ruleset is applied
- **THEN** the vulnAgentPermission is `"*": "deny"` with a small allowlist that does NOT include the skill tool, so embedding at startup is the only methodology channel (packages/cyberstrike/src/agent/agent.ts:57-64, 563-585)

### Requirement: Three-Gate Confirmation Protocol
The system SHALL require every vulnerability tester to satisfy a baseline -> exploit -> diff protocol against requests it actually sends before recording a finding, treating the captured response only as an input baseline.

#### Scenario: Mandatory gates before report
- **WHEN** a tester prepares to call report_vulnerability
- **THEN** the common prompt requires Gate 1 (baseline request/response), Gate 2 (attack request/response), and Gate 3 (a MEASURABLE diff), and forbids reporting when baseline and exploit are indistinguishable (packages/cyberstrike/src/agent/prompt/vuln/common-prompt.txt:80-99, 162-173)

#### Scenario: Captured response is baseline, not a test
- **WHEN** a tester reads the prepended ## Response
- **THEN** the prompt states every recorded verdict must come from a request the tester itself sent with modified input; reading the captured response is not a test (packages/cyberstrike/src/agent/prompt/vuln/common-prompt.txt:60, 119-137)

#### Scenario: Coverage note after send-and-compare
- **WHEN** a tester finishes its class on an endpoint
- **THEN** it calls record_coverage_note exactly once (scope wide for deployment-wide classes keyed to origin, local for per-route classes keyed to method+path) only after sending real requests, so future dispatches can skip already-covered app-wide classes (common-prompt.txt:126-159, orchestrator prompt.txt:154-173)

### Requirement: Tester Permission Gating and Request-Context Injection
The system SHALL gate each tester to a fixed capability allowlist (with an extra destructive-primitive denylist for the injection tester) and auto-prepend the current request's raw HTTP, credential, and access context to every dispatched subagent prompt.

#### Scenario: Uniform tester allowlist
- **WHEN** any of the 8 proxy-tester-* agents runs
- **THEN** it gets vulnAgentPermission: deny-all plus bash, webfetch, read-only web_get_* context tools, report_vulnerability/triage_vulnerability, and methodology tools (add_intel, update_vrt_check, record/get_coverage_note, scope_check, attack_script) (packages/cyberstrike/src/agent/agent.ts:563-585, 630-717)

#### Scenario: Injection defense-in-depth denylist
- **WHEN** the injection tester attempts a bash command
- **THEN** injectionAgentPermission (merged AFTER the user ruleset so it cannot be loosened) denies destructive SQL DDL, SQL-to-file/RCE primitives, and dangerous sqlmap flags as a category-level guard alongside prompt rules (packages/cyberstrike/src/agent/agent.ts:587-627, 663-673)

#### Scenario: prependRequestContext wiring
- **WHEN** the orchestrator dispatches an analyzer/tester via the Task tool
- **THEN** because the agent sets prependRequestContext:true, task.ts injects ## Current request, coverage block, ## Credential Context, ## Access Context (UI enrichment when source is hackbrowser), and capped raw request/response into the subagent prompt (packages/cyberstrike/src/tool/task.ts:166-240, agent.ts:495, 636-715)

#### Scenario: Soft turn budget
- **WHEN** a proxy-tester-* has no explicit steps config
- **THEN** the registry sets steps=50 (a soft cap that forces a text wrap-up) to leave headroom for multi-step verification and reporting (packages/cyberstrike/src/agent/agent.ts:767-779)

## Notes
- Tester count verified EXACTLY 8 in packages/cyberstrike/src/agent/agent.ts (loadVulnAgent calls 526-560, registrations 630-717): idor, authz, mass-assignment, injection, authn, business-logic, ssrf, file-attacks. Plus 1 orchestrator (proxy-agent) and 1 analyzer (proxy-analyzer) = 10 proxy pipeline agents total. The task framing's '8 proxy-tester-*' is accurate.
- Provenance: repo is a fork of sst/opencode, but the whole packages/hackbrowser package and the proxy pipeline (proxy-agent/analyzer/testers, ingest normalization, hackbrowser-launcher) are CyberStrike additions, not upstream opencode. The Agent namespace/permission-merge machinery in agent.ts derives from opencode but the security-agent roster and rulesets are CyberStrike-specific.
- packages/hackbrowser/README.md is standalone-CLI-centric and slightly stale relative to the code: it says 'AI navigation (Claude -> ...)' and tells users to set ANTHROPIC_API_KEY, but the engine is provider-agnostic (api.ts resolveModel / Provider.defaultModel/getModelDescriptor); its 'Project Structure' list omits scanner.ts, state.ts, executor.ts, scope.ts, and the panel/ telemetry code. Treat the README as a quickstart, not a spec.
- Two different response-size limits exist at different layers and are easy to conflate: the browser interceptor truncates the captured response body at 500KB (agent.ts:429), while the prompt-rendering guidance quoted to agents says JSON <=100KB shown in full else [TRUNCATED] (common-prompt.txt:56, analyzer prompt.txt:36). Not a contradiction - capture layer vs prompt layer.
- Turn budget history is documented in-code: the proxy-tester step cap was RAISED from 8 to 50 because 8 was below a single tester's natural ~12-turn length, causing confirmed findings to never reach report_vulnerability (agent.ts:767-779). This is a soft cap (forces text wrap-up), overridable by user config.
- Subprocess isolation is deliberate architecture: the main cyberstrike binary has ZERO Playwright references; hackbrowser runs only inside hackbrowser-worker.js (spawned via Bun.spawn) to avoid a Bun --compile startup crash (hackbrowser-launcher.ts:18-22). Model credentials that live in in-process fetch closures (non-Anthropic OAuth providers) cannot cross the IPC boundary, so the launcher fails fast for them (hackbrowser-launcher.ts:196-206).
- Asymmetric failure reporting: hackbrowser SUCCESS stays sidebar-only (HackbrowserStatus), while FAILURE writes a synthetic user message into the session so the LLM sees it on the next prompt - deliberately, to avoid nudging the model into polling loops (hackbrowser-launcher.ts:251-282, 390-392).
- Dedup is by normalized endpoint shape, so each endpoint reaches the orchestrator only once; per-credential values on already-known endpoints are still captured via Observation.observe on every ingest (including dedup skips) as the IDOR/BFLA substrate (session.ts:1171-1195).
- I documented plumbing only: permission rulesets, dispatch wiring, prompt-section names, and data flow. I did not reproduce the offensive payloads/patterns present in the WSTG skill blocks, the injection denylist entries (agent.ts:593-627), or the tester prompt bodies - those are attack technique content out of scope for this architecture spec.

## Key Files
- `packages/hackbrowser/src/agent.ts` — Crawler engine: run() BFS loop, setupRequestInterceptor, createIngestHandler, capture drain, multi-credential path (2412 lines)
- `packages/hackbrowser/src/api.ts` — Library entry point runCrawl: validation, chromium preflight, flat CrawlOptions -> nested AgentConfig, error aggregation
- `packages/hackbrowser/src/capture.ts` — UI-context snapshot (snapshotPageUI), raw HTTP request builder, param<->UI correlation (hiddenParams)
- `packages/hackbrowser/src/ingest.ts` — Transport to /session/ingest: buildIngestPayload, sendIngest, credential register/PATCH sync, page-diff
- `packages/hackbrowser/src/scanner.ts` — DOM/accessibility element collection, page fingerprinting, lazy-content reveal/disclosure expansion
- `packages/cyberstrike/src/tool/hackbrowser-launcher.ts` — Subprocess launcher: activeRuns guard, Bun.spawn worker, IPC reader (backgroundRun), launch/stop, HackbrowserStatus updates
- `packages/cyberstrike/src/server/routes/session.ts` — /ingest route (normalize->dedup->observation->enqueue->proxy-agent) and hackbrowser launch/stop/status routes
- `packages/cyberstrike/src/agent/agent.ts` — Agent registry: proxy-agent, proxy-analyzer, 8 proxy-tester-*, loadVulnAgent + stripDefensiveSections, per-agent permission rulesets, model tiering, step budget
- `packages/cyberstrike/src/tool/task.ts` — Task-tool subagent dispatch: prependRequestContext assembles Current request / Credential / Access Context into subagent prompts
- `packages/cyberstrike/src/agent/prompt/vuln/common-prompt.txt` — Shared tester prompt: lane discipline + mandatory 3-gate baseline/exploit/diff confirmation protocol + coverage-note rules
- `packages/cyberstrike/src/agent/prompt/orchestrator/web-proxy-agent/prompt.txt` — Orchestrator prompt: pure-router role, analyzer-first blocking dispatch, tester-selection heuristics
- `packages/cyberstrike/src/agent/prompt/analyzer/proxy-analyzer/prompt.txt` — Analyzer prompt: extracts objects/roles/functions/ID-values/credential-claims via web_write_* tools (architecture knowledge base)
- `packages/cyberstrike/src/session/ingest-queue.ts` — Per-session promise-chain serialization of ingest LLM tasks with pause/resume
- `packages/cyberstrike/src/session/hackbrowser-status.ts` — Per-session crawl status state machine (starting/crawling/completed/failed) fed by CSEvents
