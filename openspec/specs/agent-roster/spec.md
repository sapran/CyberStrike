# Agent Roster: Native Agents and Agent Definition/Merge

## Purpose
Defines the fixed set of built-in ("native") agents CyberStrike seeds at startup and the schema/merge machinery that lets user config and markdown files add or override agents. Each agent carries a system prompt, a permission ruleset, an optional model/skills/steps seed, and mode/visibility flags. Native agents are code-defined and immutable-by-name; config/markdown agents are merged on top, unknown names default to mode "all", and disabled agents are deleted. This is the single source of truth for which agent identities exist and what each is allowed to do.

## Requirements

### Requirement: Native Agent Roster
The system SHALL seed a fixed set of 21 code-defined native agents (native:true) at startup: primary cyberstrike; specialists web-application, mobile-application, cloud-security, internal-network; utilities general, explore, compaction, title, summary, normalize-request; proxy pipeline proxy-agent, proxy-analyzer, and 8 proxy-tester-* agents.

#### Scenario: Primary agent
- **WHEN** the agent state map is built
- **THEN** cyberstrike is registered with mode:"primary", native:true, and prompt = PROMPT_METHODOLOGY_COMMON + '---' + PROMPT_CYBERSTRIKE (agent.ts:178-192), and is the only non-subagent, non-hidden native agent

#### Scenario: Specialist subagents
- **WHEN** listing native subagents
- **THEN** web-application (color red, agent.ts:299-330), mobile-application (magenta, 331-360), cloud-security (cyan, 361-403), internal-network (yellow, 404-450) are each mode:"subagent", native:true, prompt prefixed with common+forced-continuation methodology

#### Scenario: Eight proxy testers
- **WHEN** the async IIFE at agent.ts:515-719 resolves
- **THEN** proxy-tester-idor, -authz, -mass-assignment, -injection, -authn, -business-logic, -ssrf, -file-attacks are registered — 8 testers, each mode:"subagent", native:true, hidden:true, prependRequestContext:true

#### Scenario: Count grounding
- **WHEN** counting seeded native agents before config merge
- **THEN** exactly 21 entries exist: 1 primary + 4 specialists + 6 utilities (general, explore, compaction, title, summary, normalize-request) + proxy-agent + proxy-analyzer + 8 testers

### Requirement: Agent Info Schema
The system SHALL define each resolved agent by the Agent.Info schema whose fields include name, description, mode(subagent|primary|all), native, hidden, topP, temperature, color, permission(Ruleset), model{modelID,providerID}, useSmallModel, variant, prompt, skills[], options, steps, and prependRequestContext.

#### Scenario: Schema fields
- **WHEN** validating an agent record
- **THEN** Agent.Info (agent.ts:119-151) requires name, mode, permission, options and makes the rest optional; only permission and options are non-optional besides name/mode

#### Scenario: Model is optional per-agent
- **WHEN** a native agent omits model
- **THEN** no model is pinned and the agent inherits the session/parent model at run time (agent.ts:130-135; consumed at prompt.ts:1147 `agent.model ?? lastModel(...)`); only useSmallModel or config can change this

### Requirement: Config And Markdown Agent Merge
The system SHALL merge user-defined agents from config (cfg.agent) and from {agent,agents}/**/*.md markdown files on top of the native roster, copying model, variant, prompt, description, temperature, top_p, mode, color, hidden, name, steps, options, and permission onto the matching or newly-created entry.

#### Scenario: Markdown discovery
- **WHEN** loadAgent scans a config dir
- **THEN** each {agent,agents}/**/*.md file is parsed as frontmatter+body, the body becomes prompt, the path-derived slug becomes name, validated against Config.Agent, and folded into cfg.agent (config.ts:448-486)

#### Scenario: Override of native agent
- **WHEN** cfg.agent has a key matching an existing native agent
- **THEN** listed fields are overlaid (value ?? existing) and permission is merged as PermissionNext.merge(item.permission, fromConfig(value.permission)) (agent.ts:736-748) — native:true is preserved because the loop never overwrites native

#### Scenario: Fields not copied from config
- **WHEN** a markdown/config agent declares skills, useSmallModel, or prependRequestContext
- **THEN** the merge loop (agent.ts:736-748) copies none of these onto the entry; skills (not in config knownKeys) is diverted into options (config.ts:817-821), so user agents cannot set the skills recommendation, small-tier, or request-context behaviors that native agents use

### Requirement: Unknown Name Defaults And Disable
The system SHALL create a new agent with mode:"all", native:false, and merged default+user permissions when a config key matches no native agent, and SHALL delete any agent whose config sets disable:true.

