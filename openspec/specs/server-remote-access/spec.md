# Hono API Server and Remote Access

## Purpose
The `Server` namespace (packages/cyberstrike/src/server/server.ts) is a single lazily-built Hono app that exposes the entire CyberStrike control surface over HTTP: session/agent/MCP/skill/provider/config/methodology/bolt/tui routes, an SSE event stream that re-broadcasts the internal Bus, and (in `web` mode) the SPA itself. It is fronted by a small middleware stack — CORS, a topology-based auth gate keyed on `CYBERSTRIKE_SERVER_PASSWORD`, request logging, and per-request `Instance.provide`. Two CLI commands start it (`serve` = API-only, `web` = API + bundled SPA), and browsers reach a running server three ways: locally/LAN via `cyberstrike web`, from anywhere via a Cloudflare Tunnel in front of the loopback bind, or via the hosted `app.cyberstrike.io` "hub" SPA that connects to a user-supplied server URL with Basic Auth. The auth model trusts the loopback source IP plus a two-header proxy heuristic rather than any token, which is the security-relevant gap.

## Requirements

### Requirement: HTTP API Route Surface
The system SHALL mount the CyberStrike control API as a single Hono app whose sub-routers cover global, session, mcp, skill, provider, config, methodology, bolt, tui, project, pty, permission, question, file and experimental operations, plus an OpenAPI document.

#### Scenario: Feature route groups mounted
- **WHEN** the app is built via Server.App()
- **THEN** `.route()` mounts include /global, /project, /pty, /config, /experimental, /session, /permission, /question, /provider, / (FileRoutes), /mcp, /bolt, /tui, /methodology and /skill (server.ts:168, 276-288, 472)

#### Scenario: OpenAPI + ad-hoc endpoints
- **WHEN** a client GETs /doc, /path, /vcs, /command, /agent, /lsp, /formatter or POSTs /log, /instance/dispose
- **THEN** each is served by an inline handler on the root app and /doc returns a generated OpenAPI 3.1.1 spec titled "cyberstrike" (server.ts:262-274, 311-471)

#### Scenario: Directory resolution per request
- **WHEN** a request carries `directory` query or `x-cyberstrike-directory` header (or neither)
- **THEN** the value is URL-decoded, remembered as Server.lastActiveDirectory, and the handler runs inside Instance.provide({directory}); requests without one fall back to lastActiveDirectory then process.cwd() (server.ts:231-261)

#### Scenario: Error normalization
- **WHEN** a handler throws
- **THEN** onError maps NotFoundError->404, ModelNotFoundError->400, Worktree* errors->400, other NamedError->500, HTTPException passes through, everything else->NamedError.Unknown 500 (server.ts:67-84)

### Requirement: Bus Event SSE Streaming
The system SHALL stream internal Bus events to clients over Server-Sent Events at GET /event (and global events at /global/event), with proxy-buffer padding, a periodic heartbeat, and a JSON long-poll fallback at /global/event/poll.

#### Scenario: Subscribe to the Bus
- **WHEN** a client opens GET /event
- **THEN** the handler sets Cache-Control/X-Accel-Buffering/Connection headers, subscribes via Bus.subscribeAll and writes each event as SSE JSON, closing the stream on Bus.InstanceDisposed (server.ts:515-575)

#### Scenario: Defeat proxy buffering
- **WHEN** the SSE stream opens
- **THEN** it first writes eight ~4KB comment padding lines (~32KB) to flush Cloudflare/Nginx buffers, then a server.connected frame (server.ts:538-545; identical pattern in global.ts:106-115)

#### Scenario: Keep-alive heartbeat
- **WHEN** an SSE connection stays open
- **THEN** a server.heartbeat frame is emitted every 30s to prevent the WKWebView 60s idle timeout, cleared on abort (server.ts:556-572)

#### Scenario: Poll fallback behind a tunnel
- **WHEN** SSE is unavailable and the client GETs /global/event/poll
- **THEN** the server collects GlobalBus events for up to 5s (resolving early once any arrive) and returns them as a JSON array (global.ts:146-195)

### Requirement: Topology-Based Auth Middleware
The system SHALL gate requests with an auth middleware that is a no-op when no password is set, bypasses auth for loopback non-proxied requests, and otherwise requires HTTP Basic credentials matching CYBERSTRIKE_SERVER_USERNAME (default "cyberstrike") and CYBERSTRIKE_SERVER_PASSWORD.

#### Scenario: No password configured
- **WHEN** CYBERSTRIKE_SERVER_PASSWORD is unset
- **THEN** the middleware immediately calls next() for every request -- the API is fully unauthenticated (server.ts:115-116)

