---
name: refhub-skill
description: Use when an agent needs to work with RefHub vaults through the public RefHub API using an existing API key. Supports vault lifecycle (create, update, delete, visibility, shares), tag and relation CRUD, item add/update/delete/upsert, DOI/BibTeX/URL import, vault search and stats, audit log reads, and export. Also use when checking whether a requested RefHub workflow is supported by the current public API. Do not use for Supabase-direct product internals, API-key management flows requiring a human session JWT, or features explicitly noted as outside the API surface.
---

# RefHub Skill

This skill is the agent-facing workflow layer over the RefHub public API (v2).

Use it to do real work against the current API without pretending unsupported product capabilities exist.

## Use this skill for

**Vault lifecycle** (requires `vaults:admin` scope)
- creating, updating metadata, and deleting vaults
- setting vault visibility (`private` / `protected` / `public`)
- listing, adding, updating, and removing vault collaborators (shares)

**Item operations** (requires `vaults:write` for writes, `vaults:read` for reads)
- listing and searching items in a vault
- adding one or more items to a vault
- updating an existing vault item
- deleting an item from a vault
- bulk-upserting items by DOI or title+year (with idempotency key)
- previewing what a bulk upsert would do without committing (dry-run)

**Tag operations** (requires `vaults:write` for writes, `vaults:read` for reads)
- listing, creating, updating, and deleting vault-scoped tags
- attaching and detaching tags from items

**Relation operations** (requires `vaults:write` for writes, `vaults:read` for reads)
- listing, creating, updating, and deleting relations between items

**Import** (requires `vaults:write`)
- importing an item from a DOI (Semantic Scholar enrichment)
- importing items from a BibTeX string
- importing an item from a URL (Open Graph metadata)

**Discovery and sync**
- listing all accessible vaults
- reading one vault with full contents (items, tags, relations)
- searching/filtering items within a vault
- fetching vault stats (item/tag/relation counts, last updated)
- fetching items changed since a timestamp (incremental sync)

**Export** (requires `vaults:export`)
- exporting a vault as JSON or BibTeX

**Audit** (any valid API key)
- reading audit logs for all requests by this key's owner
- reading audit logs scoped to a specific vault

## Do not use this skill for

These belong to a different auth path or are explicitly outside the public API surface:

- API key creation or revocation (requires a human session JWT — setup/admin flow)
- direct Supabase access as a fallback
- Google Drive link management (management route, JWT-only)
- Semantic Scholar paper lookup / recommendations (management route, JWT-only)
- vault archiving, soft-delete, or item revision history (not yet implemented)
- webhooks or event subscriptions (not yet implemented)

If the user asks for one of those, explain the current limitation clearly.

## Runtime assumptions

- Normal runtime auth uses a pre-issued RefHub API key: `rhk_<publicId>_<secret>`.
- API key management routes use a human session JWT; treat them as setup/admin, not the normal skill path.
- The API is canonical. Do not invent local-only behavior to emulate missing endpoints.
- Keys may carry any combination of `vaults:read`, `vaults:write`, `vaults:export`, `vaults:admin`.
- Keys may be restricted to specific vault IDs at creation time.

## Canonical v2 operations and routes

### Data routes (RefHub API key auth)

**Vaults**
- `GET    /api/v1/vaults`                                   list accessible vaults (`vaults:read`)
- `POST   /api/v1/vaults`                                   create vault (`vaults:admin`)
- `GET    /api/v1/vaults/:vaultId`                          read vault with full contents (`vaults:read`)
- `PATCH  /api/v1/vaults/:vaultId`                          update vault metadata (`vaults:admin`, owner)
- `DELETE /api/v1/vaults/:vaultId`                          delete vault — hard delete (`vaults:admin`, owner)
- `PATCH  /api/v1/vaults/:vaultId/visibility`               set visibility (`vaults:admin`, owner)
- `GET    /api/v1/vaults/:vaultId/shares`                   list collaborators (`vaults:admin`, owner)
- `POST   /api/v1/vaults/:vaultId/shares`                   add collaborator (`vaults:admin`, owner)
- `PATCH  /api/v1/vaults/:vaultId/shares/:shareId`          update collaborator role (`vaults:admin`, owner)
- `DELETE /api/v1/vaults/:vaultId/shares/:shareId`          remove collaborator (`vaults:admin`, owner)

