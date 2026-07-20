# Skill Knowledge Substrate

## Purpose
The skill system is CyberStrike's context-efficient knowledge base: a corpus of Markdown "skills" (each a SKILL.md with YAML frontmatter) discovered from many directories, indexed into inverted indexes, and lazily loaded into the agent's context on demand via the `skill` tool. It layers three concerns on plain Markdown: Ed25519 content signing (four verification tiers), a kill-chain graph (chains_with / severity_boost) that turns individual findings into escalation paths, and per-skill token accounting so only what is needed occupies context. It exists so a pentest agent can search thousands of playbooks without paying the token cost of loading them all.

## Requirements

### Requirement: Multi-Source Skill Discovery
The system SHALL discover skills by recursively scanning SKILL.md files from external agent dirs, .cyberstrike dirs, built-in and installed dirs, config-declared paths, and remote URLs, loading global before project so later sources overwrite earlier ones by name.

#### Scenario: Layered scan order
- **WHEN** state() initializes
- **THEN** it scans `.claude`/`.agents` skills/ globs under home (global) then walks up from Instance.directory (project), then `{skill,skills}/**/SKILL.md` under Config.directories(), the built-in `.cyberstrike` dir resolved relative to source, the XDG installed dir, config `skills.paths`, then `skills.urls` — packages/cyberstrike/src/skill/skill.ts:147-259

#### Scenario: Duplicate name overwrite
- **WHEN** two SKILL.md files declare the same frontmatter name
- **THEN** the later-scanned one overwrites the earlier in the `skills` record and only a debug log is emitted (skill.ts:90-96, 122); project-scope thus overrides global

#### Scenario: External skills can be disabled
- **WHEN** Flag.CYBERSTRIKE_DISABLE_EXTERNAL_SKILLS is set
- **THEN** the `.claude`/`.agents` scans are skipped entirely (skill.ts:165-179)

#### Scenario: Remote URL skills
- **WHEN** config.skills.urls contains a base URL
- **THEN** Discovery.pull fetches `index.json`, downloads each entry (inline files, single skill-md, or tar.gz archive extracted via `tar xzf`) into the cache dir, and returns dirs that contain a SKILL.md to be scanned (packages/cyberstrike/src/skill/discovery.ts:39-122)

### Requirement: Frontmatter Schema And Parsing
The system SHALL require only `name` and `description` in frontmatter, treat all signing, categorization, tech-stack and kill-chain fields as optional, publish a Session.Event.Error and log when frontmatter fails to parse, and silently skip files that merely lack the required pair.

#### Scenario: Minimal valid skill
- **WHEN** a SKILL.md has just name and description
- **THEN** Info.pick({name,description}).safeParse succeeds and the skill is registered with all optional fields undefined (skill.ts:86-87, 122-144)

#### Scenario: Malformed frontmatter
- **WHEN** ConfigMarkdown.parse throws a FrontmatterError
- **THEN** a Session.Event.Error is published, an error is logged, and addSkill returns without registering the skill (skill.ts:75-84)

#### Scenario: Optional metadata coerced
- **WHEN** frontmatter carries tags/tech_stack/cwe_ids/chains_with/prerequisites as arrays and severity_boost as a map
- **THEN** arrays are filtered to strings via toStringArray and severity_boost is accepted only if a non-array object, else undefined (skill.ts:119-143)

### Requirement: Ed25519 Signature Verification Tiers
The system SHALL compute a frontmatter-stripped SHA-256 over each skill and assign one of four tiers — official, community, unverified, tampered — verifying an Ed25519 signature only when signed_by equals 'cyberstrike-official'.

#### Scenario: No hash means unverified
- **WHEN** a skill has no sha256 field
- **THEN** verify returns "unverified" immediately (packages/cyberstrike/src/skill/signing.ts:30)

#### Scenario: Hash mismatch means tampered
- **WHEN** the recomputed hash (content with sha256/signature/signed_by lines stripped and trimmed) differs from the declared sha256
- **THEN** verify returns "tampered" (signing.ts:32-36, 13-22)

#### Scenario: Valid hash but not official signer
- **WHEN** the hash matches but signed_by is not 'cyberstrike-official' or signature is missing
- **THEN** verify returns "community" (signing.ts:38)

#### Scenario: Official signature check
- **WHEN** hash matches, signed_by is cyberstrike-official, and the Ed25519 signature over the sha256 string verifies against the embedded public key
- **THEN** verify returns "official", else "tampered" (including any thrown error) (signing.ts:40-50)

### Requirement: Tampered Blocked, All Other Tiers Load
The system SHALL refuse to register only skills whose verification is 'tampered', loading unverified, community, and official skills identically; the verified tier SHALL be display-only and gate nothing at load time.

