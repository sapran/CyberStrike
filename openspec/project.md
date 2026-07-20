# CyberStrike — Project Architecture & Extension Reference

> Descriptive reference for this fork. Captures what CyberStrike is, how it is built, and
> how to extend it — derived from a read of the codebase, verified against source with
> `file:line` citations. This is context, not a change proposal. The porting work
> (importing the operator's Claude Code agents/skills) will land as a separate OpenSpec
> change once the agent set is scoped.

## 0. Capability specs

This overview is backed by 16 detailed capability specs under `openspec/specs/` (each with
`SHALL` requirements + `WHEN`/`THEN` scenarios, cited to `file:line`):

| Capability | Covers |
|---|---|
| [`agent-runtime`](specs/agent-runtime/spec.md) | session turn loop, prompt assembly, streaming, compaction, doom-loop |
| [`provider-model`](specs/provider-model/spec.md) | 150+ providers via models.dev, auth loaders, `provider/model` resolution, gateway |
| [`tool-system`](specs/tool-system/spec.md) | `Tool.define` contract, built-in tool catalog, custom/plugin discovery, truncation |
| [`permission-scope`](specs/permission-scope/spec.md) | `PermissionNext` least-privilege, `activeTools` filtering, scope/lane discipline |
| [`skill-system`](specs/skill-system/spec.md) | loader dirs, frontmatter, kill-chain, progressive disclosure, Ed25519 signing tiers |
| [`agent-roster`](specs/agent-roster/spec.md) | native agents + specialists + proxy swarm, native-vs-config/markdown merge |
| [`methodology-engine`](specs/methodology-engine/spec.md) | WSTG phases, coverage, VRT, chain detection (hardcoded TS) |
| [`web-proxy-pipeline`](specs/web-proxy-pipeline/spec.md) | hackbrowser crawl → ingest → proxy-agent → analyzer/testers |
| [`post-exploitation`](specs/post-exploitation/spec.md) | post-exploit skill categories + loading/gating (capability inventory) |
| [`vulnerability-reporting`](specs/vulnerability-reporting/spec.md) | `report_vulnerability`, findings store, VRT scoping/dedup |
| [`mcp-and-bolt`](specs/mcp-and-bolt/spec.md) | MCP config/loading, lazy tool registry, Bolt remote tool execution |
| [`config-extension`](specs/config-extension/spec.md) | config precedence, directory discovery, extend-without-forking |
| [`server-remote-access`](specs/server-remote-access/spec.md) | Hono server, SSE, auth topology, SPA serving, remote/tunnel |
| [`session-storage-sharing`](specs/session-storage-sharing/spec.md) | SQLite/Drizzle store, snapshots, ShareNext → enterprise viewer |
| [`interfaces`](specs/interfaces/spec.md) | CLI command surface, TUI (opentui), ACP, Slack |
| [`deployment-selfhost`](specs/deployment-selfhost/spec.md) | SST tiers, distribution/signing, minimal-vs-full self-host |

Sections 1–7 below are the narrative overview.

## 1. What it is

CyberStrike bills itself as **"the first open-source AI agent built for offensive security"** —
automated penetration testing from the terminal, driven by any LLM the operator can access.

It is a **rebrand-fork of `sst/opencode`** (the open-source AI *coding* agent), retargeted from
software development to autonomous red-teaming. This is explicit, not inferred:

- `script/rebrand.ts` is a checked-in find-replace that rewrites `@opencode-ai/` → `@cyberstrike-io/`,
  `opencode.ai` → `cyberstrike.io`, `OPENCODE_` → `CYBERSTRIKE_`, GitHub org `anomalyco` → `CyberStrikeus`.
- `CHANGELOG.md` 0.1.0: *"Fork of opencode with offensive security focus."*
- License flipped **MIT → AGPL-3.0-only** + a commercial exception (open-core). Residue survives
  (`nix/cyberstrike.nix` still says "The open source coding agent"/MIT; `.github/VOUCHED.td` still
  trusts the SST team; `infra/` hardcodes upstream `thdxr`/`anomalyco`).

The thesis behind the fork: *an autonomous coding agent and an autonomous pentester are the same
machine — different tool belt, prompt corpus, and permission posture.* So the mature opencode agent
loop and provider abstraction are inherited wholesale; the domain is re-skinned.

## 2. Architecture — three layers

Monorepo: Bun + Turbo + SST. `package.json` workspaces over `packages/*`.

### 2.1 Runtime — `packages/cyberstrike` (the engine, ~473 TS files)

opencode-style TypeScript `namespace` modules over lazy `Instance.state()` singletons, wired by a
`Bus` event system, persisted to SQLite via Drizzle. Prompt → tool-call → pentest-action flow:

| File | Role |
|---|---|
| `src/index.ts` | yargs CLI (`run`/`serve`/`web`/`tui`/`acp`/`skill`/`auth`/`mcp`/…); first-run JSON→SQLite migration |
| `src/server/server.ts` | in-process Hono API + SSE `/event` stream; auth middleware; serves the SPA |
| `src/session/prompt.ts` | **the agent turn loop** — assembles system prompt, resolves tools, handles subtasks + compaction |
| `src/session/llm.ts` | the `streamText()` call; **filters tools by `PermissionNext.disabled(agent.permission)`** before the model sees them (per-agent least privilege at the model boundary) |
| `src/session/processor.ts` | consumes the ai-sdk stream → persists parts, doom-loop detection, cost/usage, snapshots |
| `src/provider/provider.ts` | 150+ providers / 5,300+ models from `models.dev`; dynamic npm provider install; per-provider auth quirks |
| `src/tool/registry.ts` | built-in tools + custom/plugin tool discovery from `.cyberstrike` dirs |
| `src/agent/agent.ts` | native agent roster (primary `cyberstrike`, specialists, proxy swarm) |
| `src/config/config.ts` | config precedence + `Config.directories()` discovery of agents/commands/plugins/tools/skills |
| `src/methodology/*` | WSTG-driven phase/coverage engine injected per-turn into the system prompt |

### 2.2 Browser-facing — `packages/app` / `console` / `ui` / `slack` / `enterprise`

- `app` — a **client-only SolidJS SPA** (no server of its own; 100% driven by the local `cyberstrike`
  daemon over HTTP/SSE). Runs as localhost page, LAN service, Cloudflare-tunnel target, or cloud "hub".
- `console` — the SaaS control plane: OpenAuth SSO, Stripe billing, teams, and the metered multi-provider
  **"Zen" gateway** (inherited from opencode; Stripe product still named `ZenBlack`). Optional.
- `enterprise` — the session-**share viewer** app (`cybrstk.us`) — see §5.

### 2.3 Deployment — SST v3, `home: "cloudflare"`

Workers + Durable Objects + R2 + PlanetScale + Stripe. Distributed via npm `@cyberstrike-io/cyberstrike`,
Homebrew, Scoop, `curl | bash`. Three tiers gated in `sst.config.ts` (see §5).

## 3. The intelligence layer (the genuinely CyberStrike-specific work)

Core thesis: **encode offensive-security expertise as data, not code or weights.** Reasoning stays in
the LLM; knowledge lives on disk as version-controlled, Ed25519-signable markdown.

- **Skills** = `<dir>/SKILL.md` = YAML frontmatter (machine index: `category`, `cwe_ids`, `chains_with`,
  `prerequisites`, `severity_boost`, optional signing fields) + methodology body. `chains_with` +
  `severity_boost` build an attack **kill-chain graph** (`src/skill/killchain.ts`).
- **Progressive disclosure**: the `skill` tool does `search`/`load`/`unload` with token accounting — one
  skill resident at a time, which is how "7,600 skills" coexist with a finite context window.
- **13+ agents** are defined **in TypeScript** (`src/agent/agent.ts`): 1 primary + 4 domain specialists
  (web/mobile/cloud/internal-network) + a web-proxy pentest swarm (`proxy-agent` orchestrator → 8 hidden
  `proxy-tester-*` specialists with WSTG skills baked in).
- **Methodology engine** (`src/methodology/`) — bespoke, no opencode analog; injects phase-scoped WSTG
  guidance per turn and tracks coverage.

**Honest sizing:** "7,600+ skills" is ~98% auto-generated framework enumeration (CIS ~5,000, NIST ~1,600,
MITRE ATT&CK ~900). The hand-authored red-team core is ~30 `attack-*`/post-exploit skills + 125 OWASP
WSTG cases (the "120+ OWASP" claim). "13 agents" = the 5 named + 8 proxy testers *pentest-facing* subset;
the full native roster defined in `agent.ts` is **21** (adds 6 utility/plumbing agents — general, explore,
compaction, title, summary, normalize-request — plus `proxy-agent` and `proxy-analyzer`). See the
[`agent-roster`](specs/agent-roster/spec.md) spec.

## 4. Extension model — extend WITHOUT forking

`Config.directories()` (`src/config/config.ts:187`) scan roots: `~/.config/cyberstrike/`, each
`<project>/.cyberstrike/` walking up to the worktree, `~/.cyberstrike/`, and `$CYBERSTRIKE_CONFIG_DIR`.

| Surface | Port existing CC assets? | Fork? | Where |
|---|---|---|---|
| **Skills** | ✅ drop-in, zero changes | No | `~/.claude/skills/`, `.claude/skills/`, `.agents/skills/`, `.cyberstrike/skill(s)/` |
| **Agents** | ⚠️ move file + 3 frontmatter fixes | No | `.cyberstrike/agent(s)/` or inline `agent:` config key |
| **Tools / commands / plugins** | ⚠️ config/dir | No | `.cyberstrike/{tool,command,plugin}/` |
| **MCP** | ⚠️ reshape (schema ≠ CC `.mcp.json`) | No | `mcp` config key |
| **Methodology** (phases/coverage) | ❌ hardcoded TS | **Yes** | `src/methodology/*.ts` |

### 4.1 Skills — Claude-Code compatible

`src/skill/skill.ts:64` `EXTERNAL_DIRS = [".claude", ".agents"]` + `skills/**/SKILL.md` → existing CC
skills load **unchanged**. Only `name:` + `description:` frontmatter is required; the rest is optional
richer metadata (buys kill-chain + search indexing). Toggle off with `skills.disabled` /
`CYBERSTRIKE_DISABLE_EXTERNAL_SKILLS`. Name collisions: last-loaded wins (project can shadow a built-in).

### 4.2 Agents — same format, different folder + 3 breakers

Agent dirs are `{agent,agents}/**/*.md` under the cyberstrike roots (`config.ts:448` `loadAgent`) — **not**
`.claude/agents`. The markdown body becomes the system prompt. When copying a CC agent:

1. `tools: Read, Bash` (comma-STRING) **hard-fails** — schema is `record(string, boolean)`. Convert to a
   `permission:` map (`allow`/`ask`/`deny` per tool) or delete it.
2. `model: sonnet` is invalid — either delete `model:` (it is **optional** and inherits the operator's
   session model) or use fully-qualified `provider/model`.
3. `name:` frontmatter is ignored — the filename is the agent name.

```md
---
# .cyberstrike/agent/recon.md   (filename = agent name)
description: External recon and asset discovery
mode: subagent            # primary | subagent | all
permission:               # NOT `tools:`
  edit: deny
  bash: ask
color: info
---
You are a recon specialist. ...
```

### 4.3 MCP — reshape, don't paste

`mcp` config key uses a `type` discriminator, `command` as full argv, `environment` (not `env`):

```jsonc
"mcp": {
  "my-tool": { "type": "local",  "command": ["npx","-y","some-mcp"], "environment": {"KEY":"{env:MY_KEY}"}, "enabled": true },
  "remote":  { "type": "remote", "url": "https://...", "headers": {"Authorization":"Bearer {env:TOK}"}, "enabled": true }
}
```

### 4.4 Skill signing — the confusing part, resolved

`src/skill/signing.ts` `verify()` tiers: no `sha256` → **`unverified` → loads fine**; hash matches, foreign
signer → `community` → loads; **hash mismatch / bad signature → `tampered` → BLOCKED** (the only blocked
tier, `skill.ts:109`). **Tier gates nothing** — skill access is filtered by *name* via the permission
system, independent of signing. `cyberstrike skill sign` **hardcodes `signed_by: cyberstrike-official`**,
so self-signing with your own key fails verification against the single embedded public key →
`tampered` → your skill stops loading. **For your own skills: write plain `SKILL.md`, no signing fields,
never run `skill sign`.** Becoming your own trust root = `script/sign-skills.ts --generate` (rewrites the
embedded key) + rebuilt binary = a fork.

### 4.5 Model is agnostic

`ModelId` is a free string (`config.ts:37`); `parseModel()` (`provider/provider.ts:1419`) splits on the
first `/`. The `provider/model` form only **disambiguates** the same model across providers (Anthropic
direct / Bedrock / Vertex / OpenRouter / …). `model:` is optional everywhere and inherits the operator's
chosen session model.

## 5. Self-hosting

Three independently-gated tiers (`sst.config.ts` conditional imports):

1. **Local server** (`cyberstrike serve`/`web`) — zero cloud. Needs `CYBERSTRIKE_SERVER_PASSWORD`
   (+ optional `CYBERSTRIKE_SERVER_USERNAME`, default `cyberstrike`) and a tunnel/VPS. Bring your own
   provider keys via `cyberstrike auth login`.
2. **Teams/share app** (`packages/enterprise`, gated `CYBERSTRIKE_ENTERPRISE`) — standalone SolidStart,
   S3/R2 only (no DB/auth/billing). The `cybrstk.us` replacement.
3. **Cloud console** (`packages/console`, gated by `STRIPE_SECRET_KEY && PLANETSCALE_SERVICE_TOKEN`) —
   never needed for a private team.

**Replace `cybrstk.us`:** single knob — `ShareNext.url()` (`src/share/share-next.ts:16`) =
`config.enterprise?.url ?? "https://cybrstk.us"`. Set config `"enterprise": { "url": "https://share.mycorp" }`
(supports `{env:VAR}`). There is no `share.url`/`CYBERSTRIKE_SHARE_URL`. Kill sharing entirely with
`CYBERSTRIKE_DISABLE_SHARE=1` (or config `"share": "disabled"`). A self-hosted server implements `POST /api/share`,
`POST /api/share/:id/sync`, `GET /api/share/:id/data`, `DELETE /api/share/:id`, + a `GET /share/:id` viewer.

**Security caveats (matter for a pentest tool):**
- Sharing defaults to Anomaly's `cybrstk.us` — session/target snapshots leak off-box unless
  `enterprise.url` is set or sharing disabled. Set this before onboarding a team.
- Password auth is **topology-based** — enforced only when `x-forwarded-for`/`cf-connecting-ip` is present,
  so a naive `ssh -L` bypasses it. Use `cloudflared` or a proxy that injects forwarded headers; verify with
  an unauthenticated `curl` expecting `401`. Single shared password, not per-user — front with SSO if needed.
- The UI silently proxies `app.cyberstrike.io` unless you `bun run build` `packages/app`.
- `models.dev` is fetched on startup — override with `CYBERSTRIKE_MODELS_URL`/`_PATH`/`_DISABLE_MODELS_FETCH`
  for air-gap.

## 6. Playground & current work

Porting the operator's own Claude Code agents + skills into CyberStrike. **Testing happens in the
`~/cab/cyberstrike` playground** (not a git repo), which already has a `.cyberstrike/` working session
(`recon-lan` nmap/TLS output + `reports/`) and a `cyberstrike.json` wired to the operator's own
OpenAI-compatible providers — **NVIDIA Build API** and **LiteLLM @ `llm.tilearn.net`** (model
`qwen3.5-122b`). Ported agents should leave `model:` unset (inherit) unless deliberately pinning one of these.

Next steps: (1) scope the operator's CC agents against CyberStrike's native roster to decide port vs skip;
(2) land the porting work as an OpenSpec change; (3) test in the playground.

## 7. Repo conventions (from `AGENTS.md`)

Default upstream branch is `dev`; **this fork uses `develop` as the integration branch, `main` for
releases.** House style: single-word names, early returns over `else`, avoid `try`/`catch`, prefer Bun
APIs, Drizzle snake_case columns, rely on type inference. Tests run from package dirs, never the repo root.
