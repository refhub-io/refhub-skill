# RefHub Skill Workflow to API Mapping

## 1. Current public API surface

From `refhub-netlify`, the current versioned API includes:

### Management routes using Supabase session JWT

- `GET /api/v1/keys`
- `POST /api/v1/keys`
- `POST /api/v1/keys/:keyId/revoke`
- `DELETE /api/v1/keys/:keyId`

### Data routes using RefHub API key

- `GET /api/v1/vaults`
- `GET /api/v1/vaults/:vaultId`
- `POST /api/v1/vaults/:vaultId/items`
- `PATCH /api/v1/vaults/:vaultId/items/:itemId`
- `GET /api/v1/vaults/:vaultId/export?format=json|bibtex`

## 2. Recommended v1 skill workflows

| Skill workflow | Current endpoint(s) | Status | Notes |
| --- | --- | --- | --- |
| List accessible vaults | `GET /api/v1/vaults` | supported now | Requires `vaults:read` |
| Read one vault with contents | `GET /api/v1/vaults/:vaultId` | supported now | Returns vault, publications, tags, publication tags, relations |
| Add one or more items | `POST /api/v1/vaults/:vaultId/items` | supported now | Requires `vaults:write`; tag IDs must already exist |
| Update one item | `PATCH /api/v1/vaults/:vaultId/items/:itemId` | supported now | Partial update; `tag_ids` replaces full tag set |
| Export one vault | `GET /api/v1/vaults/:vaultId/export` | supported now | Requires `vaults:export`; supports `json` and `bibtex` |

## 3. Workflows that should stay out of v1

| Desired workflow | Product evidence | Current API status | Recommendation |
| --- | --- | --- | --- |
| Create vault | Frontend uses direct Supabase access to `vaults` | missing | Add API endpoint first |
| Edit vault metadata | Frontend supports vault editing | missing | Add API endpoint first |
| Create tag | Frontend can insert into `tags` directly | missing | Add API endpoint first |
| Update/delete tag | Frontend has shared-vault tag operations | missing | Add API endpoint first |
| Create relation between items | Frontend uses `publication_relations` directly | missing | Add API endpoint first |
| Share vault / change permissions | Frontend uses `vault_shares` and access requests | missing | Add API endpoint first |
| Approve access requests | Product tables exist | missing | Add API endpoint first |
| Fork/favorite vault | Product tables exist | missing | Add API endpoint first |
| DOI lookup / metadata enrichment | Frontend integrates DOI and Semantic Scholar helpers | not in API | Keep outside the skill until intentionally designed |

## 4. Suggested next API additions after v1

These are the highest-value additions for making the skill materially more useful.

### Tier A

- `POST /api/v1/vaults`
  - create vault
- `PATCH /api/v1/vaults/:vaultId`
  - edit vault metadata
- `POST /api/v1/vaults/:vaultId/tags`
  - create a vault-scoped tag
- `PATCH /api/v1/vaults/:vaultId/tags/:tagId`
  - rename or move a tag

### Tier B

- `POST /api/v1/vaults/:vaultId/relations`
  - create relation between vault items
- `DELETE /api/v1/vaults/:vaultId/relations/:relationId`
  - remove relation
- `GET /api/v1/vaults/:vaultId/items/:itemId`
  - fetch one item directly

### Tier C

- `POST /api/v1/vaults/:vaultId/shares`
- `PATCH /api/v1/vaults/:vaultId/shares/:shareId`
- `GET /api/v1/vaults/:vaultId/access-requests`
- `POST /api/v1/vaults/:vaultId/access-requests/:requestId/approve`

## 5. Scope mapping

Current scopes are:

- `vaults:read`
- `vaults:write`
- `vaults:export`

Recommended skill operation requirements:

| Skill operation | Required scope |
| --- | --- |
| `vaults.list` | `vaults:read` |
| `vaults.get` | `vaults:read` |
| `items.add` | `vaults:write` |
| `items.update` | `vaults:write` |
| `vaults.export` | `vaults:export` |

## 6. Important constraints inherited from the current API

- No vault-creation endpoint exists.
- Bulk item creation is validated ahead of insertion but true atomicity still depends on future transaction/RPC work.
- Tag creation is not implicit during item writes.
- Vault access is the combination of ownership, sharing, and optional API-key vault restrictions.
- Management routes and data routes use different auth models.

## 7. Practical implication for skill design

The skill should be implemented as a narrow, trustworthy adapter over the current routes, then expanded only when the canonical API gains stable support for the next workflow tier.
