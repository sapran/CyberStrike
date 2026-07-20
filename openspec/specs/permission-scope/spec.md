# Per-Agent Least-Privilege & Engagement-Scope Safety

## Purpose
The safety spine that decides, for every tool call, whether the model is even offered a tool and whether an offered call is allowed to run. Each agent carries a `PermissionNext.Ruleset` of allow/ask/deny rules (with glob sub-rules); tools whose permission is a blanket `deny` are stripped from the toolset BEFORE the model sees them, and every surviving call is re-checked at execution time against the merged agent+session ruleset. Layered on top are hardcoded defense-in-depth denylists (injection-tester bash), a doom-loop repetition guard, and structural lane/scope discipline that keeps offensive-security sub-agents inside their assigned vulnerability class and engagement scope.

## Requirements

### Requirement: Tool Visibility Filtering by Agent Permission
The system SHALL remove a tool from the toolset presented to the model whenever the last permission rule matching that tool's name has pattern `*` and action `deny`, so a blanket-denied tool is never offered.

#### Scenario: Blanket-denied tool hidden
- **WHEN** an agent's ruleset resolves (last matching rule) to `{permission, pattern:"*", action:"deny"}` for a tool name
- **THEN** `PermissionNext.disabled()` adds the tool to the disabled set (next.ts:250-260) and `LLM.resolveTools` deletes it from `input.tools` (llm.ts:262-270), so it is absent from `activeTools` (llm.ts:207) and unknown to the model

#### Scenario: Allowlisted tool survives a `*` deny
- **WHEN** a locked-down agent has `"*":"deny"` followed by `bash:"allow"` (e.g. explore, agent.ts:213-229)
- **THEN** `findLast` matching permission "bash" returns the allow rule whose action is not deny, so bash is NOT disabled and stays visible; only tools with no later allow are hidden

#### Scenario: Glob-scoped deny does not hide the tool
- **WHEN** an agent has `bash:"allow"` plus a glob deny like `bash:{"*DROP TABLE*":"deny"}` (injection tester)
- **THEN** the last rule matching permission "bash" has pattern `*DROP TABLE*` (not `*`), so `disabled()` leaves bash visible (next.ts:257) and the dangerous command is instead blocked at execution time

#### Scenario: Edit-family tools map to one permission
- **WHEN** filtering tool `write`, `patch`, or `multiedit`
- **THEN** they are evaluated under the `edit` permission via `EDIT_TOOLS` (next.ts:248,253), so a single `edit:"deny"` hides the whole edit family

### Requirement: Rule Evaluation Semantics
The system SHALL evaluate a (permission, pattern) request against the merged rulesets by taking the LAST rule whose permission-glob and pattern-glob both match, defaulting to `ask` when nothing matches.

#### Scenario: Last match wins
- **WHEN** multiple rules match the same permission and pattern
- **THEN** `evaluate` flattens rulesets in order via `merge` and returns `findLast(...)` (next.ts:237-246), so a later ruleset overrides an earlier one — merge order equals precedence

#### Scenario: No rule matches
- **WHEN** no rule in the merged set matches the requested permission/pattern
- **THEN** `evaluate` returns `{action:"ask", pattern:"*"}` (next.ts:243), forcing an interactive prompt rather than silent allow

#### Scenario: Config expansion into rules
- **WHEN** config gives a permission a string value vs an object of pattern→action
- **THEN** `fromConfig` emits a single `pattern:"*"` rule for the string form, or one rule per pattern for the object form, expanding `~`/`$HOME` prefixes to the home dir (next.ts:17-24,47-63)

#### Scenario: Glob matching
- **WHEN** a rule pattern contains `*`/`?` or a trailing ` *`
- **THEN** `Wildcard.match` converts it to a regex (`*`→`.*`, `?`→`.`) and makes a trailing ` *` optional so `ls *` matches bare `ls` (wildcard.ts:4-17)

