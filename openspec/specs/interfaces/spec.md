# User-Facing Interfaces (CLI, TUI, ACP, Slack)

## Purpose
CyberStrike exposes its agent engine through several front ends beyond the web UI: a yargs-based CLI with ~24 top-level commands, an opentui/SolidJS terminal UI (TUI), a headless `run` mode for scripting/CI, an Agent Client Protocol (ACP) server for editor integration (Zed etc.), and a Slack bot that bridges threads to sessions. Every surface talks to the same in-process HTTP server (`Server.App()`/`Server.listen()`) through the generated SDK client (`createCyberstrikeClient`), so the interfaces are thin translators between a transport (stdin/stdout, terminal, Slack events) and the session/prompt/event REST+SSE API. This is a fork of sst/opencode; most of this layer is opencode's interface code renamed to CyberStrike, with security-specific additions (hackbrowser, skill, github, provider).

## Requirements

### Requirement: CLI Command Surface & Bootstrap
The system SHALL register a yargs CLI named `cyberstrike` exposing the TUI as the default command plus ~24 subcommands, applying global logging options and a one-time SQLite migration in middleware before dispatching.

#### Scenario: Default invocation launches TUI
- **WHEN** the binary is run with no subcommand (matching `$0 [project]`)
- **THEN** TuiThreadCommand runs, spawning a worker and the opentui TUI (packages/cyberstrike/src/index.ts:126 registers TuiThreadCommand; thread.ts command is `$0 [project]`)

#### Scenario: Full command set is registered
- **WHEN** the CLI parses argv
- **THEN** it registers acp, mcp, attach, hackbrowser, run, generate, debug, auth, agent, upgrade, uninstall, serve, web, models, stats, export, import, github, pr, session, provider, skill plus `completion`/`--help`/`--version` (index.ts:123-146)

#### Scenario: First-run DB migration
- **WHEN** the marker file `<data>/cyberstrike.db` does not yet exist
- **THEN** middleware prints a one-time migration progress bar via JsonMigration.run before any command handler executes (index.ts:83-121)

#### Scenario: Global flags and env markers
- **WHEN** any command runs
- **THEN** middleware inits logging (honoring `--print-logs`/`--log-level`) and sets `process.env.AGENT=1` and `process.env.CYBERSTRIKE=1` (index.ts:66-80)

### Requirement: Headless Run Mode
The system SHALL provide `cyberstrike run [message..]` that drives a non-interactive session over the SDK — via an in-process fetch handler by default, a real local HTTP server with `--port`, or a remote server with `--attach` — streaming formatted or JSON events and auto-denying all permission prompts.

#### Scenario: Three transports
- **WHEN** `run` executes
- **THEN** with `--attach <url>` it connects to a remote server (run.ts:609-610); with `--port` it starts `Server.listen` and connects to that URL (run.ts:617-627); otherwise it bootstraps and connects in-process via `Server.App().fetch` at baseUrl `http://cyberstrike.internal` (run.ts:630-634)

#### Scenario: Session selection
- **WHEN** flags `--continue`/`--session`/`--fork` are given
- **THEN** it continues the newest root session, a specific id, or forks before continuing, else creates a new session with a deny-`question` permission ruleset (run.ts session() at 381-394, rules at 367-373)

#### Scenario: Non-interactive permission handling
- **WHEN** a `permission.asked` event arrives during the run loop
- **THEN** it logs the request and immediately replies `reject` (auto-rejecting), never prompting the user (run.ts:540-548)

#### Scenario: Output formats
- **WHEN** `--format json` is set
- **THEN** each event (tool_use, text, reasoning, step_start/finish, error) is emitted as a JSON line to stdout instead of the formatted TTY rendering (run.ts:433-437, emit())

### Requirement: Terminal UI Architecture
The system SHALL run the TUI as an opentui/SolidJS render tree hosted in the main process while the CyberStrike server runs in a spawned Worker, communicating by RPC (direct fetch + event forwarding) unless network flags force a real HTTP server.

