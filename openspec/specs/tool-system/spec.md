# Tool Framework and Built-in Tool Catalog

## Purpose
The tool system defines a uniform interface (Tool.define / Tool.Info) for all agent-callable tools, manages their registration and initialization via ToolRegistry, provides a lazy-loading mechanism for MCP tools to prevent context overflow (LazyToolRegistry), and enforces zod-based input validation and automatic output truncation. It serves as the single dispatch surface through which LLM agents interact with the host system, external services, and the security testing pipeline.

## Requirements

### Requirement: Tool.define contract
The system SHALL define every tool as a {id, init} pair where init returns {description, parameters(zod), execute} and SHALL wrap execute to validate inputs via zod and truncate outputs via Truncate.output.

#### Scenario: Zod validation on invoke
- **WHEN** A tool is called with arguments that do not satisfy its zod schema
- **THEN** Tool.define wraps execute to call parameters.parse(args) first; on ZodError it throws 'The <id> tool was called with invalid arguments' with the error detail, or delegates to formatValidationError if present (tool.ts:57-67)

#### Scenario: Automatic output truncation
- **WHEN** A tool returns output exceeding MAX_LINES (2000) or MAX_BYTES (50KB) and metadata.truncated is undefined
- **THEN** Tool.define wraps the result through Truncate.output which saves full output to disk (data/tool-output/), returns the head (or tail) within limits, and appends a hint to use Grep/Read/Task to access the rest (tool.ts:72-83, truncation.ts:68-123)

#### Scenario: Self-managed truncation bypass
- **WHEN** A tool sets result.metadata.truncated to any defined value
- **THEN** Tool.define skips the Truncate.output wrapper and returns the result as-is (tool.ts:71-72)

### Requirement: ToolRegistry built-in tool assembly
The system SHALL assemble all built-in tools in a fixed order via the all() function, starting with InvalidTool and ending with custom/plugin tools appended at the tail.

#### Scenario: Full tool list composition
- **WHEN** ToolRegistry.all() is called
- **THEN** Returns an ordered array: InvalidTool, QuestionTool (if client is app/cli/desktop), then 52 built-in tools — 50 unconditional plus 2 conditional (LspTool, BatchTool) — from BashTool through CipipeTool, then custom/plugin tools (registry.ts:133-194)

#### Scenario: QuestionTool gating by client type
- **WHEN** Flag.CYBERSTRIKE_CLIENT is not 'app', 'cli', or 'desktop'
- **THEN** QuestionTool is excluded from the tool list (registry.ts:135)

### Requirement: Model-aware tool filtering
The system SHALL filter tools based on model and provider when ToolRegistry.tools(model, agent) is called, gating websearch/codesearch by provider or flag and swapping edit/write vs apply_patch based on model family.

#### Scenario: Websearch/codesearch gating
- **WHEN** The model providerID is not 'cyberstrike' and CYBERSTRIKE_ENABLE_EXA flag is not set
- **THEN** websearch and codesearch tools are filtered out of the returned tool set (registry.ts:233)

#### Scenario: Apply_patch vs edit/write swap for GPT models
- **WHEN** Model ID contains 'gpt-' but not 'oss' and not 'gpt-4'
- **THEN** apply_patch is included and edit/write are excluded; otherwise edit/write are included and apply_patch is excluded (registry.ts:237-240)

#### Scenario: Plugin tool.definition hook
- **WHEN** Each tool is initialized during tools()
- **THEN** Plugin.trigger('tool.definition', {toolID}, output) is called, allowing plugins to modify the description and parameters of any tool before it is served to the LLM (registry.ts:251)

