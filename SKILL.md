---
name: refhub-skill
description: Use when an agent needs to read, write, organize, import, search, export, or audit content in RefHub vaults using a pre-issued RefHub API key. Covers the full v2 public API surface. Do not use for API key creation/revocation, Google Drive management, Semantic Scholar lookups, or any flow that requires a human Supabase session JWT.
---

## Execution layer

If the `refhub` CLI is available in the environment (`which refhub` succeeds), use it instead of making HTTP calls directly. The CLI handles authentication, error formatting, and consistent output.

**Setup:** The CLI reads `REFHUB_API_KEY` from the environment. A `--api-key` flag overrides it for one-off calls.

**Command reference:** Run `refhub --help` or `refhub <group> --help` (e.g. `refhub vaults --help`) to see available commands and flags.

**Output:** JSON by default. Pass `--table` for human-readable tables.

**Exit codes:** `0` success · `1` API error · `2` bad arguments · `3` auth error (missing/invalid key)

All workflows documented below map directly to CLI commands. Agents in environments without the CLI may fall back to direct HTTP as documented in the workflow sections.

# RefHub Skill

Agent-facing runtime skill for the RefHub public API (v2). All workflows execute over HTTP using a scoped API key. No local state, no Supabase direct access, no invented behavior.

> **Status:** This repo is spec-only. No SDK or library implementation exists yet. Agents operate by following this document directly.

---

## Use this skill when

- reading or searching vault contents for analysis or synthesis
- adding, updating, deleting, or importing references into a vault
- creating or configuring vaults (name, visibility, collaborators)
- managing tags or relations on vault items
- exporting a vault or syncing changes incrementally
- checking whether a RefHub workflow is supported by the current public API

## Do NOT use this skill when

- the requested action requires a Supabase session JWT (key management, Google Drive, Semantic Scholar lookups)
- the user asks to use a frontend-only feature with no public API route
- no API key is available — stop and request one before proceeding

---

## Authentication

The API has two auth modes. **Never mix them.**

### Mode 1 — API key (all data routes)

```
Authorization: Bearer rhk_<publicId>_<secret>
```

or equivalently:

```
X-API-Key: rhk_<publicId>_<secret>
```

Use this for all vault, item, tag, relation, import, search, export, and audit operations.

**Scope requirements by operation:**

| Scope | Grants |
|---|---|
| `vaults:read` | list/read vaults, search, stats, changes, audit |
| `vaults:write` | add/update/delete items, tags, relations, import |
| `vaults:export` | export vault as JSON or BibTeX |
| `vaults:admin` | create/update/delete vaults, visibility, shares |

Keys may also be **vault-restricted** — they can only operate on the vault IDs set at creation time. Check `vault_ids` on the key record.

### Mode 2 — Session JWT (management routes only)

```
Authorization: Bearer <supabase-session-jwt>
```

Use only for: `GET/POST/DELETE /api/v1/keys`, Semantic Scholar routes, Google Drive routes, `GET /api/v1/audit`.

**If an API key (`rhk_...`) is sent to a management route, the API returns `401 refhub_api_key_not_supported`.** This is expected. Do not retry with the same key.

### When credentials are missing or insufficient

- No key present → stop, report `auth_error`, ask the user to provide an API key
- Wrong scope → report `permission_error` with the missing scope; do not attempt workarounds
- Key vault-restricted and target vault not in restriction list → report `permission_error`; do not try a different vault silently

---

## Supported workflows

Base URL: `https://refhub-api.netlify.app/api/v1`

### Vaults

**List accessible vaults** — `vaults:read`
1. `GET /vaults`
2. Filter by `permission`, `item_count`, or `updated_at` from the response as needed
3. Never infer `vault_id` from a vault name — always resolve it from this list first

**Read one vault** — `vaults:read`
1. Confirm `vault_id` is known (list first if not)
2. `GET /vaults/:vaultId`
3. Response includes vault metadata, all items, tags, publication_tags, and relations

**Create vault** — `vaults:admin`
1. Confirm key has `vaults:admin` scope
2. `POST /vaults` with `{ name, description?, color?, visibility?, category?, abstract? }`
3. `visibility` defaults to `private`
4. Return the created vault id

**Update vault metadata** — `vaults:admin` + owner
1. `PATCH /vaults/:vaultId` with any subset of `{ name, description, color, category, abstract }`
2. Do not send `visibility` or `public_slug` here — use the visibility endpoint

**Delete vault** — `vaults:admin` + owner
1. **Warn the user — this is a hard delete with no undo**
2. Confirm intent explicitly before proceeding
3. `DELETE /vaults/:vaultId` → `200 { data: { id } }`
4. Cascades: all items, tags, shares, and key restrictions for this vault are removed