### Requirement: Runtime Permission Gate at Tool Execution
The system SHALL re-check every executing tool call by resolving each requested pattern against the merged agent+session ruleset, throwing `DeniedError` on deny, awaiting user approval on ask, and proceeding only on allow.

#### Scenario: Deny halts the call
- **WHEN** a resolved pattern evaluates to `deny` during `PermissionNext.ask`
- **THEN** it throws `DeniedError` carrying every rule matching that permission name (its human-readable message narrows to the deny rules) (next.ts:142-143,283-284), which propagates back as a tool error and halts that call

#### Scenario: Ask suspends for approval
- **WHEN** a resolved pattern evaluates to `ask`
- **THEN** `ask` registers a pending promise keyed by permission id, publishes `permission.asked` on the bus, and blocks until `reply` resolves it (next.ts:144-158)

#### Scenario: Context wiring merges agent and session
- **WHEN** a tool calls `ctx.ask(...)`
- **THEN** the session builds the ruleset as `merge(agent.permission, session.permission)` so session rules are evaluated last / win (prompt.ts:955-962; subtask path prompt.ts:463-468)

#### Scenario: MCP tools gated generically
- **WHEN** any MCP tool executes
- **THEN** the wrapper calls `ctx.ask({permission:<toolName>, patterns:["*"], always:["*"]})` so MCP tools flow through the same allow/ask/deny gate (prompt.ts:1033-1038)

### Requirement: Ruleset Composition & Least-Privilege Defaults
The system SHALL compose each agent's permission as merge(global defaults, agent-specific allow/deny list, user config), producing broad access for primary agents and tight allowlists for sub-agents.

#### Scenario: Global defaults
- **WHEN** any agent ruleset is built
- **THEN** defaults set `*:allow`, `doom_loop:ask`, `external_directory:ask` (with skill dirs + truncation glob allowed), `question:deny`, and `read` `.env`→ask (agent.ts:158-174)

#### Scenario: Locked-down sub-agents
- **WHEN** building explore, compaction, title, normalize-request, summary, proxy-agent, proxy-analyzer, or a vuln tester
- **THEN** each starts from `"*":"deny"` and re-adds only a small explicit allowlist (e.g. proxy-agent allows only task + read/query tools, agent.ts:462-484; vuln testers agent.ts:563-585)

#### Scenario: User config can override agent defaults
- **WHEN** the user sets a permission in config
- **THEN** the `user` ruleset is merged AFTER the agent defaults/allowlist (agent.ts:183-189 etc.), so user rules normally win via last-match — except denylists merged after `user` (see injection denylist)

#### Scenario: Task-created subagent session locks
- **WHEN** a subagent session is created via the task tool
- **THEN** `session.permission` injects `todowrite`/`todoread` deny and, unless the agent itself has a `task` rule, a `task:"*":"deny"` to prevent unbounded recursion (task.ts:110-135)

### Requirement: Reply Handling: once / always / reject
The system SHALL resolve a pending permission per the user's reply, where reject cancels all of the session's pending requests and always installs new allow rules that auto-clear other now-satisfied pending requests.

#### Scenario: Reject cascades across the session
- **WHEN** the user replies `reject`
- **THEN** the target rejects with `RejectedError` (or `CorrectedError` if a message is given) and every other pending permission for the same session is also rejected (next.ts:180-196)

#### Scenario: Always installs allow rules
- **WHEN** the user replies `always`
- **THEN** each `always` pattern is pushed onto the in-memory `approved` ruleset as an allow rule, and other pending requests now fully satisfied are auto-resolved (next.ts:201-226)

#### Scenario: Approvals are not persisted
- **WHEN** an `always` approval is granted
- **THEN** it lives only in in-memory instance state; the disk-write is commented out pending management UI (next.ts:228-231), so approvals reset next session

### Requirement: Injection-Tester Bash Denylist (Defense-in-Depth)
The system SHALL hard-deny destructive SQL/RCE bash payloads for the proxy-tester-injection agent via glob rules merged after the user ruleset, so user config cannot loosen them.