#### Scenario: Worker + RPC transport
- **WHEN** the TUI starts without explicit `--port`/`--hostname`/`--mdns`
- **THEN** thread.ts spawns worker.ts, uses `createWorkerFetch` to proxy fetch calls and `createEventSource` to receive forwarded events at url `http://cyberstrike.internal` (thread.ts:19-40,161-166)

#### Scenario: Optional HTTP server
- **WHEN** a port/hostname/mdns is explicitly set
- **THEN** the worker's `server` RPC calls `Server.listen` and the TUI connects to the returned URL (worker.ts rpc.server; thread.ts:132-150)

#### Scenario: SolidJS provider tree
- **WHEN** `tui()` renders
- **THEN** it mounts nested context providers (SDK, Sync, Theme, Keybind, Dialog, Command, PromptHistory, etc.) and routes between Home and Session views (app.tsx:130-196,760-768)

#### Scenario: Attach to remote
- **WHEN** `cyberstrike attach <url>` runs
- **THEN** it calls `tui()` directly against the remote URL with optional Basic-auth header from `--password`/`CYBERSTRIKE_SERVER_PASSWORD`, no worker (attach.ts:33-60)

### Requirement: ACP Editor Integration
The system SHALL implement an Agent Client Protocol server (`cyberstrike acp`) over newline-delimited JSON on stdio, translating CyberStrike session events into ACP session updates and bridging permission requests to the editor.

#### Scenario: stdio JSON-RPC wiring
- **WHEN** `acp` starts
- **THEN** it boots an in-process server, builds an SDK client, and wraps stdin/stdout in `ndJsonStream` + `AgentSideConnection` from @agentclientprotocol/sdk (acp.ts handler; agent.ts init/create)

#### Scenario: Capability advertisement
- **WHEN** the client sends `initialize`
- **THEN** the agent returns protocolVersion 1 with loadSession, MCP http/sse, image+embeddedContext prompts, and fork/list/resume session capabilities, plus a `cyberstrike auth login` auth method (agent.ts:506-550)

#### Scenario: Event-to-update translation
- **WHEN** session events stream (tool part status, text/reasoning deltas)
- **THEN** handleEvent emits ACP `tool_call`/`tool_call_update`, `agent_message_chunk`, `agent_thought_chunk`, `plan` (from todowrite), and `usage_update` (agent.ts:178-504,73-120)

#### Scenario: Permission bridging
- **WHEN** a `permission.asked` event fires for a live ACP session
- **THEN** it calls `connection.requestPermission` with once/always/reject options and replies to the SDK accordingly, defaulting to reject on error; `authenticate` itself throws not-implemented (agent.ts:180-260,552-554)

### Requirement: Slack Bot Bridge
The system SHALL provide a Slack bot (@cyberstrike-io/slack) that starts an embedded CyberStrike server and maps each Slack channel+thread to one session, prompting it on messages and posting the assistant reply plus live tool updates back to the thread.

#### Scenario: Thread-to-session mapping
- **WHEN** a non-subtype text message arrives
- **THEN** it derives `sessionKey = channel-thread` (thread_ts or ts), creating and sharing a new session on first message and reusing it thereafter (slack/src/index.ts:58-102)

#### Scenario: Prompt and reply
- **WHEN** a mapped message is processed
- **THEN** it calls `session.prompt` and posts back `response.info.content` or joined text parts via `say(...thread_ts)` (index.ts:104-135)

#### Scenario: Live tool updates
- **WHEN** a `message.part.updated` tool event completes for a tracked session
- **THEN** a global event loop posts `*tool* - title` into the matching channel/thread (index.ts:23-51)

#### Scenario: Socket-mode startup
- **WHEN** the process starts
- **THEN** it constructs a Bolt `App` in socketMode using SLACK_BOT_TOKEN/SIGNING_SECRET/APP_TOKEN and an embedded `createCyberstrike({ port: 0 })` server (index.ts:4-20)

### Requirement: Headless Server & Web Commands
The system SHALL expose `serve` (API-only) and `web` (server plus auto-opened browser UI) headless server commands that require `CYBERSTRIKE_SERVER_PASSWORD` and honor shared network options (port/hostname/mdns/cors).

#### Scenario: Password gate
- **WHEN** `serve` or `web` runs without CYBERSTRIKE_SERVER_PASSWORD set
- **THEN** it prints an error instructing to export the variable and exits (serve.ts:9-14; web.ts:38-45)