#### Scenario: New config agent
- **WHEN** cfg.agent has a key not present in the native roster and not disabled
- **THEN** a fresh entry is created with mode:"all", permission = merge(defaults, user), options {}, native:false, before per-field overrides apply (agent.ts:728-735)

#### Scenario: Disable deletes
- **WHEN** cfg.agent[key].disable is true
- **THEN** result[key] is deleted and the loop continues — this removes native agents too (agent.ts:723-726)

### Requirement: User Markdown Agent Config Fields
The system SHALL accept a documented Config.Agent field set for user markdown/config agents — model, variant, temperature, top_p, prompt, description, mode(subagent|primary|all), hidden, color, steps, options, permission — with legacy tools→permission and maxSteps→steps conversion and unknown keys folded into options.

#### Scenario: Documented fields
- **WHEN** parsing frontmatter
- **THEN** Config.Agent (config.ts:760-793) defines mode enum, model (ModelId), temperature/top_p numbers, steps (positive int), color (hex #RRGGBB or theme enum), hidden (bool, subagent-only), plus prompt/description/variant

#### Scenario: Legacy conversions
- **WHEN** an agent uses deprecated tools:{} or maxSteps
- **THEN** tools booleans become allow/deny permission entries (write/edit/patch/multiedit→edit) and maxSteps is aliased to steps (config.ts:823-843)

#### Scenario: Catchall to options
- **WHEN** frontmatter has keys outside the known set
- **THEN** unknown keys are moved into options via the transform (config.ts:796-821)

### Requirement: Layered Permission Composition
The system SHALL compose each native agent's permission as PermissionNext.merge(defaults, per-agent overrides, user config) so that user overrides normally win but injection-tester hard-denies are merged after the user layer, and SHALL ensure Truncate.GLOB stays allowed unless explicitly denied.

#### Scenario: Default-deny subagents
- **WHEN** building a locked-down subagent like explore or a proxy tester
- **THEN** permission starts from defaults, then a per-agent ruleset sets '*':'deny' plus a small allowlist, then user config is merged last (e.g. agent.ts:213-230, 563-585)

#### Scenario: Injection tester defense-in-depth
- **WHEN** constructing proxy-tester-injection
- **THEN** destructive SQL/RCE bash patterns are denied AFTER the user layer (injectionAgentPermission = merge(vulnAgentPermission,...)), so user config cannot loosen them (agent.ts:593-627)

#### Scenario: Truncation glob guarantee
- **WHEN** finalizing every agent
- **THEN** unless an agent explicitly denies Truncate.GLOB, external_directory:{[GLOB]:'allow'} is merged in so truncated-file reads always work (agent.ts:751-765)

### Requirement: Model Tier, Skills Seed, Request Context, Step Budget
The system SHALL apply per-agent execution seeds: useSmallModel downgrades to the provider small tier at run time, the skills[] list is surfaced as a recommendation in the system prompt, prependRequestContext injects current-request metadata into the task prompt, and proxy-tester-* agents default to a soft 50-step budget.

#### Scenario: Small-tier downgrade
- **WHEN** the running agent has useSmallModel (proxy-analyzer, agent.ts:493)
- **THEN** the run model is swapped to Provider.getSmallModel(providerID) for that turn without persisting the downgrade onto the user message (prompt.ts:372-373, 1142-1147)

#### Scenario: Skills recommendation
- **WHEN** an agent defines skills[] (e.g. web-application, cloud-security)
- **THEN** those names are appended to the skills-availability system block as 'Recommended skills … load these first' (prompt.ts:788-791) — it is guidance, not a permission grant

#### Scenario: Request context prepended
- **WHEN** a task dispatches to an agent with prependRequestContext (all proxy testers + analyzer)
- **THEN** task.ts:167-189 prepends the current request's id/method/path/origin/status and any wide-coverage note to the sub-agent prompt

#### Scenario: Tester step cap
- **WHEN** a proxy-tester-* agent has no explicit steps
- **THEN** steps is set to 50 as a soft cap that forces a text wrap-up at the limit (agent.ts:777-779; enforced via agent.steps ?? Infinity at prompt.ts:613)

### Requirement: Default And Listing Selection
The system SHALL resolve the default primary agent from cfg.default_agent (rejecting subagent/hidden targets) or fall back to the first non-subagent non-hidden agent, and SHALL list agents sorted so the default (or cyberstrike) sorts first.

#### Scenario: Configured default validation
- **WHEN** cfg.default_agent names a subagent or hidden agent
- **THEN** defaultAgent() throws ("is a subagent" / "is hidden"); a missing name throws "not found" (agent.ts:797-807)

#### Scenario: Fallback default
- **WHEN** no default_agent is configured
- **THEN** the first agent with mode !== 'subagent' and hidden !== true is chosen — cyberstrike in the default roster (agent.ts:809-811)

#### Scenario: List ordering
- **WHEN** Agent.list() is called
- **THEN** agents are sorted descending on (name === default_agent) else (name === 'cyberstrike') so the active primary heads the list (agent.ts:788-795)

### Requirement: Agent Generation From Description
The system SHALL generate a new agent config (identifier, whenToUse, systemPrompt) from a natural-language description via the model, excluding all existing agent names from the allowed identifiers.

#### Scenario: Generate excludes existing names
- **WHEN** Agent.generate({description}) runs
- **THEN** the prompt lists every existing agent name as forbidden identifiers and returns a JSON object {identifier, whenToUse, systemPrompt} via generateObject/streamObject (agent.ts:814-869)

## Notes
- Agent count is exactly 21 native agents (1 primary + 4 specialists + 6 utilities + proxy-agent + proxy-analyzer + 8 proxy-tester-*), all native:true. 13 are hidden:true (compaction, title, normalize-request, summary, proxy-analyzer, and the 8 testers).
- None of the native agents pin a `model` field, so they inherit the session/parent model. Only proxy-analyzer downgrades via useSmallModel:true; proxy-agent deliberately keeps the full model (comment agent.ts:456-459 cites measured mis-dispatch when downgraded). A `model` is only ever set on config/markdown agents (Provider.parseModel at agent.ts:736).
- Gotcha: the config merge loop (agent.ts:736-748) copies model/variant/prompt/description/temperature/top_p/mode/color/hidden/name/steps/options/permission but NOT skills, useSmallModel, or prependRequestContext, so these are effectively code-only for native agents. prependRequestContext is in the config knownKeys set (config.ts:814) so it is not shunted to options, but it is still never applied by the merge loop; skills/useSmallModel are not config keys at all, so skills frontmatter lands in options.
- Permission ordering is load-bearing: user config normally merges LAST and wins, but injection-tester destructive-command denies merge AFTER user (agent.ts:593-627) specifically so user config cannot re-enable them. Truncate.GLOB is force-allowed post-merge unless explicitly denied.
- Vuln testers get WSTG skill content STATICALLY embedded into their prompt at build time (loadVulnAgent + stripDefensiveSections, agent.ts:70-116) because their permission denies the `skill` tool; the skills field on specialists is only a load-first recommendation surfaced in the prompt, not an access grant.
- proxy-tester steps default is 50 (soft cap forcing text wrap-up), raised from an earlier 8 after measuring ~12-turn natural runs; enforced as agent.steps ?? Infinity at prompt.ts:613 (agent.ts:767-779).
- Descriptions for proxy-agent/proxy-analyzer/proxy-tester-* are the first line of their sibling description.txt, trimmed (DESC_WEB_PROXY_AGENT.split('\n')[0].trim(), agent.ts:453,489; loadVulnAgent shortDesc at agent.ts:88).
- loadMode ({mode,modes}/*.md) is a legacy sibling of loadAgent that forces mode:"primary" (config.ts:488-523); the config layer also migrates a deprecated top-level mode field into agent (config.ts:258-260).
- Provenance: this is CyberStrike-specific. The native roster, methodology-engine prompt prefixes, proxy pipeline (agent/analyzer/8 testers), useSmallModel, and prependRequestContext are CyberStrike additions on top of the upstream opencode agent/config/permission scaffolding (Agent.Info, loadAgent, generate).

## Key Files
- `packages/cyberstrike/src/agent/agent.ts` — Agent.Info schema (119-151), native roster seed (177-720), config merge loop with unknown-name mode:all and disable-deletes (722-749), Truncate.GLOB guarantee (751-765), proxy-tester steps=50 default (777-779), get/list/defaultAgent/generate (784-869)
- `packages/cyberstrike/src/config/config.ts` — Config.Agent schema and transform: documented user fields, tools-to-permission and maxSteps-to-steps legacy conversion, catchall-to-options (760-848); loadAgent markdown discovery (448-486); loadMode (488-523)
- `packages/cyberstrike/src/session/prompt.ts` — Consumes agent.model/useSmallModel small-tier swap (372-373, 1142-1147), agent.steps budget (613), agent.skills recommendation block (788-791)
- `packages/cyberstrike/src/tool/task.ts` — prependRequestContext injection of current-request metadata into sub-agent prompt (167-189)
- `packages/cyberstrike/src/agent/prompt/` — System prompt .txt files for each native agent; vuln/*/ and orchestrator|analyzer/*/ hold prompt.txt + description.txt (first line = agent description); methodology/*.txt and vuln/common-prompt.txt are prepended prefixes
