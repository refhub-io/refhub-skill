# RefHub Skill Spec

## 1. Purpose

`refhub-skill` is an agent-facing adapter for RefHub.

Its job is to let an agent reliably do useful reference-management work without exposing raw backend complexity or pretending unsupported operations exist.

The skill is optimized for:

- managing vault structure (create, configure, share)
- reading vaults for analysis and synthesis
- adding, updating, deleting, and importing references
- organizing items with tags and relations
- searching and syncing vault contents incrementally
- exporting vault contents for downstream tools
- working safely with scoped API keys

It should not become a second backend, an alternate database contract, or a thin wrapper around every future endpoint.

## 2. Design principles

### 2.1 API is canonical

The RefHub API defines the source of truth for what the skill can do over the network.

### 2.2 Workflow-first, not endpoint-first

The skill exposes tasks like:

- create and configure a vault
- inspect a vault and its contents
- add or import references to a vault
- update notes or metadata on an existing item
- classify items with tags and relations
- search for items without a full vault download
- export a vault for local processing

not generic transport verbs.

### 2.3 Honest capability boundaries

If the product supports something via direct Supabase/frontend code but the public API does not yet support it, the skill must report that explicitly and fail clearly.

### 2.4 Stable outputs over cleverness

The skill should return structured results that downstream agents can rely on:

- normalized vault/item summaries
- request IDs when available
- explicit partial-failure reporting
- per-item action outcomes for upsert operations

### 2.5 No fake orchestration

No placeholder commands, no fake tool handlers, no pretend offline sync layer.

## 3. Core domain model

- `vault`
  - collection boundary for references and collaboration
  - key fields: `id`, `name`, `description`, `color`, `visibility`, `category`, `abstract`, `updated_at`
  - visibility values: `private`, `protected`, `public`
- `vault item`
  - agent-facing term for a `vault_publications` row
  - carries title, authors, year, DOI, URL, abstract, notes, BibTeX-oriented metadata, version
- `canonical publication`
  - underlying `publications` row created alongside a vault item; shared across vaults
- `tag`
  - vault-scoped, supports `name`, `color`, optional `parent_id` for hierarchy
- `publication tag`
  - joins tag IDs to vault items
- `publication relation`
  - connects vault items; supported types include `cites`, `extends`, `builds_on`, `contradicts`, `reviews`, `related`
- `vault share`
  - collaborator entry: user + role (`viewer`, `editor`, `owner`)
- `api key`
  - scoped bearer credential; may carry `vaults:read`, `vaults:write`, `vaults:export`, `vaults:admin`

## 4. Actor model

### 4.1 Human owner

Creates and manages RefHub API keys through authenticated management routes (Supabase session JWT).

### 4.2 Agent using the skill

Uses an existing API key to perform constrained RefHub workflows. Key scope and vault restrictions bound what the agent can do.

### 4.3 RefHub API

Authoritative service for all vault, item, tag, relation, import, search, audit, and export operations.

## 5. Skill goals

The skill supports all workflows already backed by the current public API:

1. Discover accessible vaults.
2. Read one vault with its full contents.
3. Create, update metadata, and delete vaults.
4. Set vault visibility and manage collaborators.
5. Add one or more items to a vault.
6. Update one existing vault item.
7. Delete an item from a vault.
8. Bulk-upsert items by DOI or title+year (with idempotency).
9. Preview what a bulk upsert would do (dry-run).
10. Import items from a DOI, a BibTeX string, or a URL.
11. List, create, update, and delete vault-scoped tags.
12. Attach and detach tags from items.
13. List, create, update, and delete relations between items.
14. Search/filter items within a vault.
15. Fetch vault stats (counts and last updated timestamp).
16. Fetch items changed since a given timestamp (incremental sync).
17. Export a vault as JSON or BibTeX.
18. Read audit logs for the key's owner, optionally scoped to a vault.
19. Provide enough response structure for downstream summarization, note generation, or local transformation.

## 6. Non-goals

Do not include these unless the API lands first:

- vault archiving, unarchiving, or soft-delete
- vault duplication or clone
- item restore after deletion
- item revision history
- item move/copy between vaults
- bulk relation import
- webhooks or event subscriptions
- full-text index-backed search (ILIKE is used for vault-scoped queries)

## 7. Primary workflows

### 7.1 List vaults

Intent: let an agent discover where it can act.

Endpoint: `GET /api/v1/vaults`

Output includes permission level and item count per vault.

### 7.2 Read vault

Intent: load enough structured content for analysis, summarization, curation, or export preparation.

Endpoint: `GET /api/v1/vaults/:vaultId`

Returns vault metadata, vault items, tags, item-tag links, and relations.

### 7.3 Create vault

Intent: set up a workspace for an automation task.

Endpoint: `POST /api/v1/vaults`

Requires `vaults:admin`. Body: `{ name, description?, color?, visibility?, category?, abstract? }`. Visibility defaults to `private`.

### 7.4 Update vault

Intent: revise vault metadata.

Endpoint: `PATCH /api/v1/vaults/:vaultId`

