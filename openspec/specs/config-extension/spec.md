# Config Loading and User Extension Discovery

## Purpose
Defines how CyberStrike assembles its runtime configuration by merging up to eight layered sources (built-in defaults, remote well-known, global XDG config, custom-path config, project config, .cyberstrike directories, inline env content, and managed enterprise config) and how it discovers user-supplied extensions (agents, commands, modes, plugins, custom tools, skills) by scanning fixed globs across those roots. It exists so a user can extend the tool — add agents, slash commands, MCP servers, providers, plugins, tools, and skills — by dropping files into well-known directories or editing cyberstrike.json{,c} WITHOUT forking the codebase. Config is parsed as JSONC with {env:VAR}/{file:path} interpolation and a strict Zod schema.

## Requirements

### Requirement: Layered Precedence Chain
The system SHALL merge configuration from lowest to highest precedence in the fixed order built-in defaults < remote well-known < global XDG config < CYBERSTRIKE_CONFIG < project cyberstrike.json{,c} < .cyberstrike directories < CYBERSTRIKE_CONFIG_CONTENT < managed enterprise dir, so that later sources win on scalar keys while plugin/instructions arrays are concatenated.

#### Scenario: project overrides global
- **WHEN** the same scalar key (e.g. model) is set in both global ~/.config/cyberstrike and a project cyberstrike.json
- **THEN** the project value wins because global() is merged first (config.ts:165) and project config is merged after (config.ts:174-181)

#### Scenario: managed enterprise always wins
- **WHEN** a managed config dir (macOS /Library/Application Support/cyberstrike, Linux /etc/cyberstrike, Windows %ProgramData%/cyberstrike) exists and contains cyberstrike.json{,c}
- **THEN** it is loaded LAST and overrides every other source including CYBERSTRIKE_CONFIG_CONTENT (config.ts:43-54, 248-256)

#### Scenario: inline content overrides non-managed
- **WHEN** CYBERSTRIKE_CONFIG_CONTENT env var holds a JSON string
- **THEN** it is parsed with plain JSON.parse (not JSONC) and merged above all file/dir sources but below managed (config.ts:243-246)

#### Scenario: array fields concatenate not replace
- **WHEN** both a lower and higher source define plugin or instructions arrays
- **THEN** the custom merge() unions them into a de-duplicated array instead of replacing (config.ts:57-66)

### Requirement: Project Config Discovery (findUp)
The system SHALL discover project configuration by walking up from Instance.directory to Instance.worktree looking for cyberstrike.jsonc then cyberstrike.json, merging farther-up files first so the nearest file wins and cyberstrike.json wins over cyberstrike.jsonc at the same level, and SHALL skip this entirely when CYBERSTRIKE_DISABLE_PROJECT_CONFIG is truthy.

#### Scenario: nearest ancestor wins
- **WHEN** cyberstrike.json exists in both the worktree root and a nested subdirectory that is the current dir
- **THEN** Filesystem.findUp collects both and found.toReversed() merges the farthest first so the nearest overrides (config.ts:175-180, filesystem.ts:39-51)

#### Scenario: json beats jsonc
- **WHEN** both cyberstrike.jsonc and cyberstrike.json sit at the same directory
- **THEN** the outer loop processes .jsonc first then .json, so .json is merged last and wins (config.ts:175)

#### Scenario: project discovery disabled
- **WHEN** CYBERSTRIKE_DISABLE_PROJECT_CONFIG is 'true' or '1'
- **THEN** both project cyberstrike.json{,c} lookup and project .cyberstrike directory scanning are skipped (config.ts:174,190; flag.ts:72-78)

### Requirement: Config.directories() Scan Roots
The system SHALL assemble the set of extension scan roots as: the global XDG config dir, every .cyberstrike directory found walking up from Instance.directory to worktree (unless project config is disabled), the home ~/.cyberstrike, and CYBERSTRIKE_CONFIG_DIR when set; it loads cyberstrike.json{,c} only from roots ending in .cyberstrike (or the explicit CONFIG_DIR), and exposes the de-duplicated list via Config.directories().

