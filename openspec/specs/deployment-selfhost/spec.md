# Deployment, Distribution & Self-Host Topology

## Purpose
Defines how CyberStrike is built, deployed to the cloud (SaaS), distributed to end users, and self-hosted. A single SST program composes three conditional tiers on a Cloudflare "home": an always-on app tier (API worker + docs + web UI), a Stripe/PlanetScale-gated console (the paid Zen/SaaS backend), and a flag-gated enterprise tier. The CLI itself ships as Bun-compiled single-file platform binaries distributed through npm, curl, and community Homebrew/Scoop/Chocolatey channels. As a fork of sst/opencode, it is AGPL open-core with hardcoded anomalyco / cyberstrike.io / CyberStrikeus identifiers that a real self-hoster of the cloud tier must replace.

## Requirements

### Requirement: Conditional SST Tier Composition
The system SHALL deploy the app tier unconditionally and conditionally include the console and enterprise tiers based on environment variables, gating the Stripe and PlanetScale Pulumi providers on the same secrets.

#### Scenario: App tier always runs
- **WHEN** sst deploy runs for any stage
- **THEN** infra/app.js is always imported (sst.config.ts:17), provisioning the Api Worker, docs Astro site and WebApp static site regardless of other env vars

#### Scenario: Console gated on Stripe + PlanetScale
- **WHEN** both STRIPE_SECRET_KEY and PLANETSCALE_SERVICE_TOKEN are set
- **THEN** infra/console.js is imported (sst.config.ts:18); with neither the stripe provider nor the planetscale@0.4.1 provider is even registered (sst.config.ts:11-12)

#### Scenario: Enterprise gated on a single flag
- **WHEN** CYBERSTRIKE_ENTERPRISE is set (any value)
- **THEN** infra/enterprise.js is imported, deploying the Teams SolidStart app (sst.config.ts:19)

#### Scenario: CI supplies the gating secrets
- **WHEN** the deploy workflow runs on dev/production
- **THEN** it passes CLOUDFLARE_API_TOKEN, PLANETSCALE_SERVICE_TOKEN(_NAME) and a stage-selected STRIPE_SECRET_KEY (.github/workflows/deploy.yml:36-41), so CI deploys app+console; enterprise is not triggered there

### Requirement: Cloudflare Home Topology
The system SHALL host every deployed resource on Cloudflare via SST's cloudflare home, using Workers, R2 buckets, KV, static sites and framework adapters under stage-derived cyberstrike.io / cybrstk.us domains.

#### Scenario: Cloudflare is the only home
- **WHEN** the SST app config is evaluated
- **THEN** home is set to "cloudflare" (sst.config.ts:9) so all Linkables resolve to Cloudflare resources

#### Scenario: App-tier resources
- **WHEN** the app tier deploys
- **THEN** it creates a Cloudflare Worker Api at api.<domain> (infra/app.ts:13), an Astro docs site at docs.<domain> (infra/app.ts:51), a StaticSite WebApp at app.<domain> built by `bun turbo build` (infra/app.ts:61-67), plus an R2 Bucket and a SyncServer durable object (infra/app.ts:11,33-40)

#### Scenario: Stage-derived domains
- **WHEN** the stage is production / dev / other
- **THEN** domain resolves to cyberstrike.io / dev.cyberstrike.io / <stage>.dev.cyberstrike.io and shortDomain to cybrstk.us variants against a hardcoded Cloudflare zoneID 430ba34c... (infra/stage.ts:1-20)

### Requirement: Console (SaaS) Tier
The system SHALL provision the paid cloud backend — PlanetScale database, OpenAuth worker, Stripe billing for the CyberStrike Black plan, and the Console SolidStart app — only when Stripe and PlanetScale credentials are present.

#### Scenario: PlanetScale database on anomalyco org
- **WHEN** the console tier deploys
- **THEN** it reads/creates a branch of the `cyberstrike` database under the hardcoded `anomalyco` PlanetScale organization and mints a per-stage password (infra/console.ts:8-31)

#### Scenario: Stripe Zen Black billing
- **WHEN** the console tier deploys
- **THEN** it creates a Stripe product "CyberStrike Black" with three monthly USD prices at $200 / $100 / $20 (unitAmount 20000/10000/2000, infra/console.ts:103-116) and a webhook endpoint at https://<domain>/stripe/webhook (infra/console.ts:71-72)

#### Scenario: Zen model + auth wiring
- **WHEN** the Console SolidStart app is linked
- **THEN** it binds 20 ZEN_MODELS secrets, the PlanetScale database, Stripe keys, an OpenAuth Worker at auth.<domain>, a gateway KV and R2 buckets, with smart placement and a Honeycomb log-processor tail consumer (infra/console.ts:127-215)