Requires `vaults:admin` + owner permission. Accepts any subset of `{ name, description, color, category, abstract }`. Does not accept `visibility` — use the visibility endpoint for that.

### 7.5 Delete vault

Intent: remove a vault and all its contents permanently.

Endpoint: `DELETE /api/v1/vaults/:vaultId`

Requires `vaults:admin` + owner permission. Hard delete. Cascades: items, tags, shares, API key vault restrictions. Returns `200 { data: { id } }`. No undo.

### 7.6 Set visibility

Intent: control who can discover and access the vault.

Endpoint: `PATCH /api/v1/vaults/:vaultId/visibility`

Requires `vaults:admin` + owner permission. Body: `{ visibility: 'private'|'protected'|'public', public_slug?: string }`.
- `public` requires a `public_slug` (lowercase alphanumeric + hyphens, unique).
- `private` clears `public_slug`.
- Adding a share to a `private` vault auto-upgrades it to `protected`.

### 7.7 Share management

Intent: add or modify collaborators on a vault.

Endpoints: `GET/POST /api/v1/vaults/:vaultId/shares`, `PATCH/DELETE /api/v1/vaults/:vaultId/shares/:shareId`

List requires `vaults:read` + viewer. All mutations require `vaults:admin` + owner. Valid roles: `viewer`, `editor`. `POST` body: `{ email, role, name? }` — `email` is required. DELETE returns `200 { data: { id } }`.

### 7.8 Add items

Intent: import references prepared elsewhere.

Endpoint: `POST /api/v1/vaults/:vaultId/items`

Accepts one or more items. Each must include `title`. Tag IDs must already exist. Backend prevalidates and attempts rollback on failure.

### 7.9 Update item

Intent: revise notes, metadata, or tags on an existing reference.

Endpoint: `PATCH /api/v1/vaults/:vaultId/items/:itemId`

Partial update. If `tag_ids` is present, it replaces the full tag set. Increments `version` on successful metadata updates.

Skill expectation: make tag replacement semantics explicit; distinguish `item not found` from `permission denied`.

### 7.10 Delete item

Intent: permanently remove a vault item.

Endpoint: `DELETE /api/v1/vaults/:vaultId/items/:itemId`

Requires `vaults:write` + editor. Hard delete from `vault_publications`. The underlying `publications` row is preserved (may be referenced by other vaults). Returns `200 { data: { id } }`.

### 7.11 Bulk upsert items

Intent: idempotently import a set of references.

Endpoint: `POST /api/v1/vaults/:vaultId/items/upsert`

Body: `{ items: [...], idempotency_key?: string }`. Match strategy: DOI first, then `bibtex_key` — items with neither are always created. Response: `{ data: { created: [...], updated: [...], errors: [...] } }`. If `idempotency_key` was seen within **5 minutes**, returns the cached result without re-executing.

### 7.12 Import preview (dry-run)

Intent: show the user what would change before committing.

Endpoint: `POST /api/v1/vaults/:vaultId/items/import-preview`

Requires only `vaults:read`. Same body as upsert. Writes nothing. Response: `{ data: { would_create: [...], would_update: [...], invalid: [...] } }`.

### 7.13 Import from DOI

Intent: enrich and create an item from a DOI in one step.

Endpoint: `POST /api/v1/vaults/:vaultId/import/doi`

Body: `{ doi, tag_ids?: [] }`. Calls Semantic Scholar internally. Returns the created item. Returns `409` if the DOI already exists in the vault.

### 7.14 Import from BibTeX

Intent: bulk-import from a BibTeX string.

Endpoint: `POST /api/v1/vaults/:vaultId/import/bibtex`

Body: `{ bibtex, tag_ids?: [] }`. Parses single or multi-entry BibTeX. Skips entries where `bibtex_key` already exists in the vault. Returns `{ created: [...], skipped: [...] }`.

### 7.15 Import from URL

Intent: create an item from a webpage.

Endpoint: `POST /api/v1/vaults/:vaultId/import/url`

Body: `{ url, tag_ids?: [] }`. Fetches Open Graph / meta tags. Best-effort: creates with whatever metadata is available (at minimum `title` + `url`).

### 7.16 Tag CRUD

Intent: make tags independently manageable rather than only accessible through full vault reads.

Endpoints: `GET/POST /api/v1/vaults/:vaultId/tags`, `PATCH/DELETE /api/v1/vaults/:vaultId/tags/:tagId`

- Create: `{ name, color?, parent_id? }`. Depth computed from parent chain automatically.
- Update: any subset of `{ name, color, parent_id }`. Guards against circular parent references.
- Delete: cascades `publication_tags`; child tags bubble up to deleted tag's parent. Returns `200 { data: { id } }`.

### 7.17 Attach/detach tags

Intent: assign or remove tags from an existing item without a full item update.

Endpoints: `POST /api/v1/vaults/:vaultId/tags/attach`, `POST /api/v1/vaults/:vaultId/tags/detach`

Attach is idempotent — no duplicate rows. Detach silently ignores tag IDs not currently attached.

