# Agent Runtime: Core Turn Loop

## Purpose
The agent runtime turns a user prompt into one or more assistant "turns" by running an iterative step loop that resolves the model and tools, assembles a layered system prompt, streams an LLM completion via the ai-sdk, and persists every reasoning/text/tool part while emitting Bus events for SSE consumers. It also governs the cross-cutting concerns of a running turn: doom-loop detection, context-overflow compaction, cost/usage accounting, retry/backoff, structured-output capture, and dispatch of queued subtasks. Structurally it is opencode's session engine (Session/SessionPrompt/SessionProcessor/LLM namespaces), extended in-place with CyberStrike-specific security-testing behavior (small-model tier downgrade, methodology-engine context injection, proxy-tester lane scoping, agent-performance tracking, skill/MCP awareness).

## Requirements

### Requirement: Prompt-to-Turn Loop Lifecycle
The system SHALL convert a prompt into a persisted user message and then drive an iterative step loop that repeats until the latest assistant message finishes with a reason other than tool-calls/unknown, honoring per-session cancellation.

#### Scenario: Prompt creates message then loops
- **WHEN** SessionPrompt.prompt runs with noReply unset
- **THEN** it creates the user message, touches the session, and returns loop({sessionID}); if noReply===true it returns the message without looping (prompt.ts:165-192)

#### Scenario: Loop exit on natural finish
- **WHEN** the last assistant message has a finish reason not in [tool-calls, unknown] and its id is newer than the last user message
- **THEN** the loop logs "exiting loop" and breaks, ending the turn (prompt.ts:337-344); otherwise it increments step and processes again (prompt.ts:346-897)

#### Scenario: Concurrent prompt on busy session queues
- **WHEN** loop is entered for a sessionID that already has an AbortController in state
- **THEN** start() returns undefined and the caller receives a Promise pushed onto the session's callbacks queue, resolved when the running loop yields its next assistant message (prompt.ts:245-302, 899-906)

#### Scenario: Cancel aborts and rejects joiners
- **WHEN** cancel(sessionID) is called mid-turn
- **THEN** it aborts the controller, rejects every queued callback with "session prompt cancelled", deletes session state, and sets SessionStatus idle so IngestQueue joiners don't hang (prompt.ts:263-287)

### Requirement: Layered System-Prompt Assembly
The system SHALL assemble the system prompt from environment info, project instruction files, a per-model/agent base prompt, and conditional context blocks (structured-output, methodology, MCP availability, disabled MCP servers, skills).

#### Scenario: Base environment + instructions
- **WHEN** building system in the loop
- **THEN** system = SystemPrompt.environment(model) (model id, cwd, git/platform/date) concatenated with InstructionPrompt.system() (AGENTS.md/CLAUDE.md/CONTEXT.md found up-tree, global files, config.instructions incl. fetched URLs) (prompt.ts:711; system.ts:29-53; instruction.ts:71-145)

#### Scenario: Provider/agent base prompt chosen by model
- **WHEN** LLM.stream composes the first system block
- **THEN** it uses agent.prompt if set, else SystemPrompt.provider(model) which selects anthropic/beast/gemini/codex/trinity/qwen text by model.api.id substring, then appends input.system and user.system (llm.ts:67-80; system.ts:19-27)

#### Scenario: Conditional context blocks appended
- **WHEN** the session has intel/MCP/skills state
- **THEN** MethodologyContext (if intel rows exist, scoped by testerClass for proxy-testers), MCP tool-availability + disabled-server notes, and skill-awareness blocks are pushed onto system; json_schema format adds STRUCTURED_OUTPUT_SYSTEM_PROMPT (prompt.ts:712-797)

### Requirement: Tool Resolution
The system SHALL resolve the active tool set from the agent-filtered ToolRegistry plus MCP tools, wrapping each with plugin before/after hooks, permission gating, and output truncation, and injecting a StructuredOutput tool when json_schema format is requested.

#### Scenario: Registry tools wrapped with hooks
- **WHEN** resolveTools iterates ToolRegistry.tools({modelID,providerID}, agent)
- **THEN** each tool's schema is passed through ProviderTransform.schema and its execute is wrapped to fire tool.execute.before/after plugin triggers around item.execute (prompt.ts:965-1001)

