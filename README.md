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

The [`refhub` CLI](https://github.com/refhub/refhub-cli) is the recommended execution layer. Install it from npm:

```sh
npm i -g @refhub/cli
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

## Installing in your agent

### Claude Code

```sh
# Register the marketplace (once)
claude plugin marketplace add refhub-io/refhub-marketplace

# Install the skill
claude plugin install refhub-skill@refhub-marketplace
```

The skill will be available in the next session. Claude Code automatically invokes it when you ask it to work with RefHub vaults or items.

### Gemini CLI

```sh
mkdir -p ~/.gemini/skills/refhub-skill
curl -o ~/.gemini/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

### OpenCode

```sh
mkdir -p ~/.config/opencode/skills/refhub-skill
curl -o ~/.config/opencode/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

Restart OpenCode to load the skill.

### Codex CLI

```sh
mkdir -p ~/.codex/skills/refhub-skill
curl -o ~/.codex/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

### Cursor, Windsurf, and others

For agents that support `AGENTS.md`, copy the included file to your project root:

```sh
curl -O https://raw.githubusercontent.com/refhub-io/refhub-skill/main/AGENTS.md
```

Or add it globally via your agent's rules/settings UI.

---

## Relationship to the broader stack

```
RefHub API  →  refhub-skill  →  refhub CLI  →  MCP server (planned)
```

The API is canonical. The skill adapts its surface into agent-friendly workflows without inventing behavior the API does not support.
