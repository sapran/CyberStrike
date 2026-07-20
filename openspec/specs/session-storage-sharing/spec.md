# Session Persistence & Sharing (SQLite store, snapshots, ShareNext → enterprise viewer)

## Purpose
CyberStrike persists all agent state in a single local SQLite database (Drizzle ORM) with a session/message/part core plus pentest-specific tables, git-based file snapshots for revert, and a one-time JSON→SQLite migration on first run. Layered on top is an optional "share" pipeline (ShareNext) that streams a session — its messages, parts, model list, and full before/after file diffs — to an enterprise viewer web app, defaulting to the off-box host https://cybrstk.us when no self-hosted enterprise.url is configured. The enterprise package stores shared data as JSON objects in S3/R2 and renders a public read-only viewer.

## Requirements

### Requirement: SQLite Session/Message/Part Data Model
The system SHALL persist sessions, messages, and parts as Drizzle SQLite tables where message and part bodies are stored as opaque JSON blobs, with foreign keys cascading deletes from session down to messages, parts, todos, vulnerabilities, requests and web-analysis tables.

#### Scenario: Message stored as JSON blob under session FK
- **WHEN** a message is inserted
- **THEN** it is written to the `message` table with a `data` JSON column holding the full MessageV2.Info minus id/sessionID, keyed by `session_id` referencing session(id) ON DELETE CASCADE (session.sql.ts:37-48); parts likewise carry both `message_id` (FK CASCADE) and a denormalized `session_id` (session.sql.ts:50-62)

#### Scenario: Session deletion cascades
- **WHEN** a session row is deleted
- **THEN** every child row (message, part, todo, vulnerability, request, web_* , coverage_note, endpoint_template, session_share) is removed via `onDelete: cascade` FKs, and PRAGMA foreign_keys is ON so the cascade fires (db.ts:154, session.sql.ts:43/56/69/88)

#### Scenario: Session carries share + summary state inline
- **WHEN** a session is shared or summarized
- **THEN** the `session` row itself stores `share_url`, `summary_additions/deletions/files/diffs` (JSON Snapshot.FileDiff[]) and a `revert` JSON pointer directly on the session table (session.sql.ts:23-28)

#### Scenario: Request dedup uniqueness
- **WHEN** two captured requests hash to the same key in one session
- **THEN** the `request_keyhash_idx` unique index on (session_id, key_hash) makes insert an atomic ON CONFLICT upsert, while NULL key_hash rows (legacy) stay distinct (session.sql.ts:177-181)

### Requirement: Database Initialization, Migrations & Schema Reconciliation
The system SHALL open one lazy SQLite database at Global.Path.data/cyberstrike.db with WAL/foreign-keys pragmas, apply bundled-or-dev Drizzle migrations, then run a reconcile pass that structurally reshapes legacy tables and ADD-COLUMNs any drift against the live schema.

#### Scenario: Single DB opened with pragmas
- **WHEN** the DB client is first used
- **THEN** cyberstrike.db is opened with create:true and PRAGMA journal_mode=WAL, synchronous=NORMAL, busy_timeout=5000, foreign_keys=ON (db.ts:146-157)

#### Scenario: Missing columns auto-added
- **WHEN** a table exists but lacks a column present in the Drizzle schema
- **THEN** reconcile() issues `ALTER TABLE ... ADD COLUMN` for each missing column, compensating for multi-statement migrations that Drizzle's sqlite3_prepare_v2 only partially applied (db.ts:126-142)

#### Scenario: Legacy web_credential reshape
- **WHEN** web_credential still has old type/value columns and no headers column
- **THEN** reconcile() rebuilds the table, converting bearer/jwt/cookie/api_key/basic type+value into a `headers` JSON object via a CASE expression and swapping the table in place (db.ts:92-122)

### Requirement: First-Run JSON-to-SQLite Migration
The system SHALL, only when cyberstrike.db does not yet exist, glob-scan the legacy Global.Path.data/storage JSON tree and bulk-import projects, sessions, messages, parts, todos, permissions and session_shares into SQLite inside one transaction, skipping orphans and continuing past per-file errors.