#### Scenario: MCP tools loaded lazily or eagerly
- **WHEN** LazyToolRegistry.stats().available > 0
- **THEN** only loaded MCP tools are included (getLoadedMCPTools), else MCP.tools() eager set; each MCP tool gets a mandatory ctx.ask permission check plus truncation and image/resource attachment conversion (prompt.ts:1003-1103)

#### Scenario: Agent/model gating of tools
- **WHEN** ToolRegistry.tools filters and LLM.resolveTools runs
- **THEN** codesearch/websearch require cyberstrike provider or CYBERSTRIKE_ENABLE_EXA; apply_patch vs edit/write toggled by gpt- model id; and tools disabled by agent permission or user.tools[x]===false are deleted before streaming (registry.ts:230-243; llm.ts:262-270)

#### Scenario: StructuredOutput injected for json_schema
- **WHEN** lastUser.format.type === "json_schema"
- **THEN** a StructuredOutput tool built from the schema (minus $schema) is added and toolChoice is forced to "required"; its execute captures args into structuredOutput (prompt.ts:671-678, 844, 1109-1136)

### Requirement: LLM Streaming Configuration
The system SHALL invoke ai-sdk streamText with the assembled system messages, provider/agent/variant-merged options, plugin-adjusted sampling params, tool-call repair, and a language-model middleware that rewrites the prompt via ProviderTransform.

#### Scenario: Options merge + plugin params
- **WHEN** LLM.stream builds request options
- **THEN** base ProviderTransform.options (or smallOptions) is deep-merged with model.options, agent.options, and variant; temperature/topP/topK go through the chat.params plugin trigger; headers through chat.headers (llm.ts:99-149)

#### Scenario: Tool-call repair
- **WHEN** a tool call fails validation
- **THEN** experimental_repairToolCall lowercases a mis-cased tool name if a match exists, otherwise rewrites it to the "invalid" tool carrying the error, and activeTools excludes "invalid" (llm.ts:182-207)

#### Scenario: CyberStrike routing headers
- **WHEN** model.providerID starts with "cyberstrike"
- **THEN** x-cyberstrike-project/session/request/client headers are attached; non-anthropic providers get a cyberstrike User-Agent instead (llm.ts:213-224)

#### Scenario: LiteLLM proxy no-op tool
- **WHEN** the provider is a LiteLLM proxy, no tools are active, and history contains tool calls
- **THEN** a never-called _noop tool is added to satisfy proxy validation (llm.ts:162-174)

### Requirement: Stream Consumption and Persistence
The system SHALL consume the ai-sdk fullStream and persist reasoning, text, tool, step-start/finish, and patch parts, streaming deltas and publishing Bus events, returning continue/stop/compact to the loop.

#### Scenario: Reasoning/text streamed as deltas
- **WHEN** reasoning-delta or text-delta events arrive
- **THEN** the part's accumulated text grows and Session.updatePartDelta publishes MessageV2.Event.PartDelta while full parts publish PartUpdated (processor.ts:82-95,317-329; index.ts:703-737)

#### Scenario: Tool lifecycle persisted
- **WHEN** tool-input-start/tool-call/tool-result/tool-error events arrive
- **THEN** a tool part transitions pending→running→completed/error with input, output, metadata, attachments, and a Snapshot patch recorded at step boundaries (processor.ts:112-233,237-300)

#### Scenario: Blocked / error result codes
- **WHEN** a tool errors with RejectedError and continue_loop_on_deny is not true
- **THEN** process sets blocked and returns "stop"; an assistantMessage.error returns "stop"; overflow returns "compact"; otherwise "continue" (processor.ts:49,224-231,297-299,429-432)

#### Scenario: Abort finalizes running tools
- **WHEN** the abort signal fires or the stream ends with unfinished tool parts
- **THEN** remaining non-completed/non-error tool parts are marked error "Tool execution aborted" and the message completion time is set (processor.ts:57,410-428)

### Requirement: Doom-Loop Detection
The system SHALL detect repeated identical tool calls and require an explicit doom_loop permission when a third identical tool call is attempted.