### Requirement: Enterprise (Teams) Tier
The system SHALL deploy a separate self-hostable Teams app on the short domain backed by a pluggable object-storage adapter selected at runtime by CYBERSTRIKE_STORAGE_ADAPTER.

#### Scenario: Teams app on short domain
- **WHEN** CYBERSTRIKE_ENTERPRISE deploy runs
- **THEN** a SolidStart app "Teams" is deployed at shortDomain (cybrstk.us) built with `bun run build:cloudflare`, wired to an R2 storage bucket via CYBERSTRIKE_STORAGE_* env (infra/enterprise.ts:6-16)

#### Scenario: R2 vs S3 storage adapter
- **WHEN** the enterprise app resolves its storage adapter
- **THEN** CYBERSTRIKE_STORAGE_ADAPTER selects r2 (Cloudflare R2 endpoint) or s3 (AWS S3 endpoint) via aws4fetch, and throws "No storage adapter configured" if unset (packages/enterprise/src/core/storage.ts:90-95)

#### Scenario: Node vs Cloudflare build target
- **WHEN** vite build runs with CYBERSTRIKE_DEPLOYMENT_TARGET=cloudflare
- **THEN** it uses the nitro cloudflare_module preset with nodeCompat; otherwise it falls back to the default (node) nitro preset (packages/enterprise/vite.config.ts:6-18) — so the Teams app can also be self-hosted as a plain Node server

### Requirement: Bun Single-File Platform Binary Build
The system SHALL compile the CLI into per-platform single-file Bun binaries across 11 targets and package release archives plus a separate hackbrowser worker bundle.

#### Scenario: Eleven platform targets
- **WHEN** the full build runs (packages/cyberstrike/script/build.ts)
- **THEN** it Bun.build-compiles linux/darwin/win32 x an arch/abi/avx2 matrix yielding 11 targets incl. -baseline (non-AVX2) and -musl variants (build.ts:58-115,166-199)

#### Scenario: Single native binary for local/nix
- **WHEN** invoked with --single (e.g. the nix derivation)
- **THEN** only the current-platform target is built (build.ts:117-136); nix runs `bun --bun ./script/build.ts --single --skip-install` with CYBERSTRIKE_CHANNEL=local and offline models (nix/cyberstrike.nix:36-48)

#### Scenario: Release archives to GitHub
- **WHEN** Script.release is true
- **THEN** linux targets are tar.gz'd and others zip'd, then `gh release upload v<version>` clobbers them onto the release (build.ts:251-260)

#### Scenario: Hackbrowser worker built separately
- **WHEN** any build runs
- **THEN** hackbrowser-worker.js is bundled as a node target with playwright/chromium-bidi/electron external and copied into every platform bin/ (build.ts:228-249), keeping playwright out of the main binary

### Requirement: Multi-Channel Distribution
The system SHALL publish the CLI to npm as a scoped package with per-platform optionalDependencies while consuming external Homebrew, Scoop, Chocolatey and curl channels for install and upgrade.

#### Scenario: Scoped npm package with platform deps
- **WHEN** the npm publish script runs
- **THEN** it publishes @cyberstrike-io/cyberstrike with a postinstall and optionalDependencies mapping each platform sub-package @cyberstrike-io/cyberstrike-<os>-<arch>, each also published with `npm publish --tag <channel>` (packages/cyberstrike/script/publish.ts:67-110)

#### Scenario: Postinstall resolves the platform binary
- **WHEN** the npm package is installed
- **THEN** postinstall.mjs resolves @cyberstrike-io/cyberstrike-<platform>-<arch>, symlinks the binary, and installs the web UI, skills and hackbrowser worker into ~/.local/share/cyberstrike (postinstall.mjs:51-70,268-289)

#### Scenario: curl installer pulls from GitHub Releases
- **WHEN** a user runs the install script
- **THEN** it detects os/arch (incl. rosetta, musl, avx2-baseline), then downloads the matching archive from github.com/CyberStrikeus/CyberStrike/releases (install:102-217)

#### Scenario: Homebrew/Scoop/Choco are external, consume-only
- **WHEN** the CLI self-updates or checks latest
- **THEN** it shells out to brew/scoop/choco and queries formulae.brew.sh, ScoopInstaller/Main and community.chocolatey.org (installation/index.ts:196-260) — this repo contains no code that publishes those manifests, so a fork must maintain the CyberStrikeus/tap and buckets itself

### Requirement: CLI Self-Update Channel Detection
The system SHALL detect how the CLI was installed and dispatch the correct upgrade command per channel, keyed off the compiled-in CYBERSTRIKE_CHANNEL/VERSION.