#### Scenario: Runs once, gated by DB file presence
- **WHEN** cyberstrike.db is absent at startup
- **THEN** index.ts prints a one-time migration notice and calls JsonMigration.run against the freshly created client; if the storage dir is absent it returns zeroed stats (index.ts:85-119, json-migration.ts:24-39)

#### Scenario: Orphan and error tolerance
- **WHEN** a session references a projectID not in the imported set, or a JSON file fails to parse
- **THEN** the row is counted as an orphan and skipped (not inserted), and read failures are pushed to stats.errors while the batch proceeds; inserts use onConflictDoNothing (json-migration.ts:202-205, 87-97, 100-108)

#### Scenario: Bulk-insert tuning
- **WHEN** the migration runs
- **THEN** it sets PRAGMA synchronous=OFF and wraps all inserts in a single BEGIN/COMMIT transaction with batchSize 1000 for throughput (json-migration.ts:47-49, 69, 152, 415)

### Requirement: Git-Based Session Snapshots (track/patch/revert)
The system SHALL capture workspace file state as commits in a per-project shadow git repo (Global.Path.data/snapshot/<projectID>) so that a message's recorded snapshot hash can later be diffed, patched, restored or reverted.

#### Scenario: Track writes a tree hash
- **WHEN** the processor takes a snapshot before/around tool execution
- **THEN** Snapshot.track() runs `git add .` + `git write-tree` in the shadow repo and returns the tree hash, gated off when vcs!==git, ACP client, or config.snapshot===false (snapshot/index.ts:51-77; called at processor.ts:238/271)

#### Scenario: Patch lists changed files vs a snapshot
- **WHEN** Snapshot.patch(hash) is called
- **THEN** it diffs `--name-only` against the stored hash and returns absolute paths of changed files, or an empty patch on git failure (snapshot/index.ts:85-110)

#### Scenario: Revert restores prior file contents
- **WHEN** a session revert is requested
- **THEN** Snapshot.revert() checks out each patched file from its hash, deleting files that did not exist in the snapshot, and Snapshot.restore() read-trees + checkout-index a full snapshot (snapshot/index.ts:112-161; revert.ts:59-87)

#### Scenario: Full diffs feed sharing/summary
- **WHEN** a session summary or share is produced
- **THEN** Snapshot.diffFull(from,to) emits FileDiff[] with full before/after file contents and add/del counts (snapshot/index.ts:198-250), which is the payload later uploaded by share sync

### Requirement: ShareNext Outbound Sync Client
The system SHALL, unless CYBERSTRIKE_DISABLE_SHARE is set, subscribe to session/message/part/diff bus events and mirror them to a share host — defaulting to https://cybrstk.us when config enterprise.url is unset — via POST /api/share, batched POST /api/share/:id/sync, and DELETE /api/share/:id.

#### Scenario: Default host is off-box cybrstk.us
- **WHEN** ShareNext.url() resolves with no enterprise.url configured
- **THEN** it returns "https://cybrstk.us" (share-next.ts:15-17)

#### Scenario: Kill switch
- **WHEN** CYBERSTRIKE_DISABLE_SHARE is "true" or "1"
- **THEN** init/create/sync/remove all early-return and no bus subscriptions or network calls occur (share-next.ts:19-23, 71, 128, 164)

#### Scenario: 1s batch window per session
- **WHEN** the first sync-able event for a session arrives
- **THEN** a 1000ms setTimeout is armed and further events within that window are merged into the same queue map without re-arming the timer (leading fixed-window batch, not a trailing debounce); on flush it only POSTs to /sync if a share row exists for the session (share-next.ts:126-161)

#### Scenario: Create performs a full upload
- **WHEN** ShareNext.create(sessionID) succeeds
- **THEN** it POSTs {sessionID} to /api/share, persists {id,secret,url} in the local session_share table (onConflict upsert), then fullSync() uploads the session, every message, every part, the full session_diff FileDiff[] (before/after file contents) and the model list (share-next.ts:70-94, 180-210)