### Requirement: Custom tool discovery from .cyberstrike directories
The system SHALL discover custom tools by scanning all Config.directories() for files matching {tool,tools}/*.{js,ts}, importing each module, and wrapping exported ToolDefinitions via fromPlugin().

#### Scenario: Custom tool file scanning
- **WHEN** ToolRegistry.state() initializes
- **THEN** A Bun.Glob('{tool,tools}/*.{js,ts}') scans each directory from Config.directories() (which includes global ~/.cyberstrike/, project .cyberstrike/ directories, and CYBERSTRIKE_CONFIG_DIR), imports each match, and wraps each export as a Tool.Info via fromPlugin() (registry.ts:67-83)

#### Scenario: Plugin-provided tools
- **WHEN** Plugins are loaded via Plugin.list()
- **THEN** Each plugin's .tool record is iterated; each entry is wrapped via fromPlugin(id, def) and appended to the custom tools array (registry.ts:85-91)

### Requirement: LazyToolRegistry for MCP tools
The system SHALL store MCP tool metadata (name, summary, keywords, category) in a lazy registry to avoid consuming context tokens, loading full tool definitions on-demand via load/unload with a 30,000-token context budget.

#### Scenario: Init collects metadata only
- **WHEN** LazyToolRegistry.init() runs
- **THEN** Iterates all connected MCP servers, calls client.listTools(), and stores only LazyTool metadata (id, name, summary truncated to 100 chars, extracted keywords, auto-categorized category) -- approximately 50 tokens per tool vs 500 for a full definition (lazy-registry.ts:46-98)

#### Scenario: Context budget enforcement
- **WHEN** load() would bring total loaded tools above TOOL_CONTEXT_BUDGET (30,000 tokens at 500 tokens/tool = 60 tools)
- **THEN** A warning is logged about budget exceedance; tools are still loaded but the agent is informed of token pressure via stats (lazy-registry.ts:199-209)

#### Scenario: ToolsChanged bus subscription
- **WHEN** An MCP server emits a ToolListChanged notification
- **THEN** Bus.subscribe(MCP.ToolsChanged) triggers refresh(serverName) which re-lists that server's tools and adds any new ones to the lazy registry (lazy-registry.ts:144-151)

### Requirement: Tool search and dynamic load/unload meta-tools
The system SHALL provide four meta-tools (tool_search, load_tools, unload_tools, list_loaded_tools) that let agents discover and dynamically activate MCP tools at runtime without pre-loading all definitions.

#### Scenario: Keyword search with scoring
- **WHEN** tool_search is called with a query string
- **THEN** LazyToolRegistry.search() scores all lazy tools by exact name match (+100), name contains (+50), summary contains (+30), keyword overlap (+20 per match), category match (+25), then returns top N results with IDs for load_tools (lazy-registry.ts:153-194, tool-search.ts:14-56)

#### Scenario: Load and unload lifecycle
- **WHEN** load_tools is called with tool IDs from search results
- **THEN** LazyToolRegistry.load() resolves full MCP tool definitions via MCP.tools(), stores them in loadedTools map, and reports them via ToolRegistry.getLoadedMCPTools() for inclusion in future turns (lazy-registry.ts:196-246, registry.ts:207-214)

### Requirement: Vulnerability lane discipline enforcement
The system SHALL enforce lane discipline at the tool layer so proxy-tester-<class> agents can only record findings and VRT checks within their assigned vulnerability class, rejecting off-class operations with hand-off instructions.

#### Scenario: VRT check scope rejection
- **WHEN** A proxy-tester-idor agent calls update_vrt_check with vrt_category='sqli'
- **THEN** vrtScopeViolation() matches agent name pattern proxy-tester-(.+), looks up TESTER_VRT_SCOPE['idor'] (which allows only 'idor' and 'bola'), finds no match for 'sqli', and the tool returns an offLaneMessage instructing hand-off via add_intel (vuln-scope.ts:69-82, vrt-check.ts:42-52)

#### Scenario: Anti-spoof title cross-check for report_vulnerability
- **WHEN** A proxy-tester-injection agent reports a finding with vrt_category='injection' but title contains 'IDOR'
- **THEN** reportScopeViolation() runs classifyFindingTitle() which infers 'idor' from the title, detects cls('injection') != inferred('idor'), and flags the report as an off-lane mislabel. The finding is still recorded but marked with a hand-off note (vuln-scope.ts:108-131)

### Requirement: Permission gating via ctx.ask
The system SHALL require explicit permission approval for high-risk tools by calling ctx.ask() with a permission identifier and pattern before execution proceeds.

#### Scenario: Vulnerability reporting permission
- **WHEN** report_vulnerability execute() runs
- **THEN** ctx.ask({permission:'report_vulnerability', patterns:['*'], always:['*']}) is called before Vulnerability.add(), blocking until the permission system grants approval (vulnerability.ts:39-44)

#### Scenario: Hackbrowser permission with target URL pattern
- **WHEN** hackbrowser execute() runs with target 'https://example.com'
- **THEN** ctx.ask({permission:'hackbrowser', patterns:['https://example.com'], always:['https://example.com']}) is called, allowing the user to grant blanket permission for that host on first use (hackbrowser.ts:64-69)

### Requirement: Output truncation infrastructure
The system SHALL truncate tool output exceeding 2000 lines or 50KB by saving full output to a timestamped file and returning a truncated preview with a hint to use Read/Grep/Task for the full content.

#### Scenario: Head truncation with Task hint
- **WHEN** Tool output exceeds limits and the calling agent has Task tool permission
- **THEN** Truncate.output saves full text to Global.Path.data/tool-output/<ascending-id>, returns the head preview with hint: 'Use the Task tool to have explore agent process this file' (truncation.ts:113-116)

#### Scenario: Periodic cleanup of truncated output files
- **WHEN** The scheduler fires the tool.truncation.cleanup job (hourly)
- **THEN** Files in the tool-output directory older than 7 days (RETENTION_MS) are deleted based on their timestamp-encoded IDs (truncation.ts:43-60)

## Notes
- Built-in tool inventory (52 tools: 50 unconditional + 2 conditional) organized by category: (1) Core IDE: bash, read, edit, write, glob, grep, apply_patch, question, skill, task, todo_write, batch, lsp; (2) Web/Search: webfetch, websearch, codesearch; (3) Memory: memory_search, memory_write, memory_read, memory_context; (4) Meta/Dynamic: tool_search, load_tools, unload_tools, list_loaded_tools; (5) Vulnerability Management: report_vulnerability, triage_vulnerability; (6) Web Proxy Pipeline: web_write_role, web_write_object, web_write_object_value, web_write_function, web_get_session_context, web_get_detail, web_get_request_detail, web_get_vulnerabilities, web_get_vuln_detail, web_update_credential_claims; (7) Browser Automation: hackbrowser; (8) Methodology Engine: add_intel, update_vrt_check, record_coverage_note, get_coverage_notes, scope_check, ensure_tools, methodology_status, generate_report; (9) Attack Scripts: attack_script (16 bundled Python scripts); (10) Post-Exploitation Toolkits: ebpf, winhook, machook, awshook, azurehook, kubehook, cipipe; (11) Invalid: invalid (error sentinel).
- The fromPlugin() adapter (registry.ts:95-117) bridges @cyberstrike-io/plugin ToolDefinition to internal Tool.Info, constructing a zod schema from def.args and wrapping output through Truncate.output.
- Tool gating is multi-layered: Feature flags (CYBERSTRIKE_EXPERIMENTAL_LSP_TOOL, CYBERSTRIKE_ENABLE_EXA, experimental.batch_tool config), client type (CYBERSTRIKE_CLIENT for QuestionTool), model family (GPT model detection for apply_patch vs edit/write swap), provider ID (cyberstrike provider for websearch/codesearch), and lane discipline (vuln-scope.ts for proxy-tester agents).
- The LazyToolRegistry auto-categorizes MCP tools into 9 categories (web, network, file, security, cloud, database, git, shell, general) via keyword matching against tool names and descriptions, with 'general' as the default fallback (lazy-registry.ts:379-400).
- Config.directories() scans upward from the project root for .cyberstrike/ directories, also including ~/.cyberstrike/ and optional CYBERSTRIKE_CONFIG_DIR, providing the search paths for custom tool discovery.
- Post-exploitation toolkit tools (ebpf, winhook, machook, awshook, azurehook, kubehook, cipipe) follow the same pattern as attack_script: they define an AVAILABLE_PROGRAMS catalog and spawn subprocesses from bundled scripts in data/<toolkit>/ directories.

## Key Files
- `packages/cyberstrike/src/tool/tool.ts` — Core Tool.define factory and Tool.Info interface. Wraps every tool with zod validation and automatic Truncate.output.
- `packages/cyberstrike/src/tool/registry.ts` — ToolRegistry: assembles all built-in tools, discovers custom tools from .cyberstrike/{tool,tools}/*.{js,ts} and plugin exports, filters by model/provider/flags, triggers plugin tool.definition hooks.
- `packages/cyberstrike/src/tool/lazy-registry.ts` — LazyToolRegistry: stores MCP tool metadata (~50 tokens each) instead of full definitions (~500), supports keyword search, on-demand load/unload, 30K token context budget, and ToolsChanged bus refresh.
- `packages/cyberstrike/src/tool/tool-search.ts` — Four meta-tools (tool_search, load_tools, unload_tools, list_loaded_tools) that let agents dynamically discover and activate MCP tools at runtime.
- `packages/cyberstrike/src/tool/truncation.ts` — Truncate namespace: MAX_LINES=2000, MAX_BYTES=50KB limits; saves full output to disk; head/tail truncation modes; 7-day retention cleanup via Scheduler.
- `packages/cyberstrike/src/tool/vuln-scope.ts` — Lane discipline for proxy-tester agents: TESTER_VRT_SCOPE taxonomy, vrtScopeViolation/reportScopeViolation/dispatchScopeViolation guards, anti-spoof title cross-check.
- `packages/cyberstrike/src/tool/vulnerability.ts` — report_vulnerability tool: records findings via Vulnerability.add, triggers similarity check, surfaces triage instructions, applies lane discipline via reportScopeViolation.
- `packages/cyberstrike/src/tool/hackbrowser.ts` — hackbrowser tool: permission-gated async browser crawler launcher, delegates to hackbrowser-launcher.ts, streams captures into session.
- `packages/cyberstrike/src/tool/attack-script.ts` — attack_script tool: executes bundled Python scripts (16 categories) from data/scripts/ with timeout enforcement.
- `packages/cyberstrike/src/plugin/index.ts` — Plugin namespace: loads internal plugins (Codex, Copilot, Anthropic, Gitlab auth), npm-installed plugins, and exposes Plugin.trigger() for hook dispatch including tool.definition mutation.
