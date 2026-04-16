# RefHub Skill Workflow to API Mapping

## 1. Current public API surface (v2)

From `refhub-netlify` (`functions/api-v1.js` + `src/routes/`), the versioned API includes:

### Management routes (Supabase session JWT)

- `GET    /api/v1/keys`
- `POST   /api/v1/keys`
- `POST   /api/v1/keys/:keyId/revoke`
- `DELETE /api/v1/keys/:keyId`
- `GET    /api/v1/audit`                                    all audit logs for key owner
- `POST   /api/v1/recommendations`                         Semantic Scholar (JWT-only)
- `POST   /api/v1/references`                              Semantic Scholar (JWT-only)
- `POST   /api/v1/citations`                               Semantic Scholar (JWT-only)
- `POST   /api/v1/lookup`                                  Semantic Scholar (JWT-only)
- `GET/POST/DELETE /api/v1/google-drive`                   Drive link management (JWT-only)

### Data routes (RefHub API key)

**Vaults**
- `GET    /api/v1/vaults`
- `POST   /api/v1/vaults`
- `GET    /api/v1/vaults/:vaultId`
- `PATCH  /api/v1/vaults/:vaultId`
- `DELETE /api/v1/vaults/:vaultId`
- `PATCH  /api/v1/vaults/:vaultId/visibility`
- `GET    /api/v1/vaults/:vaultId/shares`
- `POST   /api/v1/vaults/:vaultId/shares`
- `PATCH  /api/v1/vaults/:vaultId/shares/:shareId`
- `DELETE /api/v1/vaults/:vaultId/shares/:shareId`

**Items**
- `GET    /api/v1/vaults/:vaultId/items`                    search/filter
- `POST   /api/v1/vaults/:vaultId/items`                    add items (legacy bulk add)
- `PATCH  /api/v1/vaults/:vaultId/items/:itemId`            update item
- `DELETE /api/v1/vaults/:vaultId/items/:itemId`            delete item (hard)
- `POST   /api/v1/vaults/:vaultId/items/upsert`             bulk upsert by DOI / title+year
- `POST   /api/v1/vaults/:vaultId/items/import-preview`     dry-run upsert

**Tags**
- `GET    /api/v1/vaults/:vaultId/tags`
- `POST   /api/v1/vaults/:vaultId/tags`
- `PATCH  /api/v1/vaults/:vaultId/tags/:tagId`
- `DELETE /api/v1/vaults/:vaultId/tags/:tagId`
- `POST   /api/v1/vaults/:vaultId/tags/attach`
- `POST   /api/v1/vaults/:vaultId/tags/detach`

**Relations**
- `GET    /api/v1/vaults/:vaultId/relations`
- `POST   /api/v1/vaults/:vaultId/relations`
- `PATCH  /api/v1/vaults/:vaultId/relations/:relationId`
- `DELETE /api/v1/vaults/:vaultId/relations/:relationId`

**Import**
- `POST   /api/v1/vaults/:vaultId/import/doi`
- `POST   /api/v1/vaults/:vaultId/import/bibtex`
- `POST   /api/v1/vaults/:vaultId/import/url`

**Search / stats / sync**
- `GET    /api/v1/vaults/:vaultId/search`
- `GET    /api/v1/vaults/:vaultId/stats`
- `GET    /api/v1/vaults/:vaultId/changes`

**Export**
- `GET    /api/v1/vaults/:vaultId/export?format=json|bibtex`

**Audit**
- `GET    /api/v1/vaults/:vaultId/audit`