#### Scenario: Three identical calls trigger permission
- **WHEN** the last DOOM_LOOP_THRESHOLD (3) parts are the same tool with byte-identical JSON input and non-pending status
- **THEN** PermissionNext.ask is issued with permission "doom_loop" scoped to that tool name, using the agent's ruleset (processor.ts:21,152-177)

### Requirement: Context-Overflow Compaction
The system SHALL compact conversation history when token usage approaches the model's usable context, both reactively after a step and proactively before an API call, using a dedicated compaction agent that emits a structured summary.

#### Scenario: Reactive overflow after step
- **WHEN** a finished assistant message's tokens exceed usable context (context/input limit minus reserved buffer, default min(20000, maxOutput))
- **THEN** SessionCompaction.isOverflow returns true and the loop creates an auto compaction turn (compaction.ts:30-48; prompt.ts:597-609; processor.ts:297-299)

#### Scenario: Pre-flight overflow before call
- **WHEN** the estimated tokens of model messages + system reach >=95% of the input limit
- **THEN** the loop triggers compaction and continues instead of making the API call (prompt.ts:800-823)

#### Scenario: Compaction produces summary + resumes
- **WHEN** a compaction part is pending
- **THEN** SessionCompaction.process runs the "compaction" agent over a budget-limited (80% of limit) message window to emit the fixed Objective/Details/Work-State/Next-Move/Files template, then optionally injects a synthetic "Continue..." user message (compaction.ts:103-264)

### Requirement: Cost and Usage Accounting
The system SHALL compute per-step token usage and monetary cost from provider usage plus provider metadata, adjusting for cached tokens and avoiding double-billing reasoning tokens.

#### Scenario: Cache-aware token normalization
- **WHEN** getUsage processes a finish-step usage object
- **THEN** cache-read/write tokens are read from cachedInputTokens and anthropic/bedrock/venice metadata; input tokens are reduced by cache tokens unless the provider (anthropic/bedrock) already excludes them (index.ts:739-795)

#### Scenario: Cost via Decimal, reasoning not re-billed
- **WHEN** cost is computed
- **THEN** input/output/cache rates are applied with Decimal per-million and the >200K over-cap rate is used when input+cacheRead>200000; reasoningTokens are recorded but deliberately NOT added to cost (index.ts:797-818)

#### Scenario: Usage accrues on message
- **WHEN** finish-step fires
- **THEN** assistantMessage.cost += usage.cost and .tokens = usage.tokens, and a step-finish part records reason/tokens/cost (processor.ts:248-278)

### Requirement: Subtask Dispatch and Agent Performance
The system SHALL execute pending subtask parts inline in the loop by invoking the task tool with the subagent's model/agent, recording the mission outcome for agent-performance tracking gated on the subagent's actual result.

#### Scenario: Pending subtask runs task tool
- **WHEN** the newest pending task part is a subtask
- **THEN** the loop creates an assistant message + running tool part and calls TaskTool.execute with prompt/description/subagent_type/command, wrapped in tool.execute.before/after plugin triggers (prompt.ts:380-485)

#### Scenario: Mission recorded only on clean outcome
- **WHEN** the subtask returns with metadata.outcome
- **THEN** AgentPerformance.recordMission counts success/findings/coverage only when outcome==="clean"; aborted/errored/step-capped runs record success:false and don't inflate findings (prompt.ts:510-553)

#### Scenario: Command subtask injects synthetic continuation
- **WHEN** the subtask carried a command
- **THEN** a synthetic user message "Summarize the task tool output above and continue" is appended to keep reasoning-model thinking signatures valid (prompt.ts:555-578)

