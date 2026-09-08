# RefHub Skill Workflow to API Mapping

## 1. Current public API surface (v2)

From `refhub-netlify` (`functions/api-v1.js` + `src/routes/`), the versioned API includes:

### Management routes (Supabase session JWT)

- `GET    /api/v1/keys`
- `POST   /api/v1/keys`
- `POST   /api/v1/keys/:keyId/revoke`
- `DELETE /api/v1/keys/:keyId`
- `GET    /api/v1/audit`                                    all audit logs for key owner
- `POST   /api/v1/recommendations|references|citations|lookup|doi-metadata|search` legacy frontend Semantic Scholar routes (JWT-only; agents use `/semantic-scholar/*`)
- `GET/POST/DELETE /api/v1/google-drive`                   Drive link management (JWT-only)
- `POST   /api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf` browser/session item PDF upload — JSON `source_url` only, no raw bytes (JWT-only)
- `POST   /api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf/session|complete` browser/session resumable item PDF upload (JWT-only)

### Data routes (RefHub API key)

**Vaults**
- `GET    /api/v1/vaults`
- `POST   /api/v1/vaults`
- `GET    /api/v1/vaults/:vaultId`
- `PATCH  /api/v1/vaults/:vaultId`
- `DELETE /api/v1/vaults/:vaultId`
- `POST   /api/v1/vaults/:vaultId/archive`
- `PATCH  /api/v1/vaults/:vaultId/visibility`
- `GET    /api/v1/vaults/:vaultId/shares`
- `POST   /api/v1/vaults/:vaultId/shares`
- `PATCH  /api/v1/vaults/:vaultId/shares/:shareId`
- `DELETE /api/v1/vaults/:vaultId/shares/:shareId`

**Items**
- `GET    /api/v1/vaults/:vaultId/items`                    search/filter
- `POST   /api/v1/vaults/:vaultId/items`                    add items (legacy bulk add)
- `PATCH  /api/v1/vaults/:vaultId/items/:itemId`            update item; `section_id`/`section_position`/`featured`/`featured_note` require vault owner permission specifically (see §5)
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

**Sections**
- `GET    /api/v1/vaults/:vaultId/sections`
- `POST   /api/v1/vaults/:vaultId/sections`                 owner-only, despite `vaults:admin` scope covering most other admin ops
- `PATCH  /api/v1/vaults/:vaultId/sections/:sectionId`      owner-only
- `DELETE /api/v1/vaults/:vaultId/sections/:sectionId`      owner-only

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

**PDF** (API-key item routes for agents/CLI)
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf`
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf/session`
- `POST   /api/v1/vaults/:vaultId/items/:itemId/pdf/complete`
- `POST   /api/v1/publications/:publicationId/pdf/session|complete` — publication-level (no vault), same resumable-only flow, requires `vaults:write` via API key like the routes above; not JWT-only despite living outside `/vaults/*`. No CLI command wraps it yet.