**Items**
- `POST   /api/v1/vaults/:vaultId/items`                    add one or more items (`vaults:write`, editor)
- `PATCH  /api/v1/vaults/:vaultId/items/:itemId`            update item (`vaults:write`, editor)
- `DELETE /api/v1/vaults/:vaultId/items/:itemId`            delete item — hard delete (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/items/upsert`             bulk upsert by DOI or title+year (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/items/import-preview`     dry-run upsert — writes nothing (`vaults:read`)
- `GET    /api/v1/vaults/:vaultId/items`                    search/filter items (`vaults:read`, viewer)

**Tags**
- `GET    /api/v1/vaults/:vaultId/tags`                     list tags (`vaults:read`, viewer)
- `POST   /api/v1/vaults/:vaultId/tags`                     create tag (`vaults:write`, editor)
- `PATCH  /api/v1/vaults/:vaultId/tags/:tagId`              update tag (`vaults:write`, editor)
- `DELETE /api/v1/vaults/:vaultId/tags/:tagId`              delete tag (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/tags/attach`              attach tags to item (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/tags/detach`              detach tags from item (`vaults:write`, editor)

**Relations**
- `GET    /api/v1/vaults/:vaultId/relations`                list relations (`vaults:read`, viewer)
- `POST   /api/v1/vaults/:vaultId/relations`                create relation (`vaults:write`, editor)
- `PATCH  /api/v1/vaults/:vaultId/relations/:relationId`    update relation type (`vaults:write`, editor)
- `DELETE /api/v1/vaults/:vaultId/relations/:relationId`    delete relation (`vaults:write`, editor)

**Import**
- `POST   /api/v1/vaults/:vaultId/import/doi`               import from DOI (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/import/bibtex`            import from BibTeX (`vaults:write`, editor)
- `POST   /api/v1/vaults/:vaultId/import/url`               import from URL (`vaults:write`, editor)

**Search / stats / sync**
- `GET    /api/v1/vaults/:vaultId/search`                   search items (`vaults:read`, viewer)
- `GET    /api/v1/vaults/:vaultId/stats`                    counts + last updated (`vaults:read`, viewer)
- `GET    /api/v1/vaults/:vaultId/changes`                  items changed since timestamp (`vaults:read`, viewer)

**Export**
- `GET    /api/v1/vaults/:vaultId/export?format=json|bibtex` export vault (`vaults:export`, viewer)

**Audit**
- `GET    /api/v1/vaults/:vaultId/audit`                    vault-scoped audit log (any valid API key)

### Management routes (Supabase session JWT)

- `GET    /api/v1/audit`                                    all audit logs for key owner (any valid key)
- `GET /api/v1/keys`, `POST /api/v1/keys`, `DELETE /api/v1/keys/:keyId`  API key management (JWT only)

## Output expectations

Prefer structured, predictable results over chatty prose.

When possible, return:

- normalized vault/item summaries
- created or updated item IDs
- request IDs when available
- explicit partial-failure reporting
- clear error category: `auth_error`, `permission_error`, `input_error`, `not_found`, or `service_error`

## Guardrails

- Be explicit about scope failures (`vaults:read`, `vaults:write`, `vaults:export`, `vaults:admin`).
- Be explicit about vault restriction failures on API keys.
- `vaults:admin` + owner vault permission required for vault create/delete/visibility/shares — do not attempt with lower scopes.
- Treat unsupported workflows as unsupported, not as invitations to improvise.
- Bulk upsert uses DOI-first then title+year matching; pass an `idempotency_key` for safe retries.
- If `tag_ids` is present on item update or attach, treat it as full replacement semantics.
- Vault delete and item delete are hard deletes — no undo. Warn before proceeding.

## When to read the bundled docs

Read these only as needed:

- `docs/spec.md` — full workflow and behavior contract; use when you need detailed per-operation rules or edge cases
- `docs/api-mapping.md` — exact endpoint-to-workflow table, scope requirements, and unsupported areas; use for “can the API do this?” questions

## Strategic rule

RefHub should evolve in this order:

`API -> Skill -> CLI / MCP`

Keep the skill honest and aligned to the current public API.
