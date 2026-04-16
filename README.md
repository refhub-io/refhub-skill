# refhub-skill

An agent skill for operating [RefHub](https://refhub.io) through its public API.

Agents load `SKILL.md` to discover available workflows, then consult `docs/` for exact route contracts and behavioral rules. No implementation code lives here — the skill is the contract.

## What agents can do

With a scoped RefHub API key, an agent can:

- manage vaults (create, update, delete, visibility, collaborators)
- add, update, delete, search, and bulk-upsert items
- import references from a DOI, BibTeX string, or URL
- manage tags and relations as first-class objects
- sync incrementally via the changes feed
- export vaults as JSON or BibTeX
- read audit logs

## Structure

```
SKILL.md          ← skill entry point
docs/
  api-mapping.md  ← endpoint/scope/constraint reference
  spec.md         ← per-workflow behavioral contracts
```

## Auth

Agents use a pre-issued RefHub API key (`rhk_<publicId>_<secret>`). Key creation and revocation are human-managed through the RefHub UI or management API using a session JWT — not part of the agent runtime.

## Relationship to the broader stack

```
RefHub API  →  refhub-skill  →  CLI (planned)  →  MCP server (planned)
```

The API is canonical. The skill adapts its surface into agent-friendly workflows without inventing behavior the API does not support.