### Requirement: Session Share Lifecycle & Auto-Share
The system SHALL create a share on demand (or automatically for top-level sessions when share config is 'auto' or CYBERSTRIKE_AUTO_SHARE is set), reject sharing when config.share==='disabled', persist the local share record, and remove both the remote share and local record on unshare.

#### Scenario: Auto-share on session creation
- **WHEN** a session with no parentID is created and cfg.share==='auto' (or Flag.CYBERSTRIKE_AUTO_SHARE)
- **THEN** share(id) is fired-and-forgotten during creation, silently swallowing errors (session/index.ts:298-301)

#### Scenario: Disabled config blocks share
- **WHEN** Session.share() is called with cfg.share==='disabled'
- **THEN** it throws "Sharing is disabled in configuration" before any remote call (session/index.ts:322-325)

#### Scenario: Unshare tears down remote + local
- **WHEN** Session.unshare(id) runs
- **THEN** ShareNext.remove() DELETEs the remote share with its secret and deletes the local session_share row, then clears share_url on the session (session/index.ts:337-347, share-next.ts:163-178)

#### Scenario: Default is not auto but sharing is permitted
- **WHEN** config.share is left unset (undefined)
- **THEN** auto-share does NOT trigger (requires ==='auto'), but manual share() is still permitted since only ==='disabled' throws (session/index.ts:298,323; config.ts:1113-1118 has no .default())

### Requirement: Enterprise Share API & Event-Log Storage
The system SHALL expose four Hono routes (POST /api/share, POST /api/share/:id/sync, GET /api/share/:id/data, DELETE /api/share/:id) that derive the share id from the last 8 characters of the sessionID, gate mutations behind a random-UUID secret, append sync payloads to an ordered event log, and compact that log into a materialized snapshot on read.

#### Scenario: Share id is derived, not random
- **WHEN** Share.create is called for a sessionID
- **THEN** the id is set to (test-prefix +) sessionID.slice(-8) and a random crypto.randomUUID() secret is generated; if a share with that id already exists it throws AlreadyExists (core/share.ts:41-52)

#### Scenario: Sync appends to an ordered event log
- **WHEN** POST /api/share/:id/sync arrives with the correct secret
- **THEN** the payload array is written to key [share_event, id, Identifier.descending()] as an append-only log entry; a wrong secret throws InvalidSecret (core/share.ts:69-80, api/[...path].ts:64-91)

#### Scenario: Read compacts events into a snapshot
- **WHEN** GET /api/share/:id/data is called
- **THEN** Share.data() loads the prior compaction snapshot, lists pending share_event keys after compaction.event, merges each item by a type/id key via binary search, persists the new compaction, and returns the merged Data[] (core/share.ts:87-131)

#### Scenario: Delete purges share + data
- **WHEN** DELETE /api/share/:id runs with the matching secret
- **THEN** Share.remove() removes the share record and every share_data object under the id, else throws NotFound/InvalidSecret (core/share.ts:58-67, api/[...path].ts:114-138)

### Requirement: Enterprise Viewer & S3/R2 Object Storage
The system SHALL back the enterprise share store with an S3 or Cloudflare-R2 object-storage adapter (aws4fetch) selected by CYBERSTRIKE_STORAGE_ADAPTER, and render shared sessions in a public SolidStart viewer that server-side loads the compacted data and reconstructs turns, diffs, and model metadata.

#### Scenario: Adapter chosen by env
- **WHEN** the storage adapter is first resolved
- **THEN** CYBERSTRIKE_STORAGE_ADAPTER selects r2() (accountId.r2.cloudflarestorage.com) or s3() (s3.<region>.amazonaws.com), else it throws "No storage adapter configured"; keys are `<join('/')>.json` objects (core/storage.ts:66-99)