#### Scenario: dot-cyberstrike json loaded
- **WHEN** a scan root path ends with '.cyberstrike'
- **THEN** cyberstrike.jsonc and cyberstrike.json inside it are loaded and merged into the result (config.ts:218-227)

#### Scenario: explicit config dir highest
- **WHEN** CYBERSTRIKE_CONFIG_DIR env var points at a directory
- **THEN** it is pushed last onto the directories array, giving it highest precedence among scanned dirs, and its cyberstrike.json{,c} is loaded (config.ts:210-221)

#### Scenario: home directory always scanned
- **WHEN** no project config and no env overrides are present
- **THEN** Global.Path.config (XDG) and ~/.cyberstrike are still scanned as roots (config.ts:187-207)

#### Scenario: roots exposed to subsystems
- **WHEN** the skill and tool registries initialize
- **THEN** they call Config.directories() to reuse the same root list for their own globs (config.ts:1568-1570; skill.ts:182; tool/registry.ts:69)

### Requirement: Glob-Based Extension Discovery
The system SHALL discover drop-in extensions in every scan root via fixed globs — commands at {command,commands}/**/*.md, agents at {agent,agents}/**/*.md, modes at {mode,modes}/*.md, plugins at {plugin,plugins}/*.{ts,js}, and custom tools at {tool,tools}/*.{js,ts} — deriving each item's name from its relative path and registering it without any code fork.

#### Scenario: markdown command registered
- **WHEN** a file command/deploy.md or commands/deploy.md exists in a scan root
- **THEN** it is parsed and registered as command 'deploy' with its body as the template (config.ts:409-445)

#### Scenario: custom tool auto-loaded
- **WHEN** a file tool/recon.ts (or tools/recon.ts) exports functions
- **THEN** each export becomes a tool named recon (default export) or recon_<exportId>, after dependencies are installed (tool/registry.ts:65-90)

#### Scenario: local plugin file discovered
- **WHEN** a file plugin/audit.ts exists in a scan root
- **THEN** it is added as a file:// plugin specifier via pathToFileURL and later de-duplicated by canonical name (config.ts:525-538, 572-590)

#### Scenario: agent name from path
- **WHEN** an agent markdown lives under agents/ or .cyberstrike/agent/
- **THEN** the rel()/trim() helpers strip the directory prefix and extension to compute the agent name (config.ts:448-486)

### Requirement: Skill Discovery Roots
The system SHALL discover skills from Claude-Code-compatible external directories (.claude/skills, .agents/skills at home and up-tree), from .cyberstrike/{skill,skills}/**/SKILL.md across Config.directories(), from the built-in and XDG-installed skill directories, and from config.skills.paths and config.skills.urls, honoring CYBERSTRIKE_DISABLE_EXTERNAL_SKILLS and config.skills.disabled.

#### Scenario: claude code skills reused
- **WHEN** a .claude/skills/foo/SKILL.md exists in the home dir or any ancestor up to the worktree
- **THEN** it is scanned and loaded unless CYBERSTRIKE_DISABLE_EXTERNAL_SKILLS (or the broader CLAUDE_CODE disables) is set (skill.ts:62-179; flag.ts:24-27)

#### Scenario: extra skill paths from config
- **WHEN** config.skills.paths lists a directory (supports ~/ and relative-to-project expansion)
- **THEN** it is resolved and recursively scanned for **/SKILL.md (skill.ts:228-243; schema config.ts:750-757)

#### Scenario: remote skill urls
- **WHEN** config.skills.urls lists a URL such as https://example.com/.well-known/skills/
- **THEN** Discovery.pull downloads them and each resulting dir is scanned for SKILL.md (skill.ts:246-259)

### Requirement: Interpolation of env and file references
The system SHALL, before JSONC parsing, replace every {env:VAR} token with process.env[VAR] (empty string when unset) and every {file:path} token (on non-commented lines) with the JSON-escaped contents of that file, resolving ~/ to home and relative paths against the config file's directory.

#### Scenario: env var substituted
- **WHEN** a config value is "{env:OPENAI_API_KEY}"
- **THEN** it is replaced with the env var's value, or an empty string if the variable is unset (config.ts:1332-1334)

