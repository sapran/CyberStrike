# MCP & Bolt External Tool Integration

## Purpose
CyberStrike integrates external tools through two transports: standard Model Context Protocol (MCP) servers (local stdio or remote HTTP/SSE, with OAuth) and "Bolt" remote tool servers (self-hosted VPS/Docker/Kali boxes reached over MCP-over-HTTP with Ed25519 request signing). To keep hundreds of external tools from flooding the model's context window, tool definitions are not injected eagerly — a lazy registry stores ~50-token metadata per tool and the model pulls full definitions on demand via tool_search/load_tools. Eleven security-focused MCP servers ship pre-seeded but disabled so users can opt in.

## Requirements

### Requirement: MCP Server Configuration Schema
The system SHALL accept MCP servers under config.mcp as a discriminated union of local (type:"local", command[], optional environment/enabled/timeout) and remote (type:"remote", url, optional headers/oauth/enabled/timeout) entries, where remote oauth is either an McpOAuth object (clientId/clientSecret/scope) or the literal false.

#### Scenario: Local server entry
- **WHEN** a config entry has type:"local" with a command array
- **THEN** it validates against McpLocal (config.ts:592-611) and is launched via StdioClientTransport with cmd=command[0], args=rest, cwd=Instance.directory, env = process.env + mcp.environment (index.ts:501-514)

#### Scenario: Remote server entry
- **WHEN** a config entry has type:"remote" with a url
- **THEN** it validates against McpRemote (config.ts:628-650); headers (if present) are passed as requestInit.headers and oauth defaults on unless set to false (index.ts:398-436)

#### Scenario: Untyped entry ignored
- **WHEN** an mcp config entry lacks a "type" field
- **THEN** isMcpConfigured() returns false and the entry is logged and skipped rather than connected (index.ts:204-206, 222-224)

#### Scenario: Bolt entry schema
- **WHEN** a config entry is placed under config.bolt
- **THEN** it validates against Config.Bolt requiring url with optional enabled/timeout (config.ts:655-670, 1188)

### Requirement: Pre-Seeded Disabled Security MCP Servers
The system SHALL seed eleven built-in local security MCP servers at the lowest config precedence, each with enabled:false, so they are advertised but inactive until the user opts in.

#### Scenario: Servers seeded disabled
- **WHEN** config state initializes
- **THEN** result.mcp is populated with github-security, cve, osint, cloud-audit, hackbrowser, darknet, dns-security, supply-chain, mcp-scanner, steganography, satellite — each type:"local", command:["npx","-y",<pkg>], enabled:false (config.ts:82-141)

#### Scenario: User config overrides
- **WHEN** a user config also defines one of these keys
- **THEN** the seed is lowest precedence so user config wins via merge (config.ts:81, 143-256)

#### Scenario: Disabled awareness injected
- **WHEN** a session prompt is built and some servers are disabled
- **THEN** a "# Disabled MCP Servers" system block lists them and tells the agent they can be enabled via `cyberstrike mcp enable <name>` (prompt.ts:756-768)

### Requirement: MCP Connection Lifecycle & Transport Selection
The system SHALL connect local servers over stdio and remote servers by trying StreamableHTTP first then falling back to SSE, verify each connection by listing tools, and record a per-server status (connected/disabled/failed/needs_auth/needs_client_registration).

#### Scenario: Remote transport fallback
- **WHEN** a remote server is created
- **THEN** a StreamableHTTPClientTransport is attempted first, then an SSEClientTransport, each within timeout ?? 30000ms (index.ts:421-446)

#### Scenario: Tool-list verification
- **WHEN** a client connects but listTools() fails or times out
- **THEN** the client is closed and status becomes failed:"Failed to get tools" (index.ts:559-580)

#### Scenario: Local cyberstrike self-spawn
- **WHEN** a local command's cmd equals "cyberstrike"
- **THEN** BUN_BE_BUN=1 is injected into the child env (index.ts:511)

#### Scenario: Disabled skips connect
- **WHEN** an entry has enabled:false
- **THEN** status is set to {status:"disabled"} and no client is created (index.ts:227-231, 374-380)

### Requirement: Remote MCP OAuth Flow
The system SHALL enable OAuth for remote servers by default (unless oauth:false), surface needs_auth / needs_client_registration statuses on 401, and complete a browser-based authorization-code flow with CSRF-protected state through a local callback server.

#### Scenario: OAuth on by default
- **WHEN** a remote server has no oauth field
- **THEN** an McpOAuthProvider is constructed and dynamic client registration (RFC 7591) is attempted when clientId is absent (index.ts:398-419, config.ts:613-625)

#### Scenario: Needs auth on 401
- **WHEN** connect throws UnauthorizedError without a registration hint
- **THEN** the transport is stored in pendingOAuthTransports, status becomes needs_auth, and a toast tells the user to run `cyberstrike mcp auth <key>` (index.ts:456-484)