The CLI always uses the API-key `/session` + `/complete` routes, at any file size — there is no raw-bytes upload path. `POST /api/v1/vaults/:vaultId/items/:itemId/pdf` itself only accepts a JSON `{ source_url }` body (server-side fetch); a raw PDF body there returns `410 raw_pdf_upload_removed`. The returned `upload_url` is a Google Drive resumable URL; clients upload bytes directly to Drive before completing the item asset through the API-key route. Browser/session resumable uploads stay under `/google-drive/...`.

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
| Scan for citation-based relationship suggestions | `POST /api/v1/semantic-scholar/lookup`+`/references`+`/citations` then `POST /api/v1/vaults/:vaultId/relations` (client-side orchestration, no dedicated route) | `vaults:read` + `vaults:write` | editor |
| List sections | `GET /api/v1/vaults/:vaultId/sections` | `vaults:read` | viewer |
| Create section | `POST /api/v1/vaults/:vaultId/sections` | `vaults:admin` | owner |
| Update section | `PATCH /api/v1/vaults/:vaultId/sections/:sectionId` | `vaults:admin` | owner |
| Delete section | `DELETE /api/v1/vaults/:vaultId/sections/:sectionId` | `vaults:admin` | owner |
| Set item section/featured state | `PATCH /api/v1/vaults/:vaultId/items/:itemId` (`section_id`/`section_position`/`featured`/`featured_note`) | `vaults:write` | owner (not editor, unlike the rest of this endpoint) |
| Import from DOI | `POST /api/v1/vaults/:vaultId/import/doi` | `vaults:write` | editor |
| Import from BibTeX | `POST /api/v1/vaults/:vaultId/import/bibtex` | `vaults:write` | editor |
| Import from URL | `POST /api/v1/vaults/:vaultId/import/url` | `vaults:write` | editor |
| Vault stats | `GET /api/v1/vaults/:vaultId/stats` | `vaults:read` | viewer |
| Incremental sync (changes since) | `GET /api/v1/vaults/:vaultId/changes?since=` | `vaults:read` | viewer |
| Export vault | `GET /api/v1/vaults/:vaultId/export?format=` | `vaults:export` | viewer |
| Read vault audit log | `GET /api/v1/vaults/:vaultId/audit` | any API key | viewer |
| Read global audit log | `GET /api/v1/audit` | JWT (management) | — |
| Enrich publication metadata from Semantic Scholar | `POST /api/v1/semantic-scholar/doi-metadata` then `PATCH /api/v1/vaults/:vaultId/items/:itemId` | `vaults:read` + `vaults:write` | editor |
| Look up Semantic Scholar paper id | `POST /api/v1/semantic-scholar/lookup` | `vaults:read` | — |
| Get paper recommendations | `POST /api/v1/semantic-scholar/recommendations` or `/related` | `vaults:read` | — |
| Get paper references | `POST /api/v1/semantic-scholar/references` | `vaults:read` | — |
| Get paper citations | `POST /api/v1/semantic-scholar/citations` or `/cited-by` | `vaults:read` | — |
| Upload PDF to Google Drive for a vault item | `POST /api/v1/vaults/:vaultId/items/:itemId/pdf`, `/pdf/session`, `/pdf/complete` | `vaults:write` | editor |

## 3. Workflows not yet implemented

| Desired workflow | Status | Notes |
| --- | --- | --- |
| Vault unarchiving | never — by design | Not deferred; irreversibility is enforced server-side (not an app-level policy). Archiving itself IS implemented: `POST /api/v1/vaults/:vaultId/archive` |
| Vault duplication / clone | not implemented | No API route; deferred |
| Item soft-delete / restore | not implemented | Hard delete only |
| Item revision history | not implemented | No history table in current schema |
| Item move/copy between vaults | not implemented | No API route |
| Webhooks / event delivery | not implemented | Deferred to future cycle |
| Bulk relation import | not implemented | Create individually |

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
| `relations.scanSuggestions` | `vaults:read` + `vaults:write` |
| `sections.list` | `vaults:read` |
| `sections.create` / `update` / `delete` | `vaults:admin` (+ vault owner) |
| `import.doi` / `bibtex` / `url` | `vaults:write` |
| `vaults.stats` / `changes` / `search` | `vaults:read` |
| `vaults.export` | `vaults:export` |
| `audit.list` (vault-scoped) | any valid API key |
| `audit.list` (global) | JWT (management) |
| `enrichment.doiMetadata` | `vaults:read` |
| `enrichment.enrichVault` / `enrichment.enrichItem` | `vaults:read` + `vaults:write` |
| `items.uploadPdf` | `vaults:write` |
| `scholar.lookup` / `search` / `recommendations` / `references` / `citations` | `vaults:read` |

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
- Relationship-suggestion scanning only ever creates `relation_type: "cites"` — citation-graph data alone gives no basis for `extends`/`contradicts`/etc.
- Section and item-featured writes require vault **owner** permission, not just editor — this is stricter than every other `vaults:write`/`vaults:admin` operation on the same endpoints, which check editor or owner respectively. `PATCH /vaults/:vaultId/items/:itemId` specifically runs a second, owner-level access check only when the body includes `section_id`/`section_position`/`featured`/`featured_note`.
- Deleting a section unfiles its items (`section_id` → `null`) rather than deleting them.
- DOI import calls Semantic Scholar internally; it will fail if `SEMANTIC_SCHOLAR_API_KEY` is not configured.
- Agent Semantic Scholar routes live under `/semantic-scholar/*` and accept API keys with `vaults:read`; legacy root routes remain JWT-only for frontend compatibility.
- Semantic Scholar rate limit: 1 request per second. The enrichment workflow must sleep between calls when processing multiple items.
- Item PDF upload requires `vaults:write` and Google Drive linked to the account through the web UI. Uploading bytes always uses the resumable flow — API-key `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/session`, direct Drive `PUT` to the returned `upload_url`, then `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/complete` — at any file size; there is no raw-bytes upload path. `POST /api/v1/vaults/:vaultId/items/:itemId/pdf` itself only accepts a JSON `{ source_url }` body (server-side fetch); a raw PDF body there returns `410 raw_pdf_upload_removed`. Returns `503 drive_not_linked` if Drive has not been connected.
- Google Drive link management (`GET/POST/DELETE /google-drive`) is a separate JWT management workflow outside the normal agent-facing skill surface.
- Browser/session JWT item PDF routes are under `/api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf`, `/session`, and `/complete`; API-key agents must not call those `/google-drive/...` routes.
- Google Drive resumable session creation forwards only validated browser `Origin` values. Defaults include `https://refhub.io`, `http://localhost:3000`, `http://localhost:5173`, and `http://localhost:8081`; explicit `REFHUB_API_ALLOWED_ORIGINS` overrides must include the active frontend/dev origin.
- The Drive-hosted PDF's URL is readable back after upload as `drive_pdf_url` on item reads (its *content*/bytes are not — that still requires a Drive-scoped OAuth token this skill doesn't have) — see `docs/spec.md` §7.27 for the full read/access contract for both `pdf_url` (publisher PDF) and the Drive-hosted copy.

