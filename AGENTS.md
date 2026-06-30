# RefHub Agent Instructions

Agent-facing instructions for operating [RefHub](https://refhub.io) through its public API (v2). Covers the full API surface across both auth modes — API key for data routes, session JWT for management routes. No local state, no Supabase direct access, no invented behavior.

## When to apply these instructions

Apply whenever the user asks you to:
- read, search, or export vault contents
- add, update, delete, or import references
- create or configure vaults (name, visibility, collaborators)
- manage tags or relations on vault items
- sync changes incrementally
- enrich incomplete publication metadata from Semantic Scholar
- upload a PDF and store it in the user's linked Google Drive
- manage API keys or Google Drive settings

Do **not** apply for features with no API route: vault archiving, item revision history, item move/copy between vaults, webhooks.

## Execution layer

If the `refhub` CLI is available (`which refhub` succeeds), use it instead of making HTTP calls directly.

```sh
export REFHUB_API_KEY=rhk_<publicId>_<secret>   # for data routes
refhub --help            # discover commands
refhub vaults --help     # group-level help
```

Exit codes: `0` success · `1` API error · `2` bad arguments · `3` auth error.

The CLI covers data routes, API-key Semantic Scholar discovery/enrichment, and API-key item-scoped PDF upload. For account setup routes (Google Drive link management, key management) fall back to direct HTTP (base URL: `https://refhub-api.netlify.app/api/v1`).

## Authentication

Two modes — never mix them.

**API key** (all data routes):
```
Authorization: Bearer rhk_<publicId>_<secret>
```

**Session JWT** (management routes — key lifecycle, Google Drive connect/disconnect, legacy publication-level PDF upload, global audit):
```
Authorization: Bearer <supabase-session-jwt>
```

Sending an API key to a management route returns `401 refhub_api_key_not_supported`. Switch to JWT for that call — do not retry with the API key.

**Scope requirements:**

| Scope | Grants |
|---|---|
| `vaults:read` | list/read vaults, search, stats, changes, audit |
| `vaults:write` | add/update/delete items, tags, relations, import |
| `vaults:export` | export vault as JSON or BibTeX |
| `vaults:admin` | create/update/delete vaults, visibility, shares |

If a credential is missing or has insufficient scope: stop, report clearly, ask the user to provide a valid key. Do not attempt workarounds.

## Guardrails — follow these without exception

- **Never infer `vault_id` from a vault name.** Always call `GET /vaults` and resolve it from the response.
- **Use the right auth for the right route.** Management routes (key management, Google Drive link management, global audit) require a session JWT and reject API keys with `401 refhub_api_key_not_supported`. Data routes require an API key.
- **Never assume a tag exists.** List tags before using `tag_ids` in any write.
- **Never create tags implicitly during item writes.** Tag creation is a separate step.
- **Never retry a bulk write after ambiguous failure** unless you have an `idempotency_key`.
- **Never assume frontend capability equals API support.** UI features backed by direct Supabase may not have a public API route.
- **Never proceed with vault or item deletion without explicit user confirmation.** Both are hard deletes with no undo.
- **Never send `visibility` or `public_slug` in `PATCH /vaults/:vaultId`.** Use the dedicated visibility endpoint.
- **`tag_ids` on item update is a full replacement, not an append.** Make this explicit to the user before patching.

## Supported operations

### Vaults

```
GET    /vaults                              # list (vaults:read)
GET    /vaults/:vaultId                     # read with items/tags/relations (vaults:read)
POST   /vaults                              # create (vaults:admin)
PATCH  /vaults/:vaultId                     # update metadata (vaults:admin + owner)
DELETE /vaults/:vaultId                     # hard delete — confirm first (vaults:admin + owner)
PATCH  /vaults/:vaultId/visibility          # set visibility/slug (vaults:admin + owner)
GET    /vaults/:vaultId/shares              # list collaborators (vaults:read)
POST   /vaults/:vaultId/shares              # add collaborator — email required, role: viewer|editor (vaults:admin)
PATCH  /vaults/:vaultId/shares/:shareId     # update role (vaults:admin)
DELETE /vaults/:vaultId/shares/:shareId     # remove collaborator (vaults:admin)
```

Notes:
- `visibility` values: `private` | `protected` | `public`
- `public` requires a unique `public_slug` (lowercase alphanumeric + hyphens)
- Adding a share to a `private` vault auto-upgrades it to `protected`
- Vault delete cascades: all items, tags, shares, and key restrictions are removed

### Items

```
POST   /vaults/:vaultId/items               # add one or more items (vaults:write)
PATCH  /vaults/:vaultId/items/:itemId       # update item (vaults:write)
DELETE /vaults/:vaultId/items/:itemId       # hard delete — confirm first (vaults:write)
POST   /vaults/:vaultId/items/upsert        # bulk upsert by DOI / bibtex_key (vaults:write)
POST   /vaults/:vaultId/items/import-preview # dry-run upsert — writes nothing (vaults:read)
```

Notes:
- Each item must include `title`
- `authors` is a **string array** e.g. `["Smith J", "Doe A"]`, not a plain string
- `tag_ids` must reference existing vault tags
- `tag_ids` on update **replaces the full tag set** — not additive
- Bulk upsert matches on DOI first, then `bibtex_key`; items with neither are always created
- Pass `idempotency_key` on bulk upsert for safe retries (TTL: 5 minutes)
- Item delete removes the `vault_publications` row; the underlying `publications` row is preserved

### Import

```
POST   /vaults/:vaultId/import/doi          # from DOI (vaults:write)
POST   /vaults/:vaultId/import/bibtex       # from BibTeX string (vaults:write)
POST   /vaults/:vaultId/import/url          # from URL via Open Graph (vaults:write)
```

Notes:
- DOI import calls Semantic Scholar server-side; returns `409` if DOI already exists in vault
- BibTeX import skips entries where `bibtex_key` already exists; returns `{ created, skipped }`
- URL import is best-effort: creates item with whatever metadata is available

### Tags

```
GET    /vaults/:vaultId/tags                # list (vaults:read)
POST   /vaults/:vaultId/tags                # create { name, color?, parent_id? } (vaults:write)
PATCH  /vaults/:vaultId/tags/:tagId         # update { name?, color?, parent_id? } (vaults:write)
DELETE /vaults/:vaultId/tags/:tagId         # delete; child tags bubble up to parent (vaults:write)
POST   /vaults/:vaultId/tags/attach         # attach { item_id, tag_ids } — idempotent (vaults:write)
POST   /vaults/:vaultId/tags/detach         # detach { item_id, tag_ids } — ignores unattached (vaults:write)
```

### Relations

```
GET    /vaults/:vaultId/relations?type=     # list; only type filter supported (vaults:read)
POST   /vaults/:vaultId/relations           # create — not idempotent, check before creating (vaults:write)
PATCH  /vaults/:vaultId/relations/:id       # update relation_type only (vaults:write)
DELETE /vaults/:vaultId/relations/:id       # delete (vaults:write)
```

Notes:
- `publication_id` and `related_publication_id` are the item's `id` field (the `vault_publications` row id), **not** `original_publication_id`
- Supported types: `cites` `extends` `builds_on` `contradicts` `reviews` `related`
- Duplicate pair returns an error — list relations before creating

### Search, stats, sync

```
GET    /vaults/:vaultId/search?q=&author=&year=&doi=&tag=&type=&page=&per_page=   # (vaults:read)
GET    /vaults/:vaultId/stats               # item/tag/relation counts + last_updated (vaults:read)
GET    /vaults/:vaultId/changes?since=      # items updated after ISO timestamp (vaults:read)
```

- `per_page` default 25, max 100

### Semantic Scholar enrichment and lookup

These routes are API-key-compatible under `/semantic-scholar/*` and require `vaults:read`. Legacy root routes remain JWT/browser frontend routes.

```text
POST   /semantic-scholar/doi-metadata      # { doi } → full metadata; null if not found
POST   /semantic-scholar/lookup            # { doi } or { title } → Semantic Scholar paper_id
POST   /semantic-scholar/search            # { query, limit? } → topic/keyword search
POST   /semantic-scholar/recommendations   # { paper_id, limit? } → recommended papers
POST   /semantic-scholar/related           # alias for recommendations
POST   /semantic-scholar/references        # { paper_id, limit? } → papers this paper cites
POST   /semantic-scholar/citations         # { paper_id, limit? } → papers citing this paper
POST   /semantic-scholar/cited-by          # alias for citations
```

CLI: `refhub discover ...` for lookup/search/graph traversal, and `refhub enrich --vault <id> [--item <id>] [--dry-run]` for DOI metadata enrichment.

### PDF upload

Requires `vaults:write` and Google Drive already linked to the account through the web UI.

```
POST   /vaults/:vaultId/items/:itemId/pdf          # small raw application/pdf bytes → stored in Drive
POST   /vaults/:vaultId/items/:itemId/pdf/session  # create large-PDF resumable Drive session
POST   /vaults/:vaultId/items/:itemId/pdf/complete # complete large-PDF resumable upload
```

- Raw API uploads are capped at the smallest of `REFHUB_API_MAX_BODY_BYTES`, `GOOGLE_DRIVE_MAX_UPLOAD_BYTES`, and the Netlify synchronous Function ceiling (6 MiB).
- Larger vault-item PDFs use the API-key resumable flow: create `/pdf/session`, direct `PUT` bytes to the returned Google Drive `upload_url`, then call `/pdf/complete`.
- CLI: `refhub pdf upload --vault <vaultId> --item <itemId> --file <path.pdf>`
- Errors: `404 publication_not_found` · `413 pdf_upload_too_large_for_api` · `503 drive_not_linked` · `502 drive_upload_failed`

### Export and audit

```
GET    /vaults/:vaultId/export?format=json|bibtex   # (vaults:export)
GET    /vaults/:vaultId/audit?since=&until=&per_page=&page=   # vault-scoped (any API key)
GET    /audit?since=&until=&per_page=&page=            # global (JWT only)
```

- Audit `per_page` default 25, max 100

## Error handling

| Status | Action |
|---|---|
| `401 missing_api_key` / `invalid_api_key_format` / `expired` / `revoked` | Stop. Report clearly. Ask user for a valid key. Do not retry with same key. |
| `401 refhub_api_key_not_supported` | API key sent to a JWT-only route. Switch auth mode or stop. |
| `403 missing_scope` | Report which scope is needed. Do not attempt a workaround. |
| `403 vault_access_denied` / `vault_not_found` | Report and stop. |
| `404` | Resource doesn't exist. Verify the id. Do not create a replacement silently. |
| `409` | Already exists (DOI import, duplicate relation). Surface the existing resource id. |
| `413 request_too_large`, `pdf_upload_too_large_for_api` | Split batch requests; for PDFs, use API-key `/pdf/session`, direct Drive `PUT`, then `/pdf/complete` instead of retrying the raw upload. |
| `429 rate_limit_exceeded` | Back off using `retry_after_seconds`. |
| `500 bulk_insert_partial_failure` | Partial write may have occurred. Do not retry without `idempotency_key`. Alert user for manual review. |
| `5xx` | Retry once with backoff. If it persists, report and stop. |

Every error response includes `error.code`, `error.message`, and `meta.request_id`. Always surface `request_id` when reporting failures.

## Not yet implemented

Do not attempt these — the current public API does not support them:

- Vault archiving, soft-delete, or restore
- Item revision history or restore
- Item move or copy between vaults
- Webhooks or event subscriptions
- Direct Supabase access as a fallback

State this clearly if the user requests one; do not improvise an alternative.


## API-key Semantic Scholar and PDF workflows (2026-06)

Normal agent runtime is API-key-only:

- Semantic Scholar: `POST /api/v1/semantic-scholar/lookup`, `/doi-metadata`, `/search`, `/recommendations`, `/related`, `/references`, `/citations`, `/cited-by`; all require `vaults:read`. CLI: `refhub discover ...` and `refhub enrich --vault <id> [--item <id>] [--dry-run]`.
- Item PDF upload requires `vaults:write` and a Google Drive account already linked in the RefHub web UI. Small PDFs use raw `POST /api/v1/vaults/:vaultId/items/:itemId/pdf`; larger vault-item PDFs use API-key `POST /pdf/session`, direct Drive `PUT` to `upload_url`, then `POST /pdf/complete`. CLI: `refhub pdf upload --vault <vaultId> --item <itemId> --file <path.pdf>`.
- Google Drive connect/disconnect, API-key lifecycle, legacy `/publications/:publicationId/pdf`, and global audit remain session-JWT/browser account-management flows.
- Search/list accepts canonical `per_page` and `tag`; backend also accepts compatibility aliases `limit` and `tag_id`. DOI filtering is supported.