#### Scenario: file reference inlined
- **WHEN** a config value contains {file:./secret.txt} on an uncommented line
- **THEN** the referenced file (relative to the config dir, ~/ expanded) is read, JSON-escaped, and inlined; a missing file raises a ConfigInvalidError (config.ts:1336-1370)

#### Scenario: commented file ref skipped
- **WHEN** a {file:...} token appears on a line whose trimmed text starts with //
- **THEN** the interpolation is skipped for that line (config.ts:1342-1345)

### Requirement: Markdown Frontmatter Parsing with CC Fallback
The system SHALL parse agent/command/mode/skill markdown frontmatter with gray-matter, and on YAML failure retry once with a sanitizer that tolerates Claude-Code-style invalid YAML (unquoted colon-bearing scalars converted to block scalars), throwing ConfigFrontmatterError only if both attempts fail.

#### Scenario: valid yaml parsed directly
- **WHEN** a SKILL.md/command.md has well-formed YAML frontmatter
- **THEN** matter(template) returns data and content on the first try (markdown.ts:70-75)

#### Scenario: cc-invalid yaml recovered
- **WHEN** a scalar frontmatter value contains an unquoted colon (e.g. description: use when: X)
- **THEN** fallbackSanitization rewrites it as a block scalar (key: |-) and matter re-parses successfully (markdown.ts:41-68, 77-78)

#### Scenario: unrecoverable frontmatter
- **WHEN** both the direct and sanitized parses throw
- **THEN** a ConfigFrontmatterError carrying the file path and message is raised, and callers publish a Session error and skip the file (markdown.ts:79-88; config.ts:418-427)

### Requirement: Strict Schema Surface and Migrations
The system SHALL validate each config against a strict Zod Info schema whose top-level keys (agent, command, mcp, bolt, plugin, provider, skills, permission, share, enterprise, instructions, server, lsp, formatter, model, and others) define exactly what a user may add; it rejects unknown top-level keys, auto-injects and writes back $schema, resolves plugin specifiers, and migrates legacy fields (tools->permission, autoshare->share, mode->agent, maxSteps->steps).

#### Scenario: user adds mcp server
- **WHEN** a user defines mcp.myserver with type local/remote in cyberstrike.json
- **THEN** it merges over the 11 built-in security MCP defaults (all enabled:false) that were seeded at lowest precedence (schema config.ts:1174-1187; defaults 82-141)

#### Scenario: unknown key rejected
- **WHEN** a config contains a misspelled or unsupported top-level key
- **THEN** Info.strict() fails and a ConfigInvalidError with the Zod issues is thrown (config.ts:1282, 1417-1421)

#### Scenario: schema auto-written back
- **WHEN** a validated config file omits $schema and is writable
- **THEN** $schema is set to https://cyberstrike.io/config.json and written back into the original text preserving {env:} tokens (config.ts:1399-1404)

#### Scenario: legacy fields migrated
- **WHEN** a config uses the deprecated tools map, autoshare:true, mode.*, or an agent maxSteps
- **THEN** they are converted to permission entries, share:'auto', agent primary entries, and steps respectively (config.ts:258-295, 823-844)

