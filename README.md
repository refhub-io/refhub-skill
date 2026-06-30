# refhub-skill

> // agent_skill for [refhub.io](https://refhub.io)

agent skill for operating refhub through its public api (v2). agents load `SKILL.md` to discover workflows and consult `docs/` for exact route contracts and behavioral rules. no implementation code — the skill is the contract.

---

## // capabilities

with a scoped refhub api key for normal runtime work, an agent can:

- manage vaults (create, update, delete, visibility, collaborators)
- add, update, delete, search, and bulk-upsert papers
- import references from a doi, bibtex string, or url
- manage tags and relations as first-class objects
- sync incrementally via the changes feed
- export vaults as json or bibtex
- read audit logs
- enrich incomplete publication metadata from semantic scholar (api key)
- upload pdfs to google drive and link them to vault items (api key after Drive is connected)

---

## // structure

```
SKILL.md          ← skill entry point
AGENTS.md         ← instructions for cursor, windsurf, codex, and others
docs/
  api-mapping.md  ← endpoint/scope/constraint reference
  spec.md         ← per-workflow behavioral contracts
```

---

## // auth

two modes — use the right one for the right route:

**api key** — all data routes (vaults, items, tags, relations, import, export, audit):
```
REFHUB_API_KEY=rhk_<publicId>_<secret>
```

Keep this in your shell environment or a local env file. Do not paste live keys into Claude, Codex, or plugin chats.

**session jwt** — setup/admin routes only (google drive link management, key management, global audit). These are normally handled through the RefHub UI; do not require a session JWT for ordinary CLI agent workflows.

---

## // execution layer

the [`refhub` cli](https://github.com/refhub/refhub-cli) is the recommended execution layer:

```sh
npm i -g @refhub/cli
```

when available, agents use it instead of making http calls directly. the cli handles authentication, error formatting, and consistent output.

```sh
export REFHUB_API_KEY=rhk_<publicId>_<secret>   # data routes
refhub vaults list
refhub enrich --vault <id>                       # semantic scholar enrichment
refhub pdf upload --vault <id> --item <id> --file <f>  # pdf → google drive
refhub --help               # discover all commands
```

exit codes: `0` success · `1` api error · `2` bad arguments · `3` auth error.

the cli covers all data routes plus `enrich` and `pdf upload`. for other management routes (google drive, key management) fall back to direct http as documented in `docs/`.

---

## // install

### claude code

```sh
claude plugin marketplace add \
  https://github.com/refhub-io/refhub-claude
claude plugin install refhub-skill@refhub-claude
```

available in the next session. automatically invoked when you ask claude to work with refhub vaults or papers.

Why this form: `claude plugin marketplace add refhub-io/refhub-claude` uses GitHub shorthand and Claude clones it over SSH (`git@github.com:...`), which fails on machines without a GitHub SSH key. The HTTPS repo URL avoids that while keeping the normal Claude marketplace flow.

### gemini cli

```sh
mkdir -p ~/.gemini/skills/refhub-skill
curl -o ~/.gemini/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

### opencode

```sh
mkdir -p ~/.config/opencode/skills/refhub-skill
curl -o ~/.config/opencode/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

restart opencode to load the skill.

### codex cli

```sh
mkdir -p ~/.codex/skills/refhub-skill
curl -o ~/.codex/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/SKILL.md
```

### cursor, windsurf, and others

copy `AGENTS.md` to your project root:

```sh
curl -O https://raw.githubusercontent.com/refhub-io/refhub-skill/main/AGENTS.md
```

or add it globally via your agent's rules/settings ui.

## // keep the api key out of chat

Recommended pattern: store the key in a local env file and launch agents through a small wrapper instead of pasting the key into prompts.

Create the env file once:

```sh
mkdir -p ~/.config/refhub
cat > ~/.config/refhub/env <<'EOF'
export REFHUB_API_KEY='rhk_REPLACE_ME'
EOF
chmod 600 ~/.config/refhub/env
```

Optional wrappers:

```sh
mkdir -p ~/.local/bin
cat > ~/.local/bin/claude-refhub <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
source "$HOME/.config/refhub/env"
exec claude "$@"
EOF
cat > ~/.local/bin/codex-refhub <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
source "$HOME/.config/refhub/env"
exec codex "$@"
EOF
chmod +x ~/.local/bin/claude-refhub ~/.local/bin/codex-refhub
```

Then start the agent with `claude-refhub` or `codex-refhub`. The plugin/skill reads `REFHUB_API_KEY` from the environment, and the key never needs to appear in chat history.

---

## // stack

```
refhub api  →  refhub-skill  →  refhub cli  →  mcp server (planned)
```

the api is canonical. the skill adapts its surface into agent-friendly workflows without inventing behavior the api does not support.


## API-key Semantic Scholar and PDF workflows (2026-06)

Normal agent runtime is API-key-only:

- Semantic Scholar: `POST /api/v1/semantic-scholar/lookup`, `/doi-metadata`, `/search`, `/recommendations`, `/related`, `/references`, `/citations`, `/cited-by`; all require `vaults:read`. CLI: `refhub discover ...` and `refhub enrich --vault <id> [--item <id>] [--dry-run]`.
- Item PDF upload requires `vaults:write` and a Google Drive account already linked in the RefHub web UI. CLI: `refhub pdf upload --vault <vaultId> --item <itemId> --file <path.pdf>`.
- Small PDFs use raw `POST /api/v1/vaults/:vaultId/items/:itemId/pdf` with `application/pdf` bytes. Raw API uploads are capped at the smallest of `REFHUB_API_MAX_BODY_BYTES`, `GOOGLE_DRIVE_MAX_UPLOAD_BYTES`, and the Netlify synchronous Function ceiling (6 MiB).
- Larger vault-item PDFs use the API-key resumable flow: `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/session`, direct `PUT` of the PDF bytes to the returned Google Drive `upload_url`, then `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/complete`.
- Browser/session JWT item PDF routes live under `/api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf`, `/session`, and `/complete`. API-key agents must not call those `/google-drive/...` routes.
- Google Drive connect/disconnect, API-key lifecycle, legacy `/publications/:publicationId/pdf`, and global audit remain session-JWT/browser account-management flows.
- Search/list accepts canonical `per_page` and `tag`; backend also accepts compatibility aliases `limit` and `tag_id`. DOI filtering is supported.