## 6. Practical implication for skill design

The skill covers the full current public API surface. Future expansion should follow the same pattern:

`API route exists → update skill → update CLI/MCP`

Deferred features (vault duplication, webhooks, revision history) should not be approximated via existing endpoints. Vault unarchiving is not deferred — it will never exist.


## API-key Semantic Scholar and PDF workflows (2026-06)

Normal agent runtime is API-key-only:

- Semantic Scholar: `POST /api/v1/semantic-scholar/lookup`, `/doi-metadata`, `/search`, `/recommendations`, `/related`, `/references`, `/citations`, `/cited-by`; all require `vaults:read`. CLI: `refhub discover ...`, `refhub enrich --vault <id> [--item <id>] [--dry-run]`, and `refhub relations scan --vault <id> [--item <id>] [--dry-run] [--limit <n>]` (the latter two are client-side orchestration on these routes, no dedicated backend route of their own).
- Item PDF upload requires `vaults:write` and a Google Drive account already linked in the RefHub web UI. CLI: `refhub pdf upload --vault <vaultId> --item <itemId> --file <path.pdf>`.
- All API-key item PDF uploads use the resumable flow: `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/session`, direct `PUT` of the PDF bytes to the returned Google Drive `upload_url`, then `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/complete` — at any file size. `POST /api/v1/vaults/:vaultId/items/:itemId/pdf` itself only accepts a JSON `{ source_url }` body now; raw `application/pdf` bytes there return `410 raw_pdf_upload_removed`.
- Browser/session JWT item PDF routes live under `/api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf`, `/session`, and `/complete`. API-key agents must not call those `/google-drive/...` routes.
- Publication-level PDF upload (`POST /publications/:publicationId/pdf/session` + `/complete`, same resumable-only flow, no raw-bytes variant) also just requires `vaults:write` via API key — not a JWT-only route. No CLI command wraps it yet.
- Google Drive connect/disconnect, API-key lifecycle, and global audit remain session-JWT/browser account-management flows.
- `publication_pdf_assets` canonical-row delete/insert behavior is an internal frontend/schema detail; do not depend on PostgREST upsert semantics for that table from agents.
- Search/list accepts canonical `per_page` and `tag`; backend also accepts compatibility aliases `limit` and `tag_id`. DOI filtering is supported.
