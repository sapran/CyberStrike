# WSTG-Driven Methodology Engine

## Purpose
A hardcoded, in-process engine that turns a session's accumulated intel entries into a stateful pentest methodology: a 13-phase lifecycle with prerequisite gating, per-intel-type VRT checklists and coverage math, eight vulnerability-chain detectors, evidence-quality/validation gates, and per-agent "liyakat" scoring. Its central output is a Markdown block injected into every methodology-aware system prompt (context.ts) that tells the orchestrator (or a lane-scoped tester) exactly what phase it is in, what is blocking, what to test next, and which sub-agent to delegate to. All of its knowledge — phases, OWASP maps, VRT paths, chain patterns, thresholds, directives — is inline TypeScript with no data-file or config extension point, and is fully decoupled from the on-disk WSTG skill library.

## Requirements

### Requirement: Phase Model and Scope Filtering
The system SHALL model testing as a fixed 13-phase lifecycle (Phase.ALL) with prerequisite chains, filter phases to the detected scope type, and compute each phase's status from tag-matched intel deliverables and prerequisite completion.

#### Scenario: Fixed 13-phase union
- **WHEN** the phase model is enumerated
- **THEN** Phase.Id is a hardcoded 13-member union (scope_analysis, passive_recon, active_recon, technology_profiling, authentication_testing, session_management, authorization_testing, input_validation, business_logic, data_protection, api_security, infrastructure, reporting) and Phase.ALL holds one Definition per id (phase.ts:7-20, 37-178); each Definition carries prerequisites, requiredTags, minDeliverables, relatedVrtCategories, appliesTo scope types, and recommended agents

#### Scenario: Scope-type detection filters applicable phases
- **WHEN** computeState runs for a session
- **THEN** Phase.detectScopeType classifies scope items in order cidr → mobile → wildcard → single by regex (phase.ts:299-304) and Phase.forScope keeps only phases whose appliesTo includes that type (phase.ts:184-186); e.g. passive_recon/active_recon apply only to wildcard|single, so a cidr scope yields just scope_analysis, infrastructure, reporting and a mobile scope yields only scope_analysis + reporting

#### Scenario: Status derived from deliverables + prerequisites
- **WHEN** a phase is evaluated in computeState
- **THEN** status is completed when tag-matched entries >= minDeliverables (>0), else blocked when an in-scope prerequisite phase is incomplete, else in_progress when deliverableCount>0, else not_started (methodology.ts:51-71, 92-148); prerequisites outside the current scope are skipped (methodology.ts:139) and results are upserted into MethodologyPhaseTable

#### Scenario: Tag matching is fuzzy
- **WHEN** validateDeliverables checks an entry against a phase
- **THEN** an entry matches if its tags array contains a requiredTag OR its lowercased title includes the tag or the tag with dashes replaced by spaces (methodology.ts:96-102), so free-text titles can satisfy phase completion without explicit tagging

### Requirement: OWASP and VRT Category Reference Maps
The system SHALL define OWASP Web Top-10, OWASP API Top-10, and per-intel-type VRT category maps as hardcoded constants, wiring only the VRT map into runtime checklist generation while the OWASP maps remain unreferenced reference data.

#### Scenario: VRT map drives auto-generated checklists
- **WHEN** an intel entry of a given type is added
- **THEN** Intel.generateVrtChecklist returns Phase.VRT_CATEGORIES_BY_TYPE[type] (intel.ts:261-263), a hardcoded Record mapping each of ~9 populated intel types to {category, path} WSTG-style paths (phase.ts:245-295); Intel.add inserts one pending VrtCheck row per returned category (intel.ts:135-152)

#### Scenario: OWASP Top-10 maps are declared but dead
- **WHEN** OWASP_WEB_TOP10 / OWASP_API_TOP10 are searched for consumers
- **THEN** both constants (10 entries each, A01-A10 and API1-API10, each with a vrtCategories list, phase.ts:197-241) are defined but referenced nowhere outside phase.ts — they are inert reference data, not wired into coverage, violations, or prompt output

#### Scenario: Some intel types map to empty checklists
- **WHEN** an entry of type vulnerability_hint, business_rule, or sensitive_data is added
- **THEN** VRT_CATEGORIES_BY_TYPE returns an empty array (phase.ts:292-295) and no VRT checks are created, so those types contribute entries but zero coverage checks

### Requirement: VRT Coverage Computation and Confidence Weighting
The system SHALL compute session and per-asset VRT coverage as completed/total checks, plus a confidence-weighted coverage percentage using fixed CONFIDENCE_WEIGHTS, and surface untested items.

