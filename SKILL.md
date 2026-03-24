---
name: refhub-skill
description: Use when an agent needs to work with RefHub vaults through the public RefHub API using an existing API key. Supports listing accessible vaults, reading vault contents, adding items to a vault, updating an existing item, and exporting a vault. Also use when checking whether a requested RefHub workflow is actually supported by the current public API versus only existing in the frontend/product. Do not use for Supabase-direct product internals, vault/tag/relation/sharing mutations that lack public API routes, or API-key management flows that require a human session JWT.
---

# RefHub Skill

This skill is the agent-facing workflow layer over the RefHub public API.

Use it to do real work against the current API without pretending unsupported product capabilities exist.

## Use this skill for

- listing accessible vaults
- reading one vault with its items and related structures
- adding one or more items to a vault
- updating one existing vault item
- exporting a vault as JSON or BibTeX
- checking whether a requested workflow is supported by the public API yet

## Do not use this skill for

These are currently outside the public API surface or belong to a different auth path:

- creating or deleting vaults
- editing vault metadata
- creating, renaming, or deleting tags
- creating or deleting relations
- sharing / permission management / access requests
- DOI enrichment or Semantic Scholar lookups inside the skill
- direct Supabase access as a fallback
- API key creation or revocation during ordinary agent runtime

If the user asks for one of those, say the product may support it but the current public API does not cleanly expose it yet.

## Runtime assumptions

- Normal runtime auth uses a pre-issued RefHub API key in the form `rhk_<publicId>_<secret>`.
- API key management routes are separate and use a human session JWT; treat them as setup/admin flows, not the normal skill path.
- The API is canonical. Do not invent local-only behavior to emulate missing endpoints.

## Canonical v1 operations

Conceptual operation set:

- `vaults.list`
- `vaults.get`
- `items.add`
- `items.update`
- `vaults.export`

Current public API mapping:

- `GET /api/v1/vaults`
- `GET /api/v1/vaults/:vaultId`
- `POST /api/v1/vaults/:vaultId/items`
- `PATCH /api/v1/vaults/:vaultId/items/:itemId`
- `GET /api/v1/vaults/:vaultId/export?format=json|bibtex`

## Output expectations

Prefer structured, predictable results over chatty prose.

When possible, return:

- normalized vault/item summaries
- created or updated item IDs
- request IDs when available
- explicit partial-failure reporting
- clear error category: `auth_error`, `permission_error`, `input_error`, `not_found`, or `service_error`

## Guardrails

- Be explicit about scope failures (`vaults:read`, `vaults:write`, `vaults:export`).
- Be explicit about vault restriction failures on API keys.
- Treat unsupported workflows as unsupported, not as invitations to improvise.
- For bulk item creation, note that backend validation exists but true atomic guarantees may still depend on future transaction work.
- If `tag_ids` is present on item update, treat it as full replacement semantics.

## When to read the bundled docs

Read these only as needed:

- `docs/spec.md` — full workflow and behavior contract for the skill
- `docs/api-mapping.md` — exact mapping between skill workflows and current backend routes, plus unsupported areas
- `docs/roadmap.md` — staged evolution from skill to CLI/MCP and later API expansion

Recommended usage:

- For implementation or review of the skill contract: read `docs/spec.md`
- For “can the API do this yet?” questions: read `docs/api-mapping.md`
- For planning next capabilities: read `docs/roadmap.md`

## Strategic rule

RefHub should evolve in this order:

`API -> Skill -> CLI / MCP`

Keep the skill narrow, honest, and aligned to the current public API.
