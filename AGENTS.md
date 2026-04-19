# RefHub Agent Instructions

Agent-facing instructions for operating [RefHub](https://refhub.io) through its public API (v2). All operations use a scoped API key. No local state, no Supabase direct access, no invented behavior.

## When to apply these instructions

Apply whenever the user asks you to:
- read, search, or export vault contents
- add, update, delete, or import references
- create or configure vaults (name, visibility, collaborators)
- manage tags or relations on vault items
- sync changes incrementally

Do **not** apply for: API key creation/revocation, Google Drive management, Semantic Scholar lookups — those require a human session JWT and are outside the agent runtime.

## Execution layer

If the `refhub` CLI is available (`which refhub` succeeds), use it instead of making HTTP calls directly.

```sh
export REFHUB_API_KEY=rhk_<publicId>_<secret>
refhub --help            # discover commands
refhub vaults --help     # group-level help
```

Exit codes: `0` success · `1` API error · `2` bad arguments · `3` auth error.

Fall back to direct HTTP (base URL: `https://refhub-api.netlify.app/api/v1`) when the CLI is not available.

## Authentication

Two modes — never mix them.

**API key** (all data routes):
```
Authorization: Bearer rhk_<publicId>_<secret>
```

**Session JWT** (management routes only — key lifecycle, Google Drive, Semantic Scholar):
```
Authorization: Bearer <supabase-session-jwt>
```

Sending an API key to a management route returns `401 refhub_api_key_not_supported`. Do not retry with the same key.

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
- **Never mix JWT and API key auth.** Management routes reject API keys; data routes do not accept JWTs.
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
GET    /vaults/:vaultId/search?q=&author=&year=&doi=&tag_id=&type=&page=&limit=   # (vaults:read)
GET    /vaults/:vaultId/stats               # item/tag/relation counts + last_updated (vaults:read)
GET    /vaults/:vaultId/changes?since=      # items updated after ISO timestamp (vaults:read)
```

- `limit` default 20, max 100

### Export and audit

```
GET    /vaults/:vaultId/export?format=json|bibtex   # (vaults:export)
GET    /vaults/:vaultId/audit?since=&until=&limit=&page=   # vault-scoped (any API key)
GET    /audit?since=&until=&limit=&page=            # global (JWT only)
```

- Audit `limit` default 50, max 200

## Error handling

| Status | Action |
|---|---|
| `401 missing_api_key` / `invalid_api_key_format` / `expired` / `revoked` | Stop. Report clearly. Ask user for a valid key. Do not retry with same key. |
| `401 refhub_api_key_not_supported` | API key sent to a JWT-only route. Switch auth mode or stop. |
| `403 missing_scope` | Report which scope is needed. Do not attempt a workaround. |
| `403 vault_access_denied` / `vault_not_found` | Report and stop. |
| `404` | Resource doesn't exist. Verify the id. Do not create a replacement silently. |
| `409` | Already exists (DOI import, duplicate relation). Surface the existing resource id. |
| `413 request_too_large` | Split into smaller batches. |
| `429 rate_limit_exceeded` | Back off using `retry_after_seconds`. |
| `500 bulk_insert_partial_failure` | Partial write may have occurred. Do not retry without `idempotency_key`. Alert user for manual review. |
| `5xx` | Retry once with backoff. If it persists, report and stop. |

Every error response includes `error.code`, `error.message`, and `meta.request_id`. Always surface `request_id` when reporting failures.

## Out of scope

Do not attempt these — the current public API does not support them:

- API key creation, rotation, or revocation
- Google Drive link management
- Semantic Scholar lookups, recommendations, references, citations
- Vault archiving, soft-delete, or restore
- Item revision history or restore
- Item move or copy between vaults
- Webhooks or event subscriptions
- Direct Supabase access as a fallback

State this clearly if the user requests one; do not improvise an alternative.