#### Scenario: Raw vs weighted coverage
- **WHEN** Intel.computeCoverage runs
- **THEN** coveragePercent = round(non-pending checks / total checks * 100); weightedCoveragePercent multiplies each check by its entry's CONFIDENCE_WEIGHTS (confirmed 1.0, high 0.8, medium 0.5, low 0.2, default 0.2) so low-confidence findings count less toward completion (intel.ts:59-64, 385-405)

#### Scenario: Per-asset coverage ranking
- **WHEN** Intel.computePerAssetCoverage runs
- **THEN** checks are grouped by lowercased/trimmed asset and returned sorted ascending by coveragePercent (intel.ts:525-562), feeding the per_asset_coverage gate and the 'worst asset' force-continue directive

### Requirement: Red-Flag and Violation Gate Detection
The system SHALL generate methodology violations (ordering, progress, per-asset coverage, evidence quality) and coverage red flags (missing evidence, bulk copy-paste, too-fast, over-N/A) with fixed severities and numeric thresholds, persisting methodology violations to the database.

#### Scenario: Four methodology violation gates
- **WHEN** generateViolations runs inside computeState
- **THEN** it defines four gates, but methodology_ordering (blocking, intended to fire when an in_progress phase still has a blockReason) can never emit — a phase reaches in_progress status only when its prerequisites allow it to start, and blockReason is assigned undefined in exactly that case (methodology.ts:55-61, 69), so the line 206 condition is unsatisfiable; the three gates that actually fire are methodology_progress (warning) when input_validation/authorization_testing/business_logic have deliverables but no recon phase is complete, per_asset_coverage (warning) when an asset's entry count < 30% of the average, and evidence_quality (blocking) when an exploited entry's detail is under 50 chars (methodology.ts:195-267); results replace unresolved rows in ValidationViolationTable

#### Scenario: Four coverage red flags
- **WHEN** detectRedFlags runs inside computeCoverage
- **THEN** it emits no_evidence (critical) for tested_not_vulnerable checks lacking evidence, bulk_copy_paste (warning) when >=5 distinct entries share identical reasoning text, too_fast (warning) when >=10 completed checks report <5 total HTTP requests, and all_not_applicable (warning) when >60% of checks are not_applicable (intel.ts:432-511)

### Requirement: Vulnerability Chain Detection
The system SHALL detect eight named multi-finding attack-chain patterns from intel entries and their tested_vulnerable VRT checks, scoring each by confidence, deduplicating, and persisting candidates while preserving prior human/agent status.

#### Scenario: Eight hardcoded patterns
- **WHEN** Chain.detect runs on a session with >=2 entries
- **THEN** it evaluates credential_endpoint, info_disclosure_ssrf, redirect_oauth, idor_data_leak, xss_csrf, ssti_rce, race_condition_business, and custom (chain.ts:13-21, 99-250), gating candidates on same-asset-or-same-registered-domain proximity (chain.ts:60-67) and, for most patterns, a tested_vulnerable VRT check plus a regex match (OAUTH/USER_DATA/STATE_CHANGE/PAYMENT, chain.ts:37-40) against the partner entry's title+detail

#### Scenario: Confidence scoring and ordering
- **WHEN** a chain candidate is produced
- **THEN** each pattern assigns a fixed/graduated confidence (e.g. credential_endpoint 90/85/65 by cred status, chain.ts:108; info_disclosure_ssrf 90 if SSRF confirmed else 55, chain.ts:127), candidates are deduped by pattern+sorted-entry-id key and sorted descending by confidence (chain.ts:72-97, 252-253)

#### Scenario: Auto-detect on intel add; status preserved on save
- **WHEN** add_intel completes for a non-duplicate entry
- **THEN** Chain.detectAndPersist re-runs detection and rewrites ChainCandidateTable (tool/intel.ts:72), but save() first loads existing rows and re-applies their id and status (detected/testing/confirmed/disproven) by key so a human-set status is not clobbered (chain.ts:258-292)

### Requirement: Evidence-Quality and Validation Gates
The system SHALL enforce numeric evidence-quality thresholds on vulnerable checks, triager checks on findings (severity-impact, reproducibility, scope, duplicate), and a per-asset minimum-coverage gate, aggregating them into a pass/fail GateResult.