#### Scenario: Tampered dropped at scan
- **WHEN** SkillSigning.verify returns 'tampered' during addSkill
- **THEN** a warning is logged, a Session.Event.Error is published, and the skill is NOT added to the registry (skill.ts:109-117)

#### Scenario: Unverified loads normally
- **WHEN** an agent issues action=load on a skill whose verified tier is 'unverified'
- **THEN** the load path calls Skill.get, ctx.ask, and SkillContext.load with no tier check — tier only appears in the `verified="..."` output attribute (packages/cyberstrike/src/tool/skill.ts:202-252)

#### Scenario: Tier is cosmetic in list/search
- **WHEN** list or search renders results
- **THEN** each row prints `[${verified ?? "unverified"}]` purely for display with no filtering by tier (tool/skill.ts:106, 168)

### Requirement: Skill Tool Actions
The system SHALL expose a single `skill` tool with actions load/unload/search/chain/suggest/list, defaulting to load, filtering the advertised set per agent permission, and requiring an interactive permission grant before loading.

#### Scenario: Load registers content and appends chain links
- **WHEN** action=load with a valid name
- **THEN** it awaits ctx.ask({permission:"skill",patterns:[name]}), calls SkillContext.load, samples up to 10 sibling files via Ripgrep, and returns the trimmed skill content plus `<kill_chain_links>` from SkillIndex.chainsFrom (tool/skill.ts:202-268)

#### Scenario: Search dispatch precedence
- **WHEN** action=search is given cwe, tech, category, or query
- **THEN** it selects byCWE, byTechStack, byCategory, or search in that priority order (else lists all), caps at 50, and reports a truncated header when totalCount exceeds returned (tool/skill.ts:127-172)

#### Scenario: Chain and suggest require findings
- **WHEN** action=chain or suggest is called without findings
- **THEN** it throws "Findings required..."; otherwise chain returns KillChain.summary and suggest returns SkillContext.suggest output (tool/skill.ts:175-199)

#### Scenario: Per-agent accessibility cache
- **WHEN** the tool executes for an agent with a permission block
- **THEN** skills are filtered to those where PermissionNext.evaluate("skill", name, permission) is not "deny" and cached per agent — used for list/search display and load error messages (tool/skill.ts:62-76, 206)

### Requirement: SkillIndex Inverted Indexes And Scoring
The system SHALL build in-memory inverted indexes keyed by tag, lowercased tech, uppercased CWE, and lowercased category, plus a weighted keyword search, rebuilt from Skill.all() on demand.

#### Scenario: Index build
- **WHEN** ensureBuilt runs the first time (or rebuild is called)
- **THEN** it populates entries plus tagIndex/techIndex/cweIndex/categoryIndex from every skill and logs the count (packages/cyberstrike/src/skill/index-engine.ts:44-87)

#### Scenario: Weighted search scoring
- **WHEN** search(query) runs
- **THEN** it scores exact name match +100, prefix +50, substring +20, exact tag +40, tag substring +15, owasp_id +30, category +10, description +5, then sorts desc and slices to limit (index-engine.ts:97-116)

#### Scenario: Faceted lookups normalize case
- **WHEN** byTechStack/byCWE/byCategory/byTag is queried
- **THEN** the key is lowercased (tech/category/tag) or uppercased (CWE) to match how the index was built (index-engine.ts:49-63, 118-155)

### Requirement: Kill Chain Analysis
The system SHALL derive attack chains by pairing findings whose skill_id lists the other in chains_with, computing combined severity from a severity_boost annotation when present and otherwise the max of the two severities.

#### Scenario: Chain pairing dedup
- **WHEN** analyze receives findings where one chains_with another that is also present
- **THEN** it emits one chain per unordered pair (deduped via sorted key), sorted by combined severity descending (packages/cyberstrike/src/skill/killchain.ts:33-66)

#### Scenario: Severity boost parsing
- **WHEN** a severity_boost value ends in a parenthesized level like "...(Critical)"
- **THEN** combined_severity is that lowercased token, else it falls back to maxSeverity(finding, partner) (killchain.ts:49-52)

#### Scenario: Next-step suggestion
- **WHEN** SkillContext.suggest is given a finding
- **THEN** chains_with targets not already active/suggested are added at priority high (if a boost note exists) else medium, and tech-stack matches at priority low, sorted high→low (packages/cyberstrike/src/skill/context.ts:63-100)

### Requirement: Context Token Accounting
The system SHALL track loaded skills in a module-level map, estimate tokens as ceil(content.length / 4), deduplicate repeat loads, and support unload/clear/tokenCount introspection.

#### Scenario: Token estimate and dedup
- **WHEN** load(name) is called
- **THEN** if already loaded it returns the cached content without re-counting; otherwise it stores {content, ceil(len/4)} and logs the token count (context.ts:10-27)