#### Scenario: Loopback bypass
- **WHEN** a password is set and the TCP peer address is 127.0.0.1/::1/::ffff:127.0.0.1 with no `x-forwarded-for` and no `cf-connecting-ip` header
- **THEN** the request is treated as local and passes without credentials (server.ts:121-125)

#### Scenario: Remote/proxied enforcement
- **WHEN** the request is non-loopback, or carries `x-forwarded-for`/`cf-connecting-ip`
- **THEN** a matching Basic auth header is required; user/password are compared after base64-decoding, and failure returns JSON {error:"Unauthorized"} 401 (server.ts:136-149)

#### Scenario: Suppressed native dialog
- **WHEN** an unauthorized proxied request is rejected
- **THEN** the 401 intentionally omits the WWW-Authenticate header so browsers show no native Basic-auth prompt and the SPA's own form is used instead (server.ts:134-135, 149)

### Requirement: Static Asset Auth Exemption
The system SHALL exempt web UI static assets from the auth gate so the SPA shell can load in remote mode before the user supplies credentials.

#### Scenario: Root and hashed assets skip auth
- **WHEN** a proxied request targets "/", a path under /assets/, or any path ending in .html/.js/.css/.png/.jpg/.svg/.ico/.woff(2)/.ttf/.webmanifest/.map
- **THEN** the auth middleware returns next() without checking credentials (server.ts:127-133)

#### Scenario: Shell served unauthenticated then app authenticates
- **WHEN** an unauthenticated remote browser loads the app shell
- **THEN** index.html and its static bundle are served, and the SPA subsequently attaches Basic Auth to API/SSE calls via its own connection layer (server.ts:127-133 + utils/server.ts:12-30)

### Requirement: Auth Trust Gap On Header-Stripping Proxies
The system SHALL derive "is this request remote" purely from the TCP peer being non-loopback OR the presence of `x-forwarded-for`/`cf-connecting-ip`, which means any proxy or forwarder terminating on loopback that does not inject one of those two headers bypasses authentication entirely.

#### Scenario: Cloudflared works because it sets the headers
- **WHEN** cloudflared forwards a remote request from localhost
- **THEN** it always sets X-Forwarded-For/CF-Connecting-IP, so `proxied` is true and Basic auth is enforced (server.ts:117-124, matching README:263)

#### Scenario: Header-less forwarder bypasses auth
- **WHEN** a different front-end (ssh -L/-R, socat, a raw TCP forward, an nginx/Caddy block without proxy_set_header, or a tunnel that omits both headers) delivers a remote request from 127.0.0.1
- **THEN** loopback && !proxied is true and the request is served with no credentials -- full unauthenticated remote access (server.ts:121-125)

#### Scenario: Any local process is trusted
- **WHEN** any other user, container sharing the host loopback, or local process on a multi-user host connects to 127.0.0.1:4096
- **THEN** it reaches the entire API and Bus stream unauthenticated regardless of the configured password (server.ts:121-125)

#### Scenario: Missing requestIP fails closed
- **WHEN** c.env.requestIP is unavailable (non-Bun runtime) so the peer address is undefined
- **THEN** loopback is false and auth is enforced -- the failure direction is safe, but the trust model still rests on source-IP topology rather than a token (server.ts:121-123)

### Requirement: CORS Origin Policy
The system SHALL reflect the request Origin only for local dev origins, the Tauri desktop origins, any https *.cyberstrike.io host, and explicitly whitelisted origins, allowing credentials and a fixed header set.

#### Scenario: Allowed origins reflected
- **WHEN** the Origin is http://localhost:<port>, http://127.0.0.1:<port>, tauri://localhost / http(s)://tauri.localhost, matches ^https://([a-z0-9-]+\.)*cyberstrike\.io$, or is in the --cors/config whitelist
- **THEN** that exact origin is echoed back with credentials:true (server.ts:85-111)