#### Scenario: Install method detection
- **WHEN** the CLI checks its install method
- **THEN** paths under .cyberstrike/bin or .local/bin => curl, else it probes npm/yarn/pnpm/bun/brew/scoop/choco package lists (installation/index.ts:60-114)

#### Scenario: Channel-aware upgrade dispatch
- **WHEN** upgrade(method,target) is called
- **THEN** it runs the matching command e.g. `curl -fsSL https://cyberstrike.io/install | bash`, `npm i -g @cyberstrike-io/cyberstrike@<target>`, or `brew upgrade CyberStrikeus/tap/cyberstrike` (installation/index.ts:131-190)

#### Scenario: Version channel baked at build time
- **WHEN** the binary reports version/channel
- **THEN** CYBERSTRIKE_VERSION/CYBERSTRIKE_CHANNEL are compile-time defines (build.ts:191-197); npm previews publish as 0.0.0-<channel>-<timestamp> and only channel==latest is non-preview (packages/script/src/index.ts:22-45)

### Requirement: Windows Signing & Disabled Provenance
The system SHALL sign only the Windows CLI via SignPath in a workflow separate from publishing, and SHALL explicitly disable npm provenance during publish.

#### Scenario: SignPath is out-of-band
- **WHEN** the sign-cli workflow runs
- **THEN** it triggers only on push to branch brendan/desktop-signpath or manual dispatch, builds, and submits cyberstrike.exe to SignPath (.github/workflows/sign-cli.yml:5-47) — it is NOT chained from publish.yml, so released binaries are effectively unsigned

#### Scenario: npm provenance disabled
- **WHEN** the publish job runs
- **THEN** NPM_CONFIG_PROVENANCE is set to false (.github/workflows/publish.yml:153) even though the workflow grants id-token: write (line 26), so no npm provenance attestation is produced

#### Scenario: SignPath runner policy
- **WHEN** a SignPath signing request is evaluated
- **THEN** only GitHub Actions and a named Blacksmith runner group are allowed (.signpath/policies/cyberstrike/test-signing.yml:1-6)

### Requirement: Self-Host Paths & Fork Identifiers
The system SHALL support a minimal backend-less self-host of the web UI and CLI, while the full cloud tier depends on hardcoded organization/domain/repo identifiers that a self-hoster must fork and replace under AGPL open-core terms.

#### Scenario: Minimal self-host
- **WHEN** a user wants a private deployment without the SaaS backend
- **THEN** they install the CLI (npm/curl, which includes free models) and optionally serve the static packages/app/dist from their own domain — no backend or data storage (README.md:276)

#### Scenario: Full cloud tier requires forking identifiers
- **WHEN** someone tries to self-host the console/enterprise tiers
- **THEN** they must replace hardcoded anomalyco (PlanetScale org, console.ts:10), cyberstrike.io/cybrstk.us + zoneID (stage.ts), and the CyberStrikeus/CyberStrike GitHub repo used by the installer and updater (install:193-206, installation/index.ts:254) — none of which are parameterized

#### Scenario: AGPL open-core positioning
- **WHEN** license terms are consulted
- **THEN** the repo is AGPL-3.0-only (package.json, LICENSE is GNU AGPL) with commercial licensing offered via contact@cyberstrike.io (README.md:404-406)