#### Scenario: Unload frees tokens
- **WHEN** unload(name) is called for a loaded skill
- **THEN** the entry is deleted and true returned; tokenCount() sums remaining entries' tokens (context.ts:29-46)

#### Scenario: List loaded
- **WHEN** action=list with loaded=true
- **THEN** it reports SkillContext.active() names and SkillContext.tokenCount() as "N skills loaded (T tokens)" (tool/skill.ts:78-97)

## Notes
- Corpus sizing (measured, `find .cyberstrike -name SKILL.md`): 7656 SKILL.md files. Breakdown: CIS_benchmarks 5000, NIST 1606, mitre_attack 691, WEB 125, mitre_attack_mobile 124, mitre_attack_ics 83, plus ~30 hand-written custom skills (attack-*, *-postexploit, ad-security, kerberos-attacks, etc.). The tool description's 'Thousands of skills available' (tool/skill.ts:29) is accurate, not inflated.
- MAJOR: the entire signing apparatus is dormant on the shipped corpus. `grep -rl '^sha256:' / '^signature:' / 'signed_by: cyberstrike-official'` over .cyberstrike returns 0, 0, 0. Every shipped skill therefore verifies as 'unverified' and loads. `author: cyberstrike-official` appears in frontmatter but is NOT a signature (author is not consulted by verify()). So in practice tiers official/community/tampered never occur from the built-in corpus — only unverified.
- Confirmed the task's premise: unverified LOADS (no gate — tool/skill.ts:202-217 never inspects `verified`), and only 'tampered' is BLOCKED, and that block happens at scan time in the loader (skill.ts:109-117), not in the tool. The tool would happily load a tampered skill if one were in the registry, but the loader never puts it there.
- `skills.disabled` is defined in config (config.ts:756, described as skills that "will not be loaded by agents") but is NOT enforced anywhere in the codebase. Its only references are the schema and the enable/disable list-management endpoints (packages/cyberstrike/src/server/routes/skill.ts:338-343, 371-376), which merely read and rewrite the config array — no loader, `skill` tool, or List/Search/Get route ever filters by it. An agent using the `skill` tool can therefore load a skill listed in skills.disabled.
- Load-time permission enforcement is via ctx.ask (tool/skill.ts:210-215), NOT via the accessibleCache. The accessibleCache (PermissionNext.evaluate filter, tool/skill.ts:62-76) only shapes what list/search advertise and the 'Available:' error string; Skill.get at line 204 can resolve any registered skill by exact name regardless of that filter.
- SkillContext.load(params.name) at tool/skill.ts:217 is async but not awaited (fire-and-forget); the returned output uses skill.content from Skill.get directly, so display does not depend on the load completing. Minor race, no functional impact.
- severity_boost combined-severity extraction (killchain.ts:51) only fires when the boost string ends in a parenthesized word, e.g. '...(Critical)'. The shipped attack-ssrf severity_boost values ('SSRF from SSTI = full RCE chain') lack that suffix, so they fall through to maxSeverity — the parenthesis convention is followed inconsistently in the actual corpus.
- Token accounting is a crude heuristic: estimateTokens = ceil(content.length/4) (context.ts:10-12), a character-count proxy, not a real tokenizer. The 'tokens' shown to the agent are approximate.

## Key Files
- `packages/cyberstrike/src/skill/skill.ts` — Skill loader: frontmatter schema (Info), multi-source discovery/scan order, duplicate handling, tamper-drop at scan, dirsOnly fast path
- `packages/cyberstrike/src/skill/signing.ts` — Ed25519 signing: computeHash (frontmatter-stripped SHA-256), verify() four-tier logic, sign/generateKeyPair, embedded official public key
- `packages/cyberstrike/src/skill/index-engine.ts` — SkillIndex: inverted indexes (tag/tech/cwe/category), weighted keyword search, chainsFrom/prerequisitesFor
- `packages/cyberstrike/src/skill/killchain.ts` — KillChain: pairs findings via chains_with, severity_boost parsing, combined-severity ordering, summary rendering
- `packages/cyberstrike/src/skill/context.ts` — SkillContext: loaded map, token estimation (len/4), load/unload/tokenCount, suggest() next-step ranking
- `packages/cyberstrike/src/skill/discovery.ts` — Remote skill pull: fetch index.json, download inline/skill-md/tar.gz archives into cache dir
- `packages/cyberstrike/src/tool/skill.ts` — The `skill` tool: load/unload/search/chain/suggest/list actions, per-agent permission filter, ctx.ask gate, output rendering
- `.cyberstrike/skill/SKILL_GUIDE.md` — Authoring guide: documents frontmatter fields, categories, content templates, naming conventions
- `packages/cyberstrike/src/config/config.ts` — Config.Skills schema (paths/urls/disabled) at lines 750-758
