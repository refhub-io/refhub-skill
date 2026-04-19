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

## Execution layer

The `refhub` CLI is the recommended execution layer. Install it from npm:

```sh
npm install -g refhub-cli
```

When available in the environment (`which refhub` succeeds), agents use it instead of making HTTP calls directly. The CLI handles authentication, error formatting, and consistent output.

```sh
# Auth: set once, used by all commands
export REFHUB_API_KEY=rhk_<publicId>_<secret>

# Or override per call
refhub vaults list --api-key rhk_...
```

Discovery: `refhub --help` or `refhub <group> --help` (e.g. `refhub vaults --help`).

Exit codes: `0` success · `1` API error · `2` bad arguments · `3` auth error.

Agents in environments without the CLI fall back to direct HTTP as documented in `docs/`.

## Relationship to the broader stack

```
RefHub API  →  refhub-skill  →  refhub CLI  →  MCP server (planned)
```

The API is canonical. The skill adapts its surface into agent-friendly workflows without inventing behavior the API does not support.