#### Scenario: Destructive payloads blocked
- **WHEN** the injection tester runs a bash command containing `DROP TABLE`, `xp_cmdshell`, `INTO OUTFILE`, `sqlmap --os-shell`, etc. (both cases listed)
- **THEN** the case-sensitive glob deny rules (agent.ts:593-627) evaluate to deny and the bash tool's `ctx.ask(permission:"bash", patterns:[commandText])` throws `DeniedError` (bash.ts:157-164)

#### Scenario: User cannot re-enable
- **WHEN** a user config tries to allow those bash patterns
- **THEN** the injection denylist is merged AFTER `user` (agent.ts:593-627 wraps `vulnAgentPermission` which already contains user), so it wins via last-match and the deny stands

### Requirement: Doom-Loop Repetition Guard
The system SHALL treat three consecutive identical tool calls as a doom loop and raise a `doom_loop` permission request against the agent's ruleset.

#### Scenario: Three identical calls trip the guard
- **WHEN** the last 3 message parts are the same tool with byte-identical input
- **THEN** the processor fires `PermissionNext.ask({permission:"doom_loop", patterns:[toolName], ruleset: agent.permission})` (processor.ts:151-177), which defaults to `ask` (agent.ts:160) and prompts the user

#### Scenario: Threshold is exactly three
- **WHEN** fewer than 3 identical consecutive calls have occurred
- **THEN** no doom_loop request is raised (`DOOM_LOOP_THRESHOLD = 3`, processor.ts:21,153-156)

### Requirement: Engagement-Scope & Lane Discipline
The system SHALL structurally confine proxy-tester sub-agents to their own vulnerability class for recording/dispatch and provide an advisory scope_check for target-in-scope validation.

#### Scenario: Off-lane VRT check rejected
- **WHEN** a `proxy-tester-<cls>` calls update_vrt_check with a category outside its `TESTER_VRT_SCOPE` lane
- **THEN** `vrtScopeViolation` returns the violation and the tool refuses to write, returning a hand-off message (vrt-check.ts:45-52, vuln-scope.ts:69-82)

#### Scenario: Mis-routed dispatch bounced
- **WHEN** the orchestrator dispatches an objective whose inferred class differs from the target `proxy-tester-<cls>`
- **THEN** `dispatchScopeViolation` triggers and the task tool returns a re-dispatch hint instead of spawning the subagent (task.ts:74-83)

#### Scenario: Task agent visibility filtered by permission
- **WHEN** the task tool builds its list of dispatchable agents
- **THEN** it includes only agents where `evaluate("task", agentName, caller.permission)` is not deny (task.ts:49-52), so `task:{"<agent>":"deny"}` hides that subagent from the caller

#### Scenario: scope_check is advisory only
- **WHEN** scope_check reports a target NOT in scope
- **THEN** it emits a `WARNING: Do NOT perform active testing` string but does not block any tool (scope-check.ts:33-43) — enforcement is the model's responsibility, unlike the structural lane guards