#### Scenario: Disallowed origin blocked
- **WHEN** the Origin is any other host (including http:// cyberstrike.io or an unlisted domain)
- **THEN** the origin function returns undefined and no CORS allow header is emitted (server.ts:107)

#### Scenario: Header allowlist
- **WHEN** a cross-origin request is made
- **THEN** only Authorization, Content-Type and x-cyberstrike-directory are advertised as allowed headers and OPTIONS preflights skip the auth middleware (server.ts:110, 114)

### Requirement: Web UI Serving And Fallback Chain
The system SHALL, only when web UI serving is enabled, serve the SPA from the first existing dist candidate with a client-side-routing fallback, then proxy unresolved paths to https://app.cyberstrike.io, while API-only mode returns a 404 pointer instead.

#### Scenario: Dist directory resolution
- **WHEN** the catch-all first runs
- **THEN** it probes, in order, $XDG_DATA_HOME/cyberstrike/web (Global.Path.data+"/web"), <cwd>/packages/app/dist, and ../../../app/dist relative to import.meta.url, caching the first containing index.html (server.ts:589-614)

#### Scenario: SPA client-route fallback
- **WHEN** a requested path is not a real file under the resolved dist dir
- **THEN** index.html is returned with a strict CSP and Cache-Control:no-cache so client-side routes resolve (server.ts:617-638)

#### Scenario: Remote proxy fallback
- **WHEN** no local dist is found
- **THEN** the request is proxied to https://app.cyberstrike.io<path> with a rewritten Host header (CSP re-applied), and on failure returns a 503 build/deploy hint (server.ts:640-656)

#### Scenario: API-only mode
- **WHEN** the server was started with webUI:false (the `serve` command)
- **THEN** the catch-all returns 404 {error:"API-only mode. Use app.cyberstrike.io or 'cyberstrike web'..."} and never serves the SPA (server.ts:576-582, serve.ts:18)

### Requirement: Server Bootstrap And Network Binding
The system SHALL start via `cyberstrike serve` (API-only) or `cyberstrike web` (API + SPA), both requiring CYBERSTRIKE_SERVER_PASSWORD, defaulting the bind to loopback, and publishing mDNS only when bound to a non-loopback host.

#### Scenario: Password required to expose a server
- **WHEN** either command runs without CYBERSTRIKE_SERVER_PASSWORD set
- **THEN** it prints an error and process.exit(1) before Server.listen is called (serve.ts:11-16, web.ts:36-43)

#### Scenario: Default loopback bind
- **WHEN** no --hostname/--mdns/config override is given
- **THEN** hostname defaults to 127.0.0.1 on port 4096, and Bun.serve falls back to an ephemeral port if the requested one is taken (network.ts:13-14, server.ts:699-700)

#### Scenario: mDNS forces 0.0.0.0
- **WHEN** --mdns is passed (or config server.mdns) without an explicit hostname
- **THEN** hostname is switched to 0.0.0.0 and MDNS.publish runs only because the host is non-loopback; a loopback host with mdns logs a warning and skips publishing (network.ts:50-54, server.ts:704-714)

#### Scenario: web prints LAN URLs
- **WHEN** `cyberstrike web` binds 0.0.0.0
- **THEN** it prints localhost plus each non-internal, non-172.x IPv4 network address and opens the browser at localhost (web.ts:9-76)

### Requirement: Client Connection Layer And Remote-Access Topologies
The system SHALL let the SPA connect to a chosen server -- local via `cyberstrike web`, remote via a fronting tunnel, or via the app.cyberstrike.io hub connect screen -- attaching Basic Auth to all calls, polling /global/health for reachability and 401->needsAuth, and degrading SSE to long-polling for cross-origin/buffered connections.

#### Scenario: Hub mode connect screen
- **WHEN** the SPA is loaded from cyberstrike.io or *.cyberstrike.io with no defaultUrl
- **THEN** isHub is true, the default server URL is blank, and HubGate renders HubConnectScreen offering localhost:4096 or a manual URL+username+password form (app.tsx:130,193-210; hub-connect.tsx:24-42)

#### Scenario: Basic Auth on every request
- **WHEN** a stored server connection has a password
- **THEN** createSdkForServer and the raw fetch helper add Authorization: Basic base64(username|"cyberstrike":password) to API, SSE and health calls (utils/server.ts:4-30, global-sdk.tsx:124-132,322-331)

#### Scenario: Health / auth detection
- **WHEN** the client polls GET /global/health every 10s
- **THEN** a 401 sets needsAuth (prompting for credentials) while healthy===true gates the app; HTTPS pages skip HTTP-localhost checks due to Chrome Private Network Access (server-health.ts:88-92, context/server.tsx:147-176)

#### Scenario: SSE-to-poll degradation for tunnels
- **WHEN** the event URL is cross-origin or SSE fails twice / yields no data within 5s
- **THEN** the client switches to long-polling /global/event/poll instead of streaming /global/event (global-sdk.tsx:134-176,190-193)

## Notes
- THE FLAG -- auth is topology-based, not token-based. The gate (server.ts:121-125) trusts any request whose TCP peer is loopback AND that lacks both x-forwarded-for and cf-connecting-ip. Cloudflared happens to set those headers, so the documented tunnel path is protected, but ANY other loopback-terminating forwarder that omits them (ssh -L/-R, socat, nginx/Caddy without proxy_set_header, some ngrok/tailscale configs) yields full unauthenticated remote access. Spoofing the headers only forces auth ON (fail-closed direction), so the exploitable direction is header stripping, not injection.
- Second exposure from the same model: on a multi-user or shared-loopback host (other local users, containers on host network), every local process reaches the whole API and Bus SSE stream unauthenticated even with a password set. There is no per-user or session credential -- only the loopback bypass.
- Third: when CYBERSTRIKE_SERVER_PASSWORD is unset the middleware is a pure no-op (server.ts:115-116). The password requirement is enforced only by the serve/web CLI handlers; anything else calling Server.listen() gets an open server. Default bind is loopback (127.0.0.1:4096) which limits blast radius, but --mdns/config can bind 0.0.0.0.
- README:263 claims "Every API request requires Basic Auth." That is inflated: (a) loopback non-proxied requests bypass auth entirely, (b) all static-asset paths ("/", /assets/, and anything ending .html/.js/.css/.map/etc.) skip auth even when remote (server.ts:127-133), and (c) with no password nothing is required. Accurate statement: proxied/non-loopback non-asset requests require Basic Auth.
- "~/.cyberstrike/web" in the task/README is shorthand. The actual first SPA candidate is path.join(Global.Path.data, "web") = $XDG_DATA_HOME/cyberstrike/web (e.g. ~/.local/share/cyberstrike/web on Linux, ~/Library/... on macOS), not literally ~/.cyberstrike (global/index.ts:8).
- Three remote-access paths distilled: (1) web -- `cyberstrike web`, local/LAN browser, SPA served from local dist, loopback by default or 0.0.0.0 with mDNS; (2) tunnel -- Cloudflare Tunnel in front of the loopback server (README:246-264), which is why SSE padding, 30s heartbeat and the /global/event/poll long-poll fallback exist; (3) hub -- hosted static app.cyberstrike.io (or self-hosted packages/app/dist) that the browser loads then points at a user-supplied server URL via HubConnectScreen. In `web` mode the catch-all also proxies unresolved paths to app.cyberstrike.io as a last-resort SPA source (server.ts:640-656).
- The SSE stream re-broadcasts the in-process Bus via Bus.subscribeAll (server.ts:546) / GlobalBus for global events; there is no filtering by directory or auth scope inside the stream -- auth is entirely at the middleware boundary, so once the /event connection is authorized (or bypassed) the client receives all events for the active instance.
- CORS reflects any http://localhost:PORT and http://127.0.0.1:PORT origin and any https *.cyberstrike.io with credentials:true (server.ts:85-111); combined with the loopback auth bypass, a malicious local web page served from localhost could make credentialed same-machine requests -- another consequence of the loopback trust model rather than a separate bug.
- This spec reverse-engineers the CyberStrike fork of opencode; the server/auth/CORS/SSE scaffolding and the web/hub connection layer are inherited opencode patterns (Hono + SolidJS SPA), while the CYBERSTRIKE_SERVER_PASSWORD topology gate, the /global/event/poll tunnel fallback, the app.cyberstrike.io hub proxy, and the mDNS binding are CyberStrike-specific additions.

## Key Files
- `packages/cyberstrike/src/server/server.ts` — Hono app: route mounts, CORS, topology auth middleware, SSE /event, SPA serving + app.cyberstrike.io proxy fallback, listen()/mDNS
- `packages/cyberstrike/src/server/routes/global.ts` — /global/health, /global/version-check, /global/event (SSE) and /global/event/poll long-poll fallback, /global/config, /global/dispose
- `packages/cyberstrike/src/cli/cmd/serve.ts` — Headless API-only server command; enforces CYBERSTRIKE_SERVER_PASSWORD; starts with webUI:false
- `packages/cyberstrike/src/cli/cmd/web.ts` — Server + bundled SPA command; password gate; prints localhost/LAN/mDNS URLs and opens browser
- `packages/cyberstrike/src/cli/network.ts` — port/hostname/mdns/cors option resolution; default hostname 127.0.0.1; mdns forces 0.0.0.0
- `packages/cyberstrike/src/flag/flag.ts` — CYBERSTRIKE_SERVER_PASSWORD / CYBERSTRIKE_SERVER_USERNAME env flags read by auth middleware and CLI
- `packages/app/src/context/server.tsx` — SPA server-connection store: active server, health polling, hub-mode readiness, localhost vs remote
- `packages/app/src/context/global-sdk.tsx` — SPA event pipeline: authed SSE stream with cross-origin/failure degradation to long-poll; SDK/fetch factories
- `packages/app/src/utils/server.ts` — basicAuth() header builder and createSdkForServer() that injects Authorization into the typed client
- `packages/app/src/utils/server-health.ts` — GET /global/health probe with retry/cache; 401 -> needsAuth signal driving the hub connect prompt
- `packages/app/src/components/hub-connect.tsx` — Hub connect screen: localhost vs remote URL + username/password form used by app.cyberstrike.io
- `packages/app/src/app.tsx` — Hub-mode detection (cyberstrike.io host), HubGate, default server URL resolution
- `README.md` — Documented remote-access model (Cloudflare Tunnel, security claims) -- lines 242-276