## Notes
- Provenance: this repo is a fork of sst/opencode; the Session/SessionPrompt/SessionProcessor/LLM/SessionCompaction namespaces, the while(true) step loop, fullStream consumption, getUsage, retry/backoff, doom-loop, and the task-tool subtask-dispatch pattern are opencode-inherited engine structure. Package renaming (opencode->cyberstrike, x-cyberstrike-* headers, cyberstrike.io URLs) is applied throughout.
- CyberStrike-specific behavior woven into the inherited loop (identified by domain concepts, not by diffing against upstream): useSmallModel run-time tier downgrade applied once at model resolution and deliberately NOT persisted on the user message (prompt.ts:367-375, 1141-1147); MethodologyContext injection + proxy-tester lane scoping via testerClass (prompt.ts:720-721; vuln-scope.ts:138-141); AgentPerformance.recordMission with clean-outcome gating (prompt.ts:510-553); skill-awareness + MCP lazy-registry/disabled-server system blocks (prompt.ts:724-797); StructuredOutput tool + json_schema format for subagent structured returns (prompt.ts:671-678, 1109-1136); pre-flight 95% token check (prompt.ts:800-823).
- No README numeric claim was in scope to verify; sizing is honest from source. Concrete constants: DOOM_LOOP_THRESHOLD=3 (processor.ts:21); COMPACTION_BUFFER=20000, prune PROTECT=40000/MINIMUM=20000 (compaction.ts:30,50-51); retry initial delay 2000ms x2 backoff capped 30s without headers (retry.ts:6-9); pre-flight overflow at 95% and compaction input budget at 80% of limit.
- Cancellation correctness: the cancel() path explicitly rejects queued joiners before deleting session state to avoid a hang in the IngestQueue chain when Esc fires on a running parent loop (prompt.ts:270-287) — a documented bug fix, so treat this ordering as load-bearing.
- Cost caveat baked into code: reasoning tokens are recorded for display but intentionally never billed, because providers report reasoningTokens as a subset of output_tokens; the comment at index.ts:808-814 documents a prior double-charge bug. getUsage also has a known cross-provider inaccuracy for cached-token accounting on non-anthropic providers (index.ts:764-771).
- SSE/streaming is indirect: SessionProcessor calls Session.updatePart/updatePartDelta which publish MessageV2.Event.PartUpdated / PartDelta on the Bus; the HTTP/SSE layer (not in the files read) subscribes to those Bus events. I did not read the server SSE endpoint, so the Bus->SSE bridge is inferred from the event publications, not verified end-to-end.
- Plan-mode reminder injection (insertReminders) is gated behind Flag.CYBERSTRIKE_EXPERIMENTAL_PLAN_MODE and is a no-op by default (prompt.ts:1528-1535), so plan/build agent switching is dormant in the default runtime.

## Key Files
- `packages/cyberstrike/src/session/prompt.ts` — SessionPrompt namespace: prompt entrypoint, the while(true) step loop, model/tier resolution, subtask + compaction dispatch, resolveTools, StructuredOutput tool, system-prompt assembly, pre-flight overflow, shell/command entrypoints, title generation
- `packages/cyberstrike/src/session/processor.ts` — SessionProcessor: consumes ai-sdk fullStream, persists reasoning/text/tool/step/patch parts, doom-loop detection, usage capture, retry/error handling, returns continue/stop/compact
- `packages/cyberstrike/src/session/llm.ts` — LLM.stream: builds and calls ai-sdk streamText — system layering, option/param/header merges, tool-call repair, litellm no-op, wrapLanguageModel middleware, cyberstrike headers
- `packages/cyberstrike/src/session/compaction.ts` — SessionCompaction: isOverflow threshold math, prune of stale tool outputs, process (compaction agent + summary template), create (enqueue compaction turn)
- `packages/cyberstrike/src/session/index.ts` — Session store: updatePart/updatePartDelta publishing Bus PartUpdated/PartDelta events (SSE source), getUsage token+cost computation
- `packages/cyberstrike/src/session/system.ts` — SystemPrompt.provider (model->base prompt selection) and environment block
- `packages/cyberstrike/src/session/instruction.ts` — InstructionPrompt.system: AGENTS.md/CLAUDE.md/config.instructions + URL fetch assembly
- `packages/cyberstrike/src/session/retry.ts` — SessionRetry: retryable classification and exponential backoff honoring retry-after headers
- `packages/cyberstrike/src/tool/registry.ts` — ToolRegistry.tools: agent-scoped, model-gated tool list (codesearch/websearch, apply_patch vs edit)
- `packages/cyberstrike/src/methodology/context.ts` — MethodologyContext.generate: security-methodology status block injected into system prompt when intel exists