**PDF** (used by browser extension, not typical agent workflows)
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf`
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf/session`
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf/complete`

## 2. Skill workflow to API mapping

| Skill workflow | Endpoint(s) | Required scope | Min vault permission |
| --- | --- | --- | --- |
| List accessible vaults | `GET /api/v1/vaults` | `vaults:read` | — |
| Read one vault with full contents | `GET /api/v1/vaults/:vaultId` | `vaults:read` | viewer |
| Create vault | `POST /api/v1/vaults` | `vaults:admin` | — (account-level) |
| Update vault metadata | `PATCH /api/v1/vaults/:vaultId` | `vaults:admin` | owner |
| Delete vault | `DELETE /api/v1/vaults/:vaultId` | `vaults:admin` | owner |
| Set vault visibility | `PATCH /api/v1/vaults/:vaultId/visibility` | `vaults:admin` | owner |
| List collaborators | `GET /api/v1/vaults/:vaultId/shares` | `vaults:read` | viewer |
| Add collaborator | `POST /api/v1/vaults/:vaultId/shares` | `vaults:admin` | owner |
| Update collaborator role | `PATCH /api/v1/vaults/:vaultId/shares/:shareId` | `vaults:admin` | owner |
| Remove collaborator | `DELETE /api/v1/vaults/:vaultId/shares/:shareId` | `vaults:admin` | owner |
| Add one or more items | `POST /api/v1/vaults/:vaultId/items` | `vaults:write` | editor |
| Update item | `PATCH /api/v1/vaults/:vaultId/items/:itemId` | `vaults:write` | editor |
| Delete item | `DELETE /api/v1/vaults/:vaultId/items/:itemId` | `vaults:write` | editor |
| Bulk upsert items | `POST /api/v1/vaults/:vaultId/items/upsert` | `vaults:write` | editor |
| Preview upsert (dry-run) | `POST /api/v1/vaults/:vaultId/items/import-preview` | `vaults:read` | viewer |
| Search/filter items | `GET /api/v1/vaults/:vaultId/search` or `items` | `vaults:read` | viewer |
| List tags | `GET /api/v1/vaults/:vaultId/tags` | `vaults:read` | viewer |
| Create tag | `POST /api/v1/vaults/:vaultId/tags` | `vaults:write` | editor |
| Update tag | `PATCH /api/v1/vaults/:vaultId/tags/:tagId` | `vaults:write` | editor |
| Delete tag | `DELETE /api/v1/vaults/:vaultId/tags/:tagId` | `vaults:write` | editor |
| Attach tags to item | `POST /api/v1/vaults/:vaultId/tags/attach` | `vaults:write` | editor |
| Detach tags from item | `POST /api/v1/vaults/:vaultId/tags/detach` | `vaults:write` | editor |
| List relations | `GET /api/v1/vaults/:vaultId/relations` | `vaults:read` | viewer |
| Create relation | `POST /api/v1/vaults/:vaultId/relations` | `vaults:write` | editor |
| Update relation type | `PATCH /api/v1/vaults/:vaultId/relations/:relationId` | `vaults:write` | editor |
| Delete relation | `DELETE /api/v1/vaults/:vaultId/relations/:relationId` | `vaults:write` | editor |
| Import from DOI | `POST /api/v1/vaults/:vaultId/import/doi` | `vaults:write` | editor |
| Import from BibTeX | `POST /api/v1/vaults/:vaultId/import/bibtex` | `vaults:write` | editor |
| Import from URL | `POST /api/v1/vaults/:vaultId/import/url` | `vaults:write` | editor |
| Vault stats | `GET /api/v1/vaults/:vaultId/stats` | `vaults:read` | viewer |
| Incremental sync (changes since) | `GET /api/v1/vaults/:vaultId/changes?since=` | `vaults:read` | viewer |
| Export vault | `GET /api/v1/vaults/:vaultId/export?format=` | `vaults:export` | viewer |
| Read vault audit log | `GET /api/v1/vaults/:vaultId/audit` | any API key | viewer |
| Read global audit log | `GET /api/v1/audit` | JWT (management) | — |

## 3. Workflows currently outside the public API surface

| Desired workflow | Status | Notes |
| --- | --- | --- |
| Vault archiving / unarchiving | not implemented | No API route; deferred |
| Vault duplication / clone | not implemented | No API route; deferred |
| Item soft-delete / restore | not implemented | Hard delete only |
| Item revision history | not implemented | No history table in current schema |
| Item move/copy between vaults | not implemented | No API route |
| Webhooks / event delivery | not implemented | Deferred to future cycle |
| Bulk relation import | not implemented | Create individually |
| Audit log viewer in frontend | not implemented | Tracked as frontend TODO |
| DOI enrichment from data routes | management-route only | `POST /api/v1/doi-metadata` exists but is JWT-only in the Semantic Scholar path; DOI *import* via `import/doi` is available |
| API key creation/revocation at runtime | JWT-only | Intentional: setup/admin flow |

## 4. Scope mapping (v2)

Current scopes:

- `vaults:read`
- `vaults:write`
- `vaults:export`
- `vaults:admin` ← new in v2

| Skill operation | Required scope |
| --- | --- |
| `vaults.list` | `vaults:read` |
| `vaults.get` | `vaults:read` |
| `vaults.create` | `vaults:admin` |
| `vaults.update` | `vaults:admin` |
| `vaults.delete` | `vaults:admin` |
| `vaults.setVisibility` | `vaults:admin` |
| `vaults.shares.*` | `vaults:admin` |
| `items.add` | `vaults:write` |
| `items.update` | `vaults:write` |
| `items.delete` | `vaults:write` |
| `items.upsert` | `vaults:write` |
| `items.importPreview` | `vaults:read` |
| `items.search` | `vaults:read` |
| `tags.list` | `vaults:read` |
| `tags.create` / `update` / `delete` | `vaults:write` |
| `tags.attach` / `detach` | `vaults:write` |
| `relations.list` | `vaults:read` |
| `relations.create` / `update` / `delete` | `vaults:write` |
| `import.doi` / `bibtex` / `url` | `vaults:write` |
| `vaults.stats` / `changes` / `search` | `vaults:read` |
| `vaults.export` | `vaults:export` |
| `audit.list` | any valid API key |

## 5. Important constraints inherited from the current API

- Vault delete and item delete are hard deletes — no undo, no restore.
- Bulk upsert matches on DOI first, then `bibtex_key`; items with neither are always created. Pass `idempotency_key` for safe retries (TTL: 5 minutes, in-memory).
- Tag creation is not implicit during item writes — tag IDs must already exist when passed as `tag_ids`.
- `tag_ids` on item update replaces the full tag set (not additive).
- Vault access is the combination of ownership, sharing, and optional API-key vault restrictions.
- Management routes and data routes use different auth models.
- `vaults:admin` scope must be explicitly requested at API key creation time.
- When adding a share to a `private` vault, the vault is automatically upgraded to `protected`.
- Valid share roles are `viewer` and `editor` only — `owner` cannot be set via the API.
- Share creation requires `email`; `user_id` is not accepted by the current implementation.
- All delete operations return `200 { data: { id } }` — not `204 No Content`.
- Relation creation is not idempotent — submitting a duplicate pair will produce an error; check before creating.
- Relations list only supports `?type=` filter — `source_id` and `target_id` filters are not implemented.
- DOI import calls Semantic Scholar internally; it will fail if `SEMANTIC_SCHOLAR_API_KEY` is not configured.

## 6. Practical implication for skill design

The skill covers the full current public API surface. Future expansion should follow the same pattern:

`API route exists → update skill → update CLI/MCP`

Deferred features (archiving, webhooks, revision history) should not be approximated via existing endpoints.