#### Scenario: API-only serve
- **WHEN** `serve` starts successfully
- **THEN** it calls `Server.listen({ webUI: false })` and reports connecting from app.cyberstrike.io or `cyberstrike web` (serve.ts:17-21)

#### Scenario: Web opens browser
- **WHEN** `web` starts on 0.0.0.0
- **THEN** it prints local/network/mDNS URLs and opens localhost in the default browser via `open` (web.ts:52-83)

#### Scenario: Network option resolution
- **WHEN** any network command resolves options
- **THEN** CLI flags override config `server.*` for port/hostname/mdns/mdnsDomain/cors, defaulting port 4096 / hostname 127.0.0.1 (network.ts options + resolveNetworkOptions)

### Requirement: Resource Management Subcommands
The system SHALL provide grouped subcommands to manage agents, skills, MCP servers, custom providers, credentials, and sessions, each operating against the instance/SDK.

#### Scenario: Agent management
- **WHEN** `agent create`/`agent list` runs
- **THEN** create scaffolds an LLM-generated agent markdown (interactive @clack/prompts or fully non-interactive with path+description+mode+tools), list prints agents with mode+permissions (agent.ts:31-247)

#### Scenario: Skill management
- **WHEN** `skill <sub>` runs
- **THEN** it supports list, search (by keyword/tech/cwe/category), verify (Ed25519 integrity), create, install (remote URL), remove, and sign — a CyberStrike security-specific surface (skill.ts:26-63)

#### Scenario: MCP and provider management
- **WHEN** `mcp` or `provider` subcommands run
- **THEN** mcp exposes list/auth/add/logout/debug for MCP servers; provider exposes add/list/remove for OpenAI-compatible custom providers (mcp.ts:52-596; provider.ts:10-197)

#### Scenario: Credentials and sessions
- **WHEN** `auth` or `session`/`export`/`import` runs
- **THEN** auth manages provider login/logout/list; session lists sessions and export/import serialize session JSON to/from file or share URL (auth.ts:162-379; session.ts:38-56; export.ts/import.ts)

### Requirement: Hackbrowser & GitHub Surfaces
The system SHALL provide `hackbrowser <target>` (crawl a web app then open the TUI for live analysis) and a `github` agent surface (install/run) plus `pr <number>` for PR checkout, as CyberStrike-specific offensive-security entry points.

#### Scenario: Hackbrowser crawl + TUI
- **WHEN** `hackbrowser <target>` runs
- **THEN** it starts a real local server, creates a session, launches a (headless unless credentials/headfull) Playwright crawl, and opens the TUI whose sidebar updates live; onExit stops the crawl (hackbrowser.ts:43-90)

#### Scenario: GitHub agent run
- **WHEN** `github run` executes in Actions or with `--event`/`--token` mock
- **THEN** it routes supported GitHub events (issue_comment, issues, pull_request*, schedule, workflow_dispatch, etc.) to an agent session and reacts on the issue/PR (github.ts:406-470)

#### Scenario: PR checkout
- **WHEN** `pr <number>` runs
- **THEN** it fetches and checks out the GitHub PR branch, then runs cyberstrike in it (pr.ts:30,97)