## Notes
- Two-layer enforcement: (1) STATIC visibility — PermissionNext.disabled() only strips a tool when the LAST rule matching its permission NAME has pattern exactly `*` AND action `deny` (next.ts:257); it ignores the pattern otherwise, so glob-scoped denies (e.g. injection bash denylist) do NOT hide the tool and are enforced only at runtime. (2) DYNAMIC — ctx.ask at execution re-evaluates each pattern.
- Precedence = merge order, resolved by `findLast` (last wins). Agent ruleset order is: defaults -> agent allow/deny list -> user config. The injection denylist is merged AFTER user (agent.ts:593-627), which is why the code comment 'user overrides cannot loosen these denies' holds — for the USER ruleset. Note ctx.ask merges session.permission AFTER agent.permission (prompt.ts:960), so a session-level rule would win over even the injection denies; in practice session.permission is set only by the task tool to a fixed small set (task.ts:110-135) and is not user-loosenable for bash, so the denylist holds.
- Default-safe posture: `evaluate` returns `ask` (not allow) when no rule matches (next.ts:243), and the global default `question:"deny"` blocks the interactive question tool for sub-agents (cyberstrike primary re-allows it, agent.ts:186-188).
- `always` approvals are in-memory only — the disk persistence is explicitly commented out pending management UI (next.ts:228-231), so granted allowances reset each session. Not a durable allowlist.
- Lane discipline has two enforcement styles: STRUCTURAL/blocking for update_vrt_check (rejects, vrt-check.ts:45-52) and task dispatch (bounces, task.ts:74-83); but report_vulnerability RECORDS the off-lane finding and only appends a NOTE (tool/vulnerability.ts:82-88) — it does not block. scope_check (scope-check.ts) is purely advisory (text WARNING, no block).
- Provenance: the PermissionNext primitives (Rule/Ruleset/evaluate/ask/reply/disabled), Tool.Context.ask, and llm.ts activeTools filtering are opencode-lineage (upstream sst/opencode ships an equivalent permission namespace). The security-specific hardening is CyberStrike's: the full sub-agent roster and their `*:deny` allowlists, the injection-tester bash denylist, the vuln-scope lane discipline, scope_check, and the methodology/VRT tools. doom_loop is wired as a first-class permission key in this fork (config.ts:729, processor.ts).
- No inflated README numbers were relied on — every claim here is grounded in source. Concrete constants verified: DOOM_LOOP_THRESHOLD=3 (processor.ts:21); proxy-tester default step cap=50 (agent.ts:777-778); EDIT_TOOLS = edit/write/patch/multiedit (next.ts:248).

## Key Files
- `packages/cyberstrike/src/permission/next.ts` — Core PermissionNext module: Rule/Ruleset types, fromConfig/merge, evaluate (last-match-wins), ask/reply runtime gate, disabled() tool-filter, DeniedError/RejectedError/CorrectedError
- `packages/cyberstrike/src/session/llm.ts` — resolveTools (llm.ts:262-270) applies PermissionNext.disabled and sets activeTools (llm.ts:207) — filters tools BEFORE the model sees them
- `packages/cyberstrike/src/agent/agent.ts` — Per-agent least-privilege rulesets: global defaults, locked-down sub-agent allowlists, user merge, injection-tester bash denylist (593-627)
- `packages/cyberstrike/src/tool/tool.ts` — Tool.Context type incl. ctx.ask signature (tool.ts:25); tools call ctx.ask to trigger the runtime gate
- `packages/cyberstrike/src/session/prompt.ts` — Wires ctx.ask -> PermissionNext.ask with merge(agent.permission, session.permission) for both main (955-962) and subtask (463-468) paths; MCP generic gate (1033-1038)
- `packages/cyberstrike/src/session/processor.ts` — Doom-loop detection (threshold 3) raising PermissionNext.ask(permission:doom_loop) against agent.permission (151-177)
- `packages/cyberstrike/src/tool/bash.ts` — Enforces bash + external_directory denies at exec time via ctx.ask with the actual command text as pattern (149-164)
- `packages/cyberstrike/src/tool/vuln-scope.ts` — Structural lane discipline: TESTER_VRT_SCOPE taxonomy + vrtScopeViolation/reportScopeViolation/dispatchScopeViolation guards
- `packages/cyberstrike/src/tool/task.ts` — Task-tool agent visibility filter by task permission (49-52), dispatch lane guard (74-83), subagent session permission locks (110-135)
- `packages/cyberstrike/src/tool/scope-check.ts` — Advisory engagement scope_check (domain/wildcard/CIDR) — warns, does not block
- `packages/cyberstrike/src/util/wildcard.ts` — Wildcard.match glob->regex used by evaluate/disabled for permission and pattern matching
- `packages/cyberstrike/src/config/config.ts` — Config.Permission schema (700-740) defining known permission keys incl. doom_loop, catchall for custom tools