**Set visibility** — `vaults:admin` + owner
1. `PATCH /vaults/:vaultId/visibility` with `{ visibility: 'private'|'protected'|'public', public_slug? }`
2. `public` requires a unique `public_slug` (lowercase alphanumeric + hyphens)
3. `private` clears `public_slug`; adding a share to a `private` vault auto-upgrades it to `protected`

**Manage shares** — `vaults:admin` + owner
1. List: `GET /vaults/:vaultId/shares` — requires only `vaults:read`
2. Add: `POST /vaults/:vaultId/shares` with `{ email, role, name? }` — `email` is required; role must be `viewer` or `editor`
3. Update: `PATCH /vaults/:vaultId/shares/:shareId` with `{ role }` — role must be `viewer` or `editor`
4. Remove: `DELETE /vaults/:vaultId/shares/:shareId` → `200 { data: { id } }`

---

### Items

**Add items** — `vaults:write` + editor
1. Confirm vault access and that `tag_ids` (if any) already exist in the vault
2. `POST /vaults/:vaultId/items` with `{ items: [{ title, authors?, year?, doi?, tag_ids?, ... }] }`
3. Each item must include `title`; `tag_ids` must reference existing vault tags
4. On partial failure the backend attempts rollback; treat `bulk_insert_partial_failure` as a high-severity error requiring manual review

**Update item** — `vaults:write` + editor
1. `PATCH /vaults/:vaultId/items/:itemId` with any publication fields
2. If `tag_ids` is included, it **replaces the full tag set** — not additive
3. `version` increments automatically on metadata updates

**Delete item** — `vaults:write` + editor
1. **Warn the user — hard delete, no undo**
2. `DELETE /vaults/:vaultId/items/:itemId` → `200 { data: { id } }`
3. The underlying `publications` row is preserved (shared across vaults)

**Bulk upsert** — `vaults:write` + editor
1. Always pass an `idempotency_key` for safe retries
2. `POST /vaults/:vaultId/items/upsert` with `{ items: [...], idempotency_key? }`
3. Match strategy: DOI first, then `bibtex_key` — items without either are always created
4. Response: `{ data: { created: [...], updated: [...], errors: [...] } }`
5. A repeated `idempotency_key` within **5 minutes** returns the cached result without re-executing

**Import preview (dry-run)** — `vaults:read`
1. Use before committing a bulk upsert to show what would change
2. `POST /vaults/:vaultId/items/import-preview` with `{ items: [...] }` — writes nothing
3. Response: `{ data: { would_create: [...], would_update: [...], invalid: [...] } }`

---

### Import

**From DOI** — `vaults:write` + editor
1. `POST /vaults/:vaultId/import/doi` with `{ doi, tag_ids?: [] }`
2. Backend calls Semantic Scholar internally — fails if `SEMANTIC_SCHOLAR_API_KEY` is not configured server-side
3. Returns `409` if the DOI already exists in the vault; surface the existing item id

**From BibTeX** — `vaults:write` + editor
1. `POST /vaults/:vaultId/import/bibtex` with `{ bibtex, tag_ids?: [] }`
2. Accepts single or multi-entry BibTeX strings
3. Skips entries where `bibtex_key` already exists; returns `{ created: [...], skipped: [...] }`

**From URL** — `vaults:write` + editor
1. `POST /vaults/:vaultId/import/url` with `{ url, tag_ids?: [] }`
2. Best-effort Open Graph scrape; creates the item with whatever metadata is available

---

### Tags

All tag writes require `vaults:write` + editor. Reads require `vaults:read`.

1. List: `GET /vaults/:vaultId/tags`
2. Create: `POST /vaults/:vaultId/tags` with `{ name, color?, parent_id? }`
3. Update: `PATCH /vaults/:vaultId/tags/:tagId` with `{ name?, color?, parent_id? }`
4. Delete: `DELETE /vaults/:vaultId/tags/:tagId` → `200 { data: { id } }`; cascades publication_tags; child tags bubble up to deleted tag's parent
5. Attach: `POST /vaults/:vaultId/tags/attach` with `{ item_id, tag_ids: [] }` — idempotent
6. Detach: `POST /vaults/:vaultId/tags/detach` with `{ item_id, tag_ids: [] }` — silently ignores unattached ids

---

### Relations

All relation writes require `vaults:write` + editor. Reads require `vaults:read`.