#### Scenario: State mismatch rejected
- **WHEN** the OAuth callback returns and stored state != generated state
- **THEN** the flow throws "OAuth state mismatch - potential CSRF attack" (index.ts:1252-1257)

#### Scenario: Finish and reconnect
- **WHEN** authenticate() receives the auth code
- **THEN** transport.finishAuth(code) runs, the code verifier is cleared, and add() re-establishes the connection returning the new status (index.ts:1268-1299)

### Requirement: MCP Tool Prefixing & Conversion
The system SHALL expose each connected server's tools under a sanitized `{server}_{tool}` key, wrapping each as an AI-SDK dynamicTool that proxies execution back to the MCP client and returns errors as isError tool results rather than throwing.

#### Scenario: Namespaced key
- **WHEN** MCP.tools() builds the tool map
- **THEN** server and tool names have every [^a-zA-Z0-9_-] char replaced with _ and are joined as sanitizedClientName + "_" + sanitizedToolName (index.ts:1012-1015)

#### Scenario: Only connected servers
- **WHEN** tools() enumerates clients
- **THEN** only clients whose status is "connected" contribute tools (index.ts:985-987)

#### Scenario: Schema hardening
- **WHEN** an MCP tool schema is converted
- **THEN** type is forced to "object" and additionalProperties:false is set before wrapping in dynamicTool (index.ts:153-160)

#### Scenario: Execution error handling
- **WHEN** client.callTool throws during execute
- **THEN** the wrapper returns {content:[text: "Error calling MCP tool..."], isError:true} instead of propagating (index.ts:176-190)

### Requirement: Lazy Tool Registry (Context-Budget Loading)
The system SHALL avoid injecting full external-tool definitions into context by collecting ~50-token metadata (name/summary/keywords/category) per MCP+Bolt tool at init, exposing search/load/unload/list meta-tools, and injecting only loaded tools into the model against a 30000-token budget.

#### Scenario: Metadata-only at init
- **WHEN** LazyToolRegistry.init() runs
- **THEN** for every connected server's tools it stores id=`{server}_{tool}` metadata (summary sliced to 100 chars, keyword extraction, categorize) without loading full definitions (lazy-registry.ts:46-98)

#### Scenario: On-demand loading
- **WHEN** the model calls tool_search then load_tools with returned IDs
- **THEN** search ranks metadata by name/summary/keyword score (lazy-registry.ts:153-194) and load pulls the full converted tool from MCP.tools() into loadedTools (lazy-registry.ts:196-246, tool-search.ts:14-100)

#### Scenario: Loaded-only injection
- **WHEN** a session prompt is assembled and lazyStats.available > 0
- **THEN** mcpToolSource = getLoadedMCPTools() (only loaded tools) instead of the eager MCP.tools() fallback, and a "# MCP Tools" block tells the model the tools are NOT yet in context (prompt.ts:1003-1011, 723-752)

#### Scenario: Live refresh
- **WHEN** a server emits a ToolListChanged notification
- **THEN** the Bus subscription calls refresh(server) to add new tool metadata (lazy-registry.ts:144-151; index.ts:141-146)

### Requirement: Bolt Ed25519 Pairing & Signed Remote Execution
The system SHALL pair with a Bolt server via an admin-token key exchange that yields an Ed25519 keypair and clientId, then connect to `{url}/mcp` over StreamableHTTP where every request is signed with X-Client-Id/X-Timestamp/X-Nonce/X-Signature, storing credentials at 0600.

#### Scenario: Two-step pairing
- **WHEN** BoltAuth.pair(name,url,adminToken) runs
- **THEN** POST /pair (Bearer adminToken) returns a code, a fresh ed25519 keypair is generated, POST /pair/exchange returns clientId + server public key, and credentials are saved with mode 0600 (bolt-auth.ts:61-130, 197-200)

#### Scenario: Request signing
- **WHEN** any Bolt HTTP request is issued
- **THEN** signRequest signs message `{timestamp}\n{nonce}\n{method}\n{path}\n{bodyHash}` with the private key and attaches the four X- headers (bolt-auth.ts:138-158; transport fetch wrapper index.ts:627-645)

#### Scenario: Needs pairing
- **WHEN** a bolt server is enabled but has no stored credentials
- **THEN** status is needs_auth and a toast says "Use /bolt -> add" (index.ts:602-613)

#### Scenario: URL change re-pair
- **WHEN** stored creds.serverUrl differs from the configured bolt.url
- **THEN** credentials are deleted and status becomes needs_auth (index.ts:616-621)

### Requirement: Bolt Server Management & Tool Groups
The system SHALL manage Bolt servers over dedicated /bolt HTTP routes (status/add/pair/connect/disconnect/remove) and enumerate a connected Bolt's advertised plugins and mcp-servers by fetching its signed /tools endpoint.

