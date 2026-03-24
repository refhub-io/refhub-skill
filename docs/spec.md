# RefHub Skill Spec

## 1. Purpose

`refhub-skill` is an agent-facing adapter for RefHub.

Its job is to let an agent reliably do useful reference-management work without exposing raw backend complexity or pretending unsupported operations exist.

The skill should be optimized for:

- reading vaults for analysis and synthesis
- adding or updating references in a target vault
- exporting vault contents for downstream tools
- working safely with scoped API keys

It should not become a second backend, an alternate database contract, or a thin wrapper around every future endpoint.

## 2. Design principles

### 2.1 API is canonical

The RefHub API defines the source of truth for what the skill can do over the network.

### 2.2 Workflow-first, not endpoint-first

The skill should expose tasks like:

- list accessible vaults
- inspect a vault
- add references to a vault
- update notes or metadata on an existing vault item
- export a vault for local processing

not generic transport verbs.

### 2.3 Honest capability boundaries

If the product supports something via direct Supabase/frontend code but the public API does not yet support it, the skill must report that explicitly and fail clearly.

### 2.4 Stable outputs over cleverness

The skill should return structured results that downstream agents can rely on:

- normalized vault summaries
- normalized item summaries
- request IDs when available
- explicit partial-failure reporting

### 2.5 No fake orchestration

No placeholder commands, no fake tool handlers, no pretend offline sync layer.

## 3. Core domain model

Grounded in the current frontend schema and backend handler:

- `vault`
  - collection boundary for references and collaboration
  - key fields include `id`, `name`, `description`, `visibility`, `category`, `abstract`, `updated_at`
- `vault item`
  - agent-facing term for a `vault_publications` row
  - carries title, authors, year, DOI, URL, abstract, notes, BibTeX-oriented metadata, version
- `canonical publication`
  - underlying `publications` row created alongside a vault item
- `tag`
  - currently vault-scoped for shared-vault workflows
- `publication tag`
  - joins tag IDs to vault items
- `publication relation`
  - connects vault items, for example `cites`, `extends`, `builds_on`, `contradicts`, `reviews`
- `api key`
  - scoped bearer credential for API access

## 4. Actor model

### 4.1 Human owner

Creates and manages RefHub API keys through authenticated management routes.

### 4.2 Agent using the skill

Uses an existing API key to perform constrained RefHub workflows.

### 4.3 RefHub API

Authoritative service for vault access, item writes, and export.

## 5. v1 skill goals

The first skill release should support only workflows already backed by the current public API:

1. Discover accessible vaults.
2. Read one vault and its contents.
3. Add one or more items to a vault.
4. Update one existing vault item.
5. Export a vault as JSON or BibTeX.
6. Provide enough response structure for downstream summarization, note generation, or local transformation.

## 6. Non-goals for v1

Do not include these in the first implementation unless the API lands first:

- vault creation
- vault deletion
- tag creation or tag editing
- relation creation or deletion
- sharing and permission management
- favorites or forks
- access-request approval flows
- direct DOI lookup or Semantic Scholar enrichment inside the skill
- broad search across RefHub content
- local database sync

Those are real product capabilities or likely future capabilities, but they are not currently clean skill targets through the public API.

## 7. Primary workflows

### 7.1 List vaults

Intent:

- let an agent discover where it can act

Expected behavior:

- returns all accessible vaults allowed by the key's scopes and optional vault restrictions
- includes permission level and item count

Minimum output shape:

```json
{
  "vaults": [
    {
      "id": "uuid",
      "name": "AI Reading List",
      "visibility": "private",
      "permission": "owner",
      "item_count": 12,
      "updated_at": "2026-03-23T18:00:00Z"
    }
  ],
  "request_id": "uuid"
}
```

### 7.2 Read vault

Intent:

- load enough structured content for analysis, summarization, curation, or export preparation

Expected behavior:

- returns vault metadata
- returns vault items
- returns vault tags
- returns item-tag links
- returns relations for the returned items

Skill expectation:

- preserve raw API fields
- also provide an agent-friendly summary count block

### 7.3 Add items to a vault

Intent:

- import references prepared elsewhere

Expected behavior:

- accepts one or more items
- requires title per item
- allows existing vault tag IDs only
- rejects oversized batches or unknown tag IDs

Skill expectation:

- pre-validate obvious shape errors before sending
- return created vault item IDs
- treat a backend rollback failure as a high-severity error

### 7.4 Update one vault item

Intent:

- revise notes, metadata, or tags on an existing reference

Expected behavior:

- partial update of a vault item
- if `tag_ids` is present, it replaces the full tag set
- increments the version field on successful metadata updates

Skill expectation:

- make tag replacement semantics explicit
- distinguish `item not found` from `permission denied`

### 7.5 Export vault

Intent:

- hand off a vault to local scripts, citation tooling, or another agent stage

Expected behavior:

- supports `json` and `bibtex`
- returns attachment-like serialized output

Skill expectation:

- expose export format choice explicitly
- preserve the raw serialized payload
- optionally attach lightweight metadata such as byte length and request ID

## 8. Failure behavior

The skill should normalize failures into a predictable structure.

Categories:

- `auth_error`
  - missing API key, invalid key format, expired key, revoked key
- `permission_error`
  - missing scope, vault access denied, write access denied
- `input_error`
  - invalid JSON, invalid item payload, invalid tag IDs, unsupported export format
- `not_found`
  - vault not found, item not found, route not found
- `service_error`
  - internal backend error, rollback failure, transient upstream issue

Every surfaced error should keep:

- HTTP status when available
- RefHub error code when available
- human-readable message
- `request_id` when available

## 9. Authentication assumptions

The skill should assume:

- API key creation and revocation are management concerns outside normal agent runtime
- normal skill execution uses a pre-issued `rhk_<publicId>_<secret>` key
- keys may be scope-limited and vault-limited

The skill should not require a Supabase session JWT for ordinary use.

## 10. Skill interface shape

The implementation can choose its manifest format later, but the conceptual operations should be:

- `vaults.list`
- `vaults.get`
- `items.add`
- `items.update`
- `vaults.export`

These names are stable enough for the spec and can later be mapped onto CLI verbs and MCP tools.

## 11. Data handling rules

- Treat RefHub as the system of record.
- Avoid hidden local state beyond transient request execution.
- Never infer missing IDs when the API requires explicit IDs.
- Do not silently create tags.
- Do not silently re-run bulk writes after ambiguous failure.
- Prefer idempotent read flows and explicit write confirmations.

## 12. What exists now vs what is proposed

### Exists now

- API key management routes
- vault list/read routes
- item add/update routes
- JSON and BibTeX export routes

### Exists in product, but not in public API contract yet

- vault creation and editing
- richer tag management
- publication relation management
- sharing workflows
- favorites and forks
- DOI-assisted import flows in the frontend

### Proposed next

- add API coverage for high-value missing workflows before expanding the skill surface

## 13. Acceptance criteria for first implementation

The first implementation is acceptable when:

- every v1 workflow maps directly to a current public API route
- unsupported workflows fail clearly with documented reasons
- outputs are structured enough for downstream agents
- the implementation does not require direct Supabase access
- the same operation names can be mirrored later by a CLI and MCP server