1. List: `GET /vaults/:vaultId/relations?type=` — only `type` filter is supported
2. Create: `POST /vaults/:vaultId/relations` with `{ publication_id, related_publication_id, relation_type? }` — both items must belong to the vault; duplicate pair will error
3. Update: `PATCH /vaults/:vaultId/relations/:relationId` with `{ relation_type }` — only the type is mutable
4. Delete: `DELETE /vaults/:vaultId/relations/:relationId` → `200 { data: { id } }`

Supported relation types: `cites`, `extends`, `builds_on`, `contradicts`, `reviews`, `related`.

---

### Search, stats, and sync

All require `vaults:read` + viewer.

**Search items:** `GET /vaults/:vaultId/search?q=&author=&year=&doi=&tag_id=&type=&page=&limit=`
- `limit` default 20, max 100
- `q` is case-insensitive substring match across title, abstract, authors

**Stats:** `GET /vaults/:vaultId/stats`
- Returns `{ item_count, tag_count, relation_count, last_updated }`

**Incremental sync:** `GET /vaults/:vaultId/changes?since=<ISO timestamp>`
- Returns all items where `updated_at > since`
- Use this instead of a full vault read when only detecting changes

---

### Export

Requires `vaults:export` + viewer.

`GET /vaults/:vaultId/export?format=json|bibtex`

Returns attachment-style serialized content. Preserve the raw payload; do not re-parse unless explicitly needed.

---

### Audit

Requires any valid API key (data route for vault-scoped; JWT for global).

- Vault-scoped: `GET /vaults/:vaultId/audit?since=&until=&limit=&page=`
- Global (JWT only): `GET /audit?since=&until=&limit=&page=`
- `limit` default 50, max 200

---

## Error handling

| Status | Code examples | Agent action |
|---|---|---|
| `401` | `missing_api_key`, `invalid_api_key_format`, `expired_api_key`, `revoked_api_key` | Stop. Report clearly. Do not retry with the same key. Ask user to provide a valid key. |
| `401` | `refhub_api_key_not_supported` | API key sent to a JWT-only route. Switch auth mode or stop. |
| `403` | `missing_scope` | Key lacks required scope. Report which scope is needed. Do not attempt workaround. |
| `403` | `vault_access_denied`, `vault_not_found` | No access to this vault with this key. Report and stop. |
| `404` | `item_not_found`, `tag_not_found`, `relation_not_found` | Resource doesn't exist. Verify the id. Do not create a replacement silently. |
| `409` | (DOI import, duplicate relation) | Resource already exists. Surface the existing item id to the user. |
| `413` | `request_too_large` | Payload too large. Split the request into smaller batches. |
| `429` | `rate_limit_exceeded` | Back off. Use `retry_after_seconds` from the response. |
| `500` | `bulk_insert_partial_failure`, `bulk_insert_failed` | Partial write may have occurred. Do not retry without `idempotency_key`. Alert user for manual review if partial failure. |
| `5xx` | `internal_error`, `service_error` | Transient. Retry once with backoff. If it persists, report and stop. |

Every error response includes `error.code`, `error.message`, and `meta.request_id`. Always surface `request_id` when reporting failures.

---

## Guardrails

- **Never infer `vault_id` from a vault name.** Always call `GET /vaults` and resolve it from the response.
- **Never mix JWT and API key auth.** Management routes reject API keys. Data routes do not accept JWTs.
- **Never assume a tag exists.** List tags before using `tag_ids` in any write.
- **Never create tags implicitly during item writes.** Tag creation is a separate step.
- **Never retry a bulk write after ambiguous failure** unless you have an `idempotency_key`.
- **Never assume frontend capability equals API support.** Features in the RefHub UI backed by direct Supabase access may not have a public API route.
- **Never proceed with vault or item deletion without explicit user confirmation.** Both are hard deletes with no undo.
- **Never send `visibility` or `public_slug` in `PATCH /vaults/:vaultId`.** Use the dedicated visibility endpoint.
- **`tag_ids` on item update is a full replacement, not an append.** Make this explicit to the user before patching.

---

## Scope boundary

The following are explicitly outside this skill:

- API key creation, rotation, or revocation (requires human session JWT)
- Google Drive link management (JWT-only management route)
- Semantic Scholar paper lookup, recommendations, references, citations (JWT-only)
- Vault archiving, soft-delete, or restore (not implemented in the API)
- Item revision history or restore (not implemented)
- Item move or copy between vaults (not implemented)
- Webhooks or event subscriptions (not implemented)
- Any direct Supabase access as a fallback

If a user requests one of these, state clearly that the current public API does not support it and do not improvise an alternative.

---

## References

- `docs/api-mapping.md` — full endpoint table with scopes, permissions, and constraints
- `docs/spec.md` — detailed per-workflow behavioral contracts and edge cases