#### Scenario: REST management
- **WHEN** the /bolt routes are called
- **THEN** GET / returns boltStatus, POST / adds+persists a server to global config, POST /:name/pair pairs via admin token, and connect/disconnect/DELETE manage lifecycle (routes/bolt.ts:10-176, mounted at /bolt in server.ts:286)

#### Scenario: Tool group discovery
- **WHEN** boltToolGroups() runs for a connected Bolt
- **THEN** it signed-fetches `{url}/tools` and maps data.plugins (type:"plugin") and data.mcpServers (type:"mcp-server") into BoltToolGroup entries (index.ts:724-779)

#### Scenario: Removal cleans config
- **WHEN** removeBolt(name) is called
- **THEN** the client is disconnected, the bolt.<name> key is stripped from every discovered config file, and the instance is disposed (index.ts:855-875)

## Notes
- '176+ MCP tools' (README.md:40 and 'MCP Ecosystem ... 176+ security tools across 5 domains' at README.md:311) is NOT verifiable from this repo. The codebase seeds only 11 built-in security MCP servers (config.ts:82-141), all external npm packages invoked via `npx -y <pkg>` — none of their tool definitions live in this repo, so no tool count can be summed from source. The 176 is a marketing aggregate assuming every server enabled with its published tool set; out of the box all 11 are enabled:false, so a fresh install exposes 0 MCP tools until the user opts in.
- The README groups the servers into '5 domains'; the code comments group the same 11 into 3 tiers (Tier 1 Core Security, Tier 2 Extended Intelligence, Tier 3 Specialist) at config.ts:83,109,125. The '5 domains' framing is not reflected in source.
- Timeout discrepancy: the MCP local/remote schema descriptions state 'Defaults to 5000 (5 seconds)' (config.ts:606,645) but the actual runtime default is DEFAULT_TIMEOUT = 30_000ms (index.ts:33). Those two doc strings are stale; effective default is 30s for both MCP and Bolt. The Bolt schema description (config.ts:664) already correctly documents 'Defaults to 30000 (30 seconds)'.
- LazyTool.source is typed as 'mcp'|'plugin'|'builtin' (lazy-registry.ts:29) but init/refresh only ever set source:'mcp' — plugin/builtin lazy sourcing is not wired. Both regular MCP and Bolt clients funnel through the same registry (init merges mcpStatus + boltStatus at lazy-registry.ts:49-51).
- Bolt reuses the MCP Client/StreamableHTTP transport; the only difference from a remote MCP server is the signed-fetch wrapper (index.ts:627-645) and Ed25519 pairing instead of OAuth. Connection target is always `{bolt.url}/mcp` (index.ts:625).
- The '~50 token' metadata vs '~500 token full definition' figures come from the module's own doc comment and constant AVG_TOKENS_PER_TOOL=500 (lazy-registry.ts:14-20,44); they are hardcoded estimates, not measured — the budget-exceeded path only warns, it does not actually evict tools (lazy-registry.ts:203-210).
- Repo is a fork of sst/opencode; the MCP core (transport selection, OAuth, tools() prefixing) is opencode-derived, while Bolt (bolt-auth.ts, routes/bolt.ts, createBolt/boltToolGroups), the lazy registry, tool-search meta-tools, and the seeded security-MCP block are CyberStrike additions.

## Key Files
- `packages/cyberstrike/src/mcp/index.ts` — MCP namespace: connection lifecycle (create/createBolt), transport selection, OAuth flow, tools() prefixing+conversion, Bolt tool groups, add/remove/connect/disconnect
- `packages/cyberstrike/src/config/config.ts` — McpLocal/McpRemote/McpOAuth/Mcp/Bolt Zod schemas (592-670) and the 11 pre-seeded disabled security MCP servers (82-141); mcp/bolt Info fields (1174,1188)
- `packages/cyberstrike/src/tool/lazy-registry.ts` — LazyToolRegistry: ~50-token metadata store, search/load/unload/stats, 30000 budget, ToolsChanged refresh subscription
- `packages/cyberstrike/src/tool/tool-search.ts` — tool_search / load_tools / unload_tools / list_loaded_tools meta-tools
- `packages/cyberstrike/src/mcp/bolt-auth.ts` — BoltAuth: Ed25519 keypair, /pair + /pair/exchange pairing, signRequest header format, signedFetch, 0600 credential storage, healthCheck
- `packages/cyberstrike/src/server/routes/bolt.ts` — /bolt HTTP routes: status/add/pair/connect/disconnect/remove
- `packages/cyberstrike/src/tool/registry.ts` — Registers the four lazy meta-tools; initLazyRegistry() + getLoadedMCPTools() bridge to sessions (158-214)
- `packages/cyberstrike/src/session/prompt.ts` — Wires MCP into the model: loaded-only vs eager tool source (1003-1011), '# MCP Tools' and disabled-server system blocks (723-768)
- `packages/cyberstrike/src/mcp/oauth-provider.ts` — McpOAuthProvider used for remote OAuth (referenced by index.ts create/startAuth)