## Notes
- Fork provenance: this is sst/opencode rebranded. Every 'opencode' identifier was string-swapped to 'cyberstrike' (@cyberstrike-io npm scope, CYBERSTRIKE_* env vars, cyberstrike.io domain, CyberStrikeus GitHub org). The SST/infra topology (Cloudflare home, three tiers, Zen/Black billing, PlanetScale anomalyco org) is inherited near-verbatim from opencode's zen backend.
- License inconsistency (real bug): root package.json + enterprise + LICENSE are AGPL-3.0-only, but nix/cyberstrike.nix:93 still declares meta.license = lib.licenses.mit (leftover from opencode which is MIT). A packager consuming the flake would mislabel the license.
- 'anomalyco' is a leftover upstream org name, not CyberStrike's own — the PlanetScale org in console.ts:10 is hardcoded to `organization: "anomalyco"`. Self-hosting the console tier is impossible without editing this and the cyberstrike.io/cybrstk.us domains + zoneID in stage.ts.
- Homebrew/Scoop/Chocolatey are consume-only in this repo: the CLI queries and upgrades via them (installation/index.ts) and the README advertises `brew install CyberStrikeus/tap/cyberstrike` / `scoop install cyberstrike`, but there is NO code, workflow, or manifest here that publishes a tap or bucket. The scoop path even points at the community ScoopInstaller/Main bucket (raw.githubusercontent.com/.../bucket/cyberstrike.json) which would not exist for this fork — aspirational/broken unless separately maintained.
- SignPath signing is not part of the release path. sign-cli.yml only triggers on push to a stale dev branch `brendan/desktop-signpath` (another upstream-author artifact) or manual dispatch, and signs only the Windows exe. publish.yml never calls it, so npm/GitHub-release binaries ship unsigned. Combined with NPM_CONFIG_PROVENANCE=false, the published artifacts carry no signature or provenance attestation despite id-token:write being granted.
- curl endpoint naming mismatch: README/install advertise https://cyberstrike.io/install.sh, but the CLI self-updater and the file at repo root use `/install` (no .sh). Both presumably served by the cyberstrike.io site; the repo file is named `install`.
- Build target count is 11 (build.ts allTargets), producing binaries for linux(arm64/x64/x64-baseline/arm64-musl/x64-musl/x64-musl-baseline), darwin(arm64/x64/x64-baseline), win32(x64/x64-baseline). Windows has no arm64 build.
- The main binary deliberately excludes playwright (subprocess.md design): hackbrowser runs in a spawned node worker (hackbrowser-worker.js) with playwright resolved at runtime from ~/.local/share/cyberstrike/node_modules, installed by postinstall (pinned 1.58.2, chromium via `npx playwright install` as a separate one-time step).
- Nix reproducibility relies on a fixed-output node_modules derivation with per-system sha256 in nix/hashes.json; the flake exposes node_modules_updater (fakeHash) to surface the correct hash on build failure, and a nix-hashes workflow presumably updates them.
- README marketing claims (176+ MCP tools, 13+ agents, compliance frameworks) are product-surface claims not verifiable from the deployment code and were out of scope for this capability; the deployment/infra numbers cited above are all read directly from source.
- Provider gating is defense-in-depth: both the Pulumi provider registration (sst.config.ts:11-12) and the tier import (18-19) key off the same env vars, so a partial secret set (e.g. Stripe without PlanetScale) deploys only the app tier and registers only the stripe provider.

## Key Files
- `sst.config.ts` — Root SST program: Cloudflare home, conditional Stripe/PlanetScale providers, and the three-tier import gating (app always, console on secrets, enterprise on flag)
- `infra/app.ts` — Always-on app tier: Api Worker (api.<domain>), SyncServer durable object, R2 bucket, docs Astro site, WebApp static site
- `infra/console.ts` — SaaS/console tier: PlanetScale (anomalyco/cyberstrike), OpenAuth worker, Stripe CyberStrike Black billing, 20 Zen model secrets, Console SolidStart app
- `infra/enterprise.ts` — Enterprise Teams tier: SolidStart app on cybrstk.us with R2 storage env wiring
- `infra/stage.ts` — Stage->domain resolution and hardcoded Cloudflare zoneID (cyberstrike.io / cybrstk.us)
- `infra/secret.ts` — R2 access/secret key secret Linkables for enterprise storage
- `install` — curl installer: OS/arch/musl/avx2 detection, downloads release archives from CyberStrikeus/CyberStrike GitHub Releases
- `packages/cyberstrike/script/build.ts` — Bun single-file compile across 11 platform targets, release archive creation, separate hackbrowser-worker bundle
- `packages/cyberstrike/script/publish.ts` — npm publish: scoped main package + per-platform optionalDependencies, channel-tagged publish
- `packages/cyberstrike/script/postinstall.mjs` — npm postinstall: platform binary resolution/symlink, web UI + skills + hackbrowser/playwright install
- `packages/cyberstrike/src/installation/index.ts` — CLI self-update: install-method detection and per-channel (npm/brew/scoop/choco/curl) upgrade + latest-version dispatch
- `packages/script/src/index.ts` — Version/channel computation (Script module) driving preview vs latest publishing
- `.github/workflows/publish.yml` — Release pipeline: version->build-cli->build-app->publish; NPM_CONFIG_PROVENANCE=false
- `.github/workflows/sign-cli.yml` — Out-of-band SignPath Windows-exe signing (branch/dispatch triggered, not wired into publish)
- `.github/workflows/deploy.yml` — SST deploy on dev/production supplying Cloudflare+PlanetScale+Stripe secrets
- `flake.nix` — Nix flake: devShell + cyberstrike/node_modules packages via overlay
- `nix/cyberstrike.nix` — Nix build derivation: --single local binary build with offline models; meta.license mismatched to MIT
- `nix/node_modules.nix` — Fixed-output node_modules derivation keyed on nix/hashes.json per system
- `packages/enterprise/vite.config.ts` — Enterprise build target switch: nitro cloudflare_module preset vs default node preset
- `packages/enterprise/src/core/storage.ts` — Runtime R2/S3 storage-adapter selection for the self-hostable Teams app