### 7.18 Relation CRUD

Intent: make item relationships independently manageable.

Endpoints: `GET/POST /api/v1/vaults/:vaultId/relations`, `PATCH/DELETE /api/v1/vaults/:vaultId/relations/:relationId`

List supports `?type=` query param only. Create is not idempotent — submitting a duplicate pair will error; check before creating. Only `relation_type` is mutable after creation. Delete returns `200 { data: { id } }`.

### 7.19 Search and filter items

Intent: answer narrow questions without downloading a full vault.

Endpoints: `GET /api/v1/vaults/:vaultId/search` or `GET /api/v1/vaults/:vaultId/items`

Query params: `?q=`, `?author=`, `?year=`, `?doi=`, `?tag_id=`, `?type=`, `?page=`, `?limit=` (default 20, max 100). Returns paginated items with their `tag_ids`.

### 7.20 Vault stats

Intent: quick summary for an agent planning how to work with a vault.

Endpoint: `GET /api/v1/vaults/:vaultId/stats`

Returns `{ item_count, tag_count, relation_count, last_updated }`.

### 7.21 Incremental sync (changes feed)

Intent: detect what changed since last sync without a full vault download.

Endpoint: `GET /api/v1/vaults/:vaultId/changes?since=<ISO timestamp>`

Returns all `vault_publications` where `updated_at > since`.

### 7.22 Export vault

Intent: hand off a vault to local scripts, citation tooling, or another agent stage.

Endpoint: `GET /api/v1/vaults/:vaultId/export?format=json|bibtex`

Requires `vaults:export`. Returns attachment-like serialized output.

### 7.23 Read audit log

Intent: observe what operations were performed, by whom, and when.

Endpoints:
- `GET /api/v1/vaults/:vaultId/audit` — vault-scoped (API key auth)
- `GET /api/v1/audit` — all requests for key's owner (JWT management auth)

Query params: `?since=`, `?until=`, `?limit=` (default 50, max 200), `?page=`.

## 8. Failure behavior

The skill should normalize failures into a predictable structure.

Categories:

- `auth_error` — missing API key, invalid key format, expired key, revoked key
- `permission_error` — missing scope, vault access denied, write access denied, wrong vault permission level
- `input_error` — invalid JSON, invalid item payload, invalid tag IDs, unsupported export format, unknown relation type
- `not_found` — vault not found, item not found, tag not found, relation not found, route not found
- `service_error` — internal backend error, rollback failure, transient upstream issue

Every surfaced error should keep:

- HTTP status when available
- RefHub error code when available
- human-readable message
- `request_id` when available

## 9. Authentication assumptions

The skill should assume:

- API key creation and revocation are management concerns outside normal agent runtime
- normal skill execution uses a pre-issued `rhk_<publicId>_<secret>` key
- keys may be scope-limited to any combination of `vaults:read`, `vaults:write`, `vaults:export`, `vaults:admin`
- keys may be vault-restricted

The skill should not require a Supabase session JWT for ordinary use.

## 10. Skill interface shape

Conceptual operations:

- `vaults.list` / `vaults.get` / `vaults.create` / `vaults.update` / `vaults.delete`
- `vaults.setVisibility`
- `vaults.shares.list` / `vaults.shares.add` / `vaults.shares.update` / `vaults.shares.remove`
- `items.add` / `items.update` / `items.delete` / `items.upsert` / `items.importPreview`
- `items.search` / `items.stats` / `items.changes`
- `import.doi` / `import.bibtex` / `import.url`
- `tags.list` / `tags.create` / `tags.update` / `tags.delete` / `tags.attach` / `tags.detach`
- `relations.list` / `relations.create` / `relations.update` / `relations.delete`
- `vaults.export`
- `audit.list`

These names are stable enough for the spec and can later be mapped onto CLI verbs and MCP tools.

## 11. Data handling rules

- Treat RefHub as the system of record.
- Avoid hidden local state beyond transient request execution.
- Never infer missing IDs when the API requires explicit IDs.
- Do not silently create tags.
- Do not silently re-run bulk writes after ambiguous failure.
- Prefer idempotent read flows and explicit write confirmations.
- Vault and item deletes are permanent — warn before proceeding.

## 12. What exists now vs what is deferred

### Exists now (public API)

- API key management routes
- vault list / read / create / update / delete / visibility / shares
- item add / update / delete / upsert / import-preview
- DOI, BibTeX, and URL import
- tag CRUD + attach/detach
- relation CRUD
- search, stats, and changes feed
- JSON and BibTeX export
- audit log read endpoints

### Deferred

- vault archiving and soft-delete
- item revision history and restore
- item move/copy between vaults
- webhooks and event delivery
- audit log viewer in the frontend

## 13. Acceptance criteria

The skill implementation is acceptable when:

- every workflow maps directly to a current public API route
- unsupported workflows fail clearly with documented reasons
- outputs are structured enough for downstream agents
- the implementation does not require direct Supabase access
- the same operation names can be mirrored later by a CLI and MCP server