#### Scenario: Viewer server-loads and rebuilds session
- **WHEN** the /share/:shareID route renders
- **THEN** getData() ("use server") calls Share.get + Share.data, buckets items into session/message/part/model/diff maps, preloads unified+split multi-file diffs, and throws SessionDataMissingError→404 if the share or its session record is absent (routes/share/[shareID].tsx:49-147)

#### Scenario: Public no-index viewer with off-site OG card
- **WHEN** a share page renders
- **THEN** it emits <Meta robots=noindex,nofollow> yet builds an OpenGraph image URL on social-cards.sst.dev embedding the base64 session title (routes/share/[shareID].tsx:182, 197-209)

### Requirement: Off-Box Exfil Default & Unauthenticated Share Read (Security Posture)
The system SHALL, by default, transmit shared session content — including full before/after source-file diffs — to the third-party host cybrstk.us, expose share-data reads with no secret behind deterministic sessionID-derived ids (an 8-char slice of the session's random base62 tail, not a separate secret token), and collide shares across tenants in a single global id namespace.

#### Scenario: Full source contents leave the box by default
- **WHEN** any share is created without configuring enterprise.url
- **THEN** fullSync uploads the complete FileDiff[] (before+after full file contents) plus all messages/parts to https://cybrstk.us — a domain the operator does not own (share-next.ts:15-17, 180-210; snapshot/index.ts:226-247)

#### Scenario: Reads need no secret
- **WHEN** anyone issues GET /api/share/:id/data
- **THEN** the route returns the full compacted session with no secret check — only sync/remove verify the secret (create mints a new secret rather than verifying one) (api/[...path].ts:92-113 vs 64-91/114-138)

#### Scenario: Share ids are derived from the sessionID, not an independent secret
- **WHEN** a share is created for a sessionID
- **THEN** the share id is a deterministic 8-char slice of the sessionID's random base62 tail (~62^8 ≈ 2.2e14 combinations, so not practically brute-forceable), not a freshly generated secret token — the id is fully determined by the sessionID and stable rather than a rotating capability (core/share.ts:44; identifier.ts:50)

#### Scenario: Cross-tenant id collision
- **WHEN** two sessions from different installs share the same trailing 8 chars in one global bucket
- **THEN** the second Share.create throws AlreadyExists (a cross-tenant DoS / namespace clash) because ids are not namespaced per tenant (core/share.ts:48-49)

## Notes
- OFF-BOX EXFIL: share host defaults to https://cybrstk.us (share-next.ts:16), an external domain the operator does not own. A created share uploads full source-file diffs (before+after complete file contents, snapshot/index.ts:226-247) plus all messages/parts. For a pentest tool this means target source/artifacts can leave the box to a third party. Mitigations present: set config enterprise.url to self-host, config share:'disabled', or env CYBERSTRIKE_DISABLE_SHARE=1/true.
- SHARING IS NOT AUTO BY DEFAULT but IS PERMITTED: config.share has no .default() (config.ts:1113), so auto-share only fires when share==='auto' or CYBERSTRIKE_AUTO_SHARE is set (session/index.ts:298); however manual share() works unless share==='disabled'. So the risky default is the destination + unauthenticated read, not silent auto-upload.
- UNAUTHENTICATED READ + DERIVED IDS: GET /api/share/:id/data performs NO secret check (api/[...path].ts:92-113); only sync/remove verify the UUID secret. The share id is a deterministic 8-char tail of the sessionID (core/share.ts:44), not a freshly generated token — but that tail is the session's random base62 suffix (identifier.ts:50), ~62^8 combinations, so it is NOT practically brute-forceable/enumerable. The real exposure is that anyone who obtains the id (e.g. via the share URL) reads the full session with no authentication, and the id is a stable derivation of the sessionID rather than a rotating capability.
- ENTERPRISE ID COLLISION: ids live in one global (non-tenant-scoped) storage namespace. Two sessions whose ids share the last 8 chars collide; Share.create's get()+AlreadyExists check (core/share.ts:48-49) makes the second create throw, a cross-tenant DoS. Birthday-bound collisions become plausible only at large share counts, but there is zero tenant isolation on the key.
- BATCHING NUANCE: the '1s debounce' is a fixed leading window, not a trailing debounce — the timer is armed on the first event and NOT reset by later events (share-next.ts:129-135,142). Also queue items are keyed by a fresh ulid() because the wrapper objects {type,data} have no top-level 'id' (share-next.ts:132,139), so repeated part-updates in a window are NOT coalesced — each is uploaded. If no share row exists at flush time the queued data is silently dropped (share-next.ts:146-147).
- syncOld() (core/share.ts:133-172) writes per-object share_data/* keys and is the format the DELETE path still purges (remove() lists share_data/*), but the live sync() uses the share_event log + compaction model; the two storage layouts coexist.
- Snapshot 'track' returns a git write-tree hash (not a commit); parts persist that hash so revert/restore can rebuild workspace state (processor.ts:238/271, revert.ts:59-87). Snapshots are per-project under Global.Path.data/snapshot/<projectID>, gated off for non-git/ACP or config.snapshot===false.
- No inflated README claim found for sharing specifically; README:276 says app.cyberstrike.io (the static TUI web app) has 'no backend, no data storage' — that is a different host from the share backend (cybrstk.us) and does not describe the share pipeline, so it is not contradicted but could be misread as implying no data ever leaves the box.
- The enterprise share package (Hono + SolidStart + aws4fetch S3/R2, cybrstk.us domain, social-cards.sst.dev OG cards) is inherited near-verbatim from opencode's share app, rebranded; the cyberstrike-specific additions are the pentest tables in session.sql.ts (request/web_*/vulnerability/coverage_note/observation) and the JSON->SQLite migration, none of which are shared (share Data union only covers session/message/part/session_diff/model).