#### Scenario: Evidence-quality thresholds
- **WHEN** Validation.validateEvidenceQuality inspects a tested_vulnerable check
- **THEN** it raises blocking violations when requestSent lacks an HTTP verb/curl token, responseSummary < 50 chars, reasoning < 100 chars, requestCount < 1, or an exploited entry's responseSummary lacks real-data indicators (EXPLOITED_DATA_REGEX) (validation.ts:59-156)

#### Scenario: Triager checks
- **WHEN** Validation.runTriagerChecks evaluates an exploited or vulnerable-tested entry
- **THEN** it flags critical severity without P1 indicators (blocking), high severity showing only info-disclosure (warning), exploited entries with <3 total requests as non-reproducible (blocking), out-of-scope assets (blocking), and same-asset+category duplicates (warning) (validation.ts:160-274)

#### Scenario: Combined gate with 60% coverage floor
- **WHEN** Validation.runAllGates runs
- **THEN** it adds a blocking per_asset_coverage violation for any asset with checks below MIN_COVERAGE=60% and returns overallPassed=false if any blocking violation exists, with a formatted PASS/FAIL summary (validation.ts:278-334); this gate backs generate_report and the methodology-status tool

### Requirement: Per-Turn System-Prompt Context Injection
The system SHALL generate a Markdown methodology block per turn only when the session has intel entries, assembling phase progress, blocking violations, current-phase directive, coverage, chains, untested items, and orchestrator-only delegation/work-queue sections, lane-scoped for proxy-testers, and inject it into the system prompt at session/prompt.ts.

#### Scenario: Activation gated on intel presence
- **WHEN** MethodologyContext.generate(sessionID, agentClass) is called
- **THEN** it returns null if no IntelEntryTable row exists for the session (context.ts:29-40); otherwise it builds sections 1-11 (header, phase progress, blocking violations, current-phase directive from getPhaseDirectives, coverage+red flags, chain opportunities, untested items, protocol reminders, delegation briefing, work queue, intel brief, evidence feedback) (context.ts:42-154)

#### Scenario: Wired into the system prompt build
- **WHEN** session/prompt.ts assembles the system prompt for a turn
- **THEN** it calls MethodologyContext.generate(Session.root(sessionID), testerClass(lastUser.agent)) and pushes the result onto the system array when non-null (session/prompt.ts:720-721)

#### Scenario: Tester lane scoping
- **WHEN** the running agent is a proxy-tester-<class>
- **THEN** testerClass extracts the class (vuln-scope.ts:138-141); generate then filters untested items to that lane via categoryInLane and skips the orchestrator-only delegation briefing and work queue, so a tester sees only its own queue (context.ts:113-144)

#### Scenario: Per-phase directives are hardcoded strings
- **WHEN** a current phase directive is needed
- **THEN** Methodology.getPhaseDirectives returns a value from a hardcoded Record keyed by all 13 Phase.Ids (methodology.ts:355-385), e.g. the input_validation directive instructs testing XSS/SQLi/SSTI/command-injection/SSRF/XXE and recording via update_vrt_check

### Requirement: Hardcoded, Skill-Library-Decoupled Architecture
The system SHALL encode all methodology knowledge as inline TypeScript constants with no data-file, config, or plugin extension point, and SHALL be fully decoupled from the on-disk WSTG skill library, deriving its phases and VRT categories from free-text strings rather than skill IDs.

#### Scenario: No data-file / config loading
- **WHEN** the methodology/ modules are inspected for external inputs
- **THEN** none of methodology.ts, intel.ts, phase.ts, chain.ts, validation.ts, context.ts, performance.ts read JSON/YAML, import a config, or expose a registration hook; every phase, OWASP entry, VRT path, chain pattern, threshold, and directive is a literal const/array/Record, so extending the engine requires editing and recompiling the .ts source

#### Scenario: Decoupled from the skill library
- **WHEN** the methodology engine is checked for skill references
- **THEN** no module under methodology/ imports or references the skill system; requiredTags and VRT categories are free-text (e.g. 'input-validation', 'SQLi'), never the wstg-* skill IDs used by the skill library (docs/skill-signing.md), so phase completion and coverage are computed independently of which skills exist or are loaded

#### Scenario: 'WSTG-driven' is loose framing, not literal WSTG
- **WHEN** the engine is compared to OWASP WSTG
- **THEN** the 13 custom phases and VRT paths are inspired by WSTG/OWASP naming but contain zero WSTG identifiers in the engine itself; the '125 WSTG' figure in packages/cyberstrike/README.md:103 counts skill-library entries, not engine phases