## Notes
- README INFLATION: README.md:40 and :311 advertise '176+ MCP tools' / '176+ security tools across 5 domains'. config.ts hard-codes exactly 11 built-in security MCP servers (github-security, cve, osint, cloud-audit, hackbrowser, darknet, dns-security, supply-chain, mcp-scanner, steganography, satellite) at config.ts:82-141, and ALL are enabled:false by default. The 176 is a downstream per-server tool count, not verifiable from config; nothing is enabled out of the box, and each server runs via `npx -y <pkg>` only after the user enables it.
- The built-in MCP block is seeded into result.mcp BEFORE remote/global/project merges (config.ts:82), so it is the LOWEST precedence: user config both overrides and enables these servers. This built-in security-MCP block, the Bolt (Docker Kali) schema, and the report_vulnerability permission are CyberStrike additions on top of the upstream opencode config system (this repo is a rename fork opencode->cyberstrike; the loading architecture, findUp/up walkers, gray-matter fallback, and glob discovery are inherited from opencode).
- PROVENANCE nuance: remote .well-known/cyberstrike config is only fetched for Auth entries of type 'wellknown' (config.ts:143-158) — it is NOT fetched for arbitrary project URLs; it requires a configured auth key, and its token is injected into process.env before fetch.
- CYBERSTRIKE_CONFIG_CONTENT (config.ts:244) and CYBERSTRIKE_PERMISSION (config.ts:270) are parsed with plain JSON.parse, so unlike file-based configs they do NOT support comments or trailing commas; file loads use jsonc-parser with allowTrailingComma (config.ts:1374).
- Interpolation supports BOTH {env:VAR} (task-named) AND {file:path} (config.ts:1336-1370) — the latter inlines JSON-escaped file contents and is the recommended way to keep secrets out of the committed config.
- Info schema is .strict() (config.ts:1282): users extend via the known top-level keys plus filesystem drop-ins, NOT via arbitrary keys; unknown keys raise ConfigInvalidError. Agent schema, by contrast, uses .catchall(z.any()) and funnels unknown keys into options (config.ts:795, 817-821).
- WHAT A USER CAN ADD WITHOUT FORKING: (1) edit cyberstrike.json{,c} at project root, ~/.config/cyberstrike, ~/.cyberstrike, or any .cyberstrike dir up-tree — keys mcp/bolt/provider/agent/command/plugin/skills/permission/instructions/server/lsp/formatter/model; (2) drop markdown into {agent,agents}/, {command,commands}/, {mode,modes}/ under any scan root; (3) drop {plugin,plugins}/*.{ts,js} and {tool,tools}/*.{js,ts}; (4) skills via .claude/skills, .agents/skills, .cyberstrike/{skill,skills}, config.skills.paths, or config.skills.urls. Enterprise admins use the managed dir for org-wide overrides that users cannot override.
- Config dirs that are writable get @cyberstrike-io/plugin auto-installed (config.ts:321-349) and a .gitignore written; read-only dirs (e.g. the managed enterprise dir) skip install (needsInstall/isWritable, config.ts:351-394) — this is why the managed config is loaded outside the directories loop (config.ts:249-256).
- config.skills.disabled is declared in the Skills schema (config.ts:756) as skill names not to be loaded by agents; skill.ts loads all discovered skills and the disable list is applied at agent-consumption time (not at discovery in skill.ts state()).

## Key Files
- `packages/cyberstrike/src/config/config.ts` — Core config namespace: precedence chain (state, lines 68-314), built-in MCP defaults (82-141), directories() roots (187-213), command/agent/mode/plugin globs (409-538), plugin dedup (550-590), {env}/{file} interpolation + JSONC load + $schema writeback (1330-1421), full Info Zod schema (1093-1287), global() loader (1289-1316)
- `packages/cyberstrike/src/config/markdown.ts` — ConfigMarkdown: gray-matter frontmatter parse with CC-invalid-YAML fallbackSanitization (block-scalar rewrite) and FrontmatterError; FILE_REGEX/SHELL_REGEX for @file and !`shell` template refs
- `packages/cyberstrike/src/global/index.ts` — Global.Path: XDG-based data/config/cache/state dirs (~/.config/cyberstrike etc.), CYBERSTRIKE_TEST_HOME override, cache versioning
- `packages/cyberstrike/src/util/filesystem.ts` — findUp() and up() up-tree walkers and globUp() used for project config, .cyberstrike dirs, and external skill discovery
- `packages/cyberstrike/src/skill/skill.ts` — Skill discovery roots: EXTERNAL_DIRS (.claude/.agents), .cyberstrike/{skill,skills}, builtin + XDG installed, config.skills.paths/urls; disable flags
- `packages/cyberstrike/src/tool/registry.ts` — Custom tool discovery: {tool,tools}/*.{js,ts} glob over Config.directories(), dynamic import, namespacing, plugin-provided tools
- `packages/cyberstrike/src/flag/flag.ts` — Env flags gating discovery: CYBERSTRIKE_CONFIG, CYBERSTRIKE_CONFIG_DIR, CYBERSTRIKE_CONFIG_CONTENT, CYBERSTRIKE_PERMISSION, CYBERSTRIKE_DISABLE_PROJECT_CONFIG, CYBERSTRIKE_DISABLE_EXTERNAL_SKILLS