## Key Files
- `packages/cyberstrike/src/session/session.sql.ts` — Drizzle SQLite schema: session/message/part + pentest tables, FKs, indexes, unique dedup index
- `packages/cyberstrike/src/storage/db.ts` — Lazy DB client, pragmas, Drizzle migrate + reconcile() drift/reshape repair, Database.use/transaction context
- `packages/cyberstrike/src/storage/json-migration.ts` — First-run JSON->SQLite bulk importer (glob scan, batched inserts, orphan/error handling, single txn)
- `packages/cyberstrike/src/index.ts` — Startup: db-file marker gate that triggers JsonMigration.run once with a progress bar (lines 85-119)
- `packages/cyberstrike/src/snapshot/index.ts` — Git shadow-repo snapshots: track/patch/restore/revert/diff/diffFull (FileDiff before/after)
- `packages/cyberstrike/src/share/share-next.ts` — Outbound share client: url() default cybrstk.us, bus subscriptions, create/sync(1s batch)/remove, CYBERSTRIKE_DISABLE_SHARE, fullSync
- `packages/cyberstrike/src/share/share.sql.ts` — Local session_share table (session_id PK, id, secret, url)
- `packages/cyberstrike/src/session/index.ts` — Session.share/unshare + auto-share trigger (cfg.share==='auto'/flag) and 'disabled' guard (lines 297-347)
- `packages/cyberstrike/src/config/config.ts` — share enum manual/auto/disabled + deprecated autoshare->auto migration (lines 292-294, 1113-1122)
- `packages/enterprise/src/core/share.ts` — Enterprise Share core: id=sessionID.slice(-8), UUID secret, event-log append + compaction in data()
- `packages/enterprise/src/core/storage.ts` — S3/R2 object-storage adapter (aws4fetch) selected by CYBERSTRIKE_STORAGE_ADAPTER
- `packages/enterprise/src/routes/api/[...path].ts` — Hono API: 4 routes POST /share, POST /share/:id/sync, GET /share/:id/data, DELETE /share/:id
- `packages/enterprise/src/routes/share/[shareID].tsx` — Public SolidStart viewer: server getData(), diff preload, no-index meta, off-site OG card
- `packages/cyberstrike/src/cli/cmd/import.ts` — Reverse path: fetch GET /api/share/:slug/data and re-insert into local SQLite