## Notes
- HARDCODED / NO EXTENSION POINT: every phase, OWASP entry, VRT path, chain pattern, threshold, directive, and agent persona is an inline TypeScript literal. There is no data file, JSON/YAML config, env override, or plugin/registration hook anywhere in methodology/. Adding a phase, VRT category, chain pattern, or evidence threshold means editing the .ts source and recompiling.
- DECOUPLED FROM SKILL LIBRARY: no methodology/ module imports or references the skill system. Phase requiredTags and VRT categories are free-text strings (e.g. 'input-validation', 'SQLi', 'IDOR'), never the wstg-* skill IDs used on disk (docs/skill-signing.md). Coverage and phase completion are computed from intel tags/titles independently of which skills exist or are loaded.
- OWASP maps are DEAD DATA: OWASP_WEB_TOP10 and OWASP_API_TOP10 (phase.ts:197-241, 10 entries each) are defined but referenced nowhere outside phase.ts — they do not feed coverage, violations, or prompt output. Only VRT_CATEGORIES_BY_TYPE is wired (into Intel.generateVrtChecklist).
- 'WSTG-driven' is marketing framing, not literal WSTG. The engine's 13 phases are a custom lifecycle loosely inspired by WSTG/PTES; the engine contains ZERO WSTG identifiers. The README's '125 WSTG' (packages/cyberstrike/README.md:103) counts skill-library files, not engine phases — do not conflate the two.
- Phase count is exactly 13 (Phase.Id union and getPhaseDirectives both have 13 keys); chain patterns are exactly 8 (7 signature-based + 'custom'), matching the file header comments.
- Scope filtering silently drops phases: cidr scope yields only scope_analysis/infrastructure/reporting; mobile yields only scope_analysis/reporting (phase.ts appliesTo). This means most testing phases are invisible for non-web scopes — an important behavioral gotcha, not a bug in the reading.
- Two different evidence thresholds coexist: methodology.ts evidence_quality checks exploited-entry.detail >= 50 chars; validation.ts checks the VRT-check evidence object (responseSummary >= 50, reasoning >= 100, requestCount >= 1, triager reproducibility >= 3 requests). They are separate gates with separate call sites.
- Tag matching is fuzzy (title substring, dash/space normalization), so phases can complete without explicit tagging and coverage can be satisfied loosely — deliberate leniency, worth flagging for anyone reasoning about gate strictness.
- This is opencode-lineage code living under packages/cyberstrike/src/methodology — a CyberStrike-specific subsystem layered on the sst/opencode fork; the session/prompt.ts injection point is the only coupling to the base agent loop.

## Key Files
- `packages/cyberstrike/src/methodology/phase.ts` — Hardcoded 13-phase model (Phase.Id union, Phase.ALL Definitions with prereqs/tags/scope/agents), scope-type detection, OWASP Web/API Top-10 maps (declared but unused), and per-intel-type VRT_CATEGORIES_BY_TYPE map
- `packages/cyberstrike/src/methodology/methodology.ts` — Phase-state computation, prerequisite gating, four violation gates + persistence, prompt formatting, and hardcoded per-phase directives (getPhaseDirectives)
- `packages/cyberstrike/src/methodology/intel.ts` — Intel CRUD + dedup, auto-generated VRT checklists, raw+confidence-weighted coverage computation, per-asset coverage, and four coverage red-flag detectors
- `packages/cyberstrike/src/methodology/chain.ts` — Eight vulnerability-chain detectors with proximity/regex gating, confidence scoring, dedup, and status-preserving persistence
- `packages/cyberstrike/src/methodology/validation.ts` — Evidence-quality thresholds, triager checks (severity/reproducibility/scope/duplicate), and combined runAllGates with 60% per-asset coverage floor
- `packages/cyberstrike/src/methodology/context.ts` — MethodologyContext.generate — per-turn system-prompt block assembly, tester lane scoping, force-continue directives, delegation briefing, and work queue
- `packages/cyberstrike/src/methodology/performance.ts` — Static agent BONES (codenames/archetypes/strengths), liyakat scoring formula, morale, and phase/mission agent selection consumed by context.ts delegation sections
- `packages/cyberstrike/src/session/prompt.ts` — Wiring point (line 720-721): injects MethodologyContext.generate output into the system prompt, scoped by testerClass(agent)
- `packages/cyberstrike/src/tool/vuln-scope.ts` — testerClass() and categoryInLane() helpers that lane-scope the injected context for proxy-tester agents
- `packages/cyberstrike/src/tool/intel.ts` — add_intel tool that triggers Chain.detectAndPersist after each non-duplicate entry