## Notes
- Provenance: this is a fork of sst/opencode. run.ts, the TUI (opentui/SolidJS), acp/agent.ts, serve/web, and the Slack package are opencode interface code renamed to CyberStrike (baseUrl `http://cyberstrike.internal`, env `CYBERSTRIKE=1`). Security-specific ADDITIONS are hackbrowser, skill, github, provider, and the report_vulnerability tool rendering in run.ts.
- All interfaces are thin translators over the same server: they obtain an SDK client (`createCyberstrikeClient` v2, or `createCyberstrike` v1 in Slack) and drive session.create/prompt/command + event.subscribe. There is no separate business logic in the interface layer.
- Default `run` and default TUI use NO network socket — they call `Server.App().fetch()` in-process (run.ts:630-634) or proxy fetch through the worker via RPC (thread.ts). A real HTTP listener only appears with `--port`/`--attach` (run) or explicit network flags (TUI).
- Slack bot is demo-quality: it uses the OLDER v1 SDK (`@cyberstrike-io/sdk` `createCyberstrike`, not `/sdk/v2`), types the client as `any`, is heavily console.log/emoji-instrumented, and builds replies from a synchronous `session.prompt` result (`response.info.content` or joined text parts) while streaming tool updates separately. It is a single 145-line file, package version 1.1.15.
- ACP `authenticate()` explicitly throws 'Authentication not implemented' (agent.ts:552-554); auth is expected to happen out-of-band via `cyberstrike auth login`. Session list/fork/resume are exposed as `unstable_*` methods.
- `run --format json` emits newline-delimited JSON events (tool_use/text/reasoning/step_start/step_finish/error); default format is human-formatted with per-tool renderers (bash/glob/grep/read/edit/write/task/skill/report_vulnerability/etc.) in run.ts:73-260.
- `serve` and `web` hard-require `CYBERSTRIKE_SERVER_PASSWORD` and exit without it; `attach` sends Basic auth `cyberstrike:<password>`. Worker/event forwarding also injects this Basic header (worker.ts getAuthorizationHeader).
- `debug` groups internal tools: config, lsp, ripgrep, file, scrap, skill, snapshot, agent, paths, wait (debug/index.ts:18-42). `generate` dumps the server's OpenAPI spec (with JS code samples) to stdout — used to generate the SDK, not a user-facing runtime surface.
- Windows-specific input guarding (win32DisableProcessedInput / CtrlCGuard) wraps every TUI/attach entry to prevent the OS from killing the process group before the worker spawns (thread.ts, attach.ts, app.tsx).

## Key Files
- `packages/cyberstrike/src/index.ts` — CLI entry point: yargs setup, global options, first-run DB migration middleware, registration of all ~24 top-level commands
- `packages/cyberstrike/src/cli/cmd/run.ts` — Headless run mode: three transports (in-process/--port/--attach), session create/continue/fork, formatted vs JSON streaming, auto-reject permissions
- `packages/cyberstrike/src/cli/cmd/tui/app.tsx` — TUI render root: opentui render() call, SolidJS provider tree, command palette registration, route switch (Home/Session)
- `packages/cyberstrike/src/cli/cmd/tui/thread.ts` — Default TUI command: spawns worker, worker-fetch/event-source RPC transport vs real HTTP server decision
- `packages/cyberstrike/src/cli/cmd/tui/worker.ts` — TUI worker: runs the CyberStrike server in a Worker, exposes fetch/server/checkUpgrade/reload/shutdown RPC and forwards events
- `packages/cyberstrike/src/cli/cmd/tui/attach.ts` — `attach <url>`: connects the TUI to a remote server with optional Basic auth
- `packages/cyberstrike/src/acp/agent.ts` — ACP Agent implementation: initialize capabilities, session lifecycle (new/load/fork/resume/list), event->sessionUpdate translation, permission bridge
- `packages/cyberstrike/src/cli/cmd/acp.ts` — `acp` command: ndJsonStream over stdio, AgentSideConnection wiring, in-process server + SDK client
- `packages/cyberstrike/src/acp/session.ts` — ACPSessionManager: in-memory map of ACP sessionId -> {cwd, model, variant, modeId}, create/load/get
- `packages/slack/src/index.ts` — Slack bot: Bolt socket-mode app, channel+thread<->session map, prompt bridging and live tool-update posting (145 lines, single file)
- `packages/cyberstrike/src/cli/cmd/serve.ts` — `serve` headless API-only server, password-gated
- `packages/cyberstrike/src/cli/cmd/web.ts` — `web` server + auto-open browser UI, network IP discovery
- `packages/cyberstrike/src/cli/network.ts` — Shared network options (port/hostname/mdns/cors) and config-vs-flag resolution used by serve/web/acp/tui
- `packages/cyberstrike/src/cli/cmd/skill.ts` — Skill management subcommands (list/search/verify/create/install/remove/sign) — CyberStrike security addition
- `packages/cyberstrike/src/cli/cmd/hackbrowser.ts` — `hackbrowser <target>`: crawl web app + open TUI for live analysis
