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
skills/refhub-skill/SKILL.md  ← skill entry point
AGENTS.md                      ← instructions for cursor, windsurf, pi, and other generic harnesses
.claude-plugin/                ← Claude Code plugin + self-hosted marketplace manifest
.codex-plugin/                 ← Codex plugin manifest
.agents/plugins/marketplace.json ← self-hosted Codex marketplace manifest
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
claude plugin marketplace add https://github.com/refhub-io/refhub-skill
claude plugin install refhub-skill@refhub-skill
```

available in the next session. automatically invoked when you ask claude to work with refhub vaults or papers.

Why the explicit HTTPS URL: `claude plugin marketplace add refhub-io/refhub-skill` uses GitHub shorthand and Claude clones it over SSH (`git@github.com:...`), which fails on machines without a GitHub SSH key. The HTTPS repo URL avoids that while keeping the normal Claude marketplace flow.

### codex

**from github** — add to your Codex plugin marketplace configuration (`~/.agents/plugins/marketplace.json` or project `.agents/plugins/marketplace.json`):

```json
{
  "name": "refhub-skill",
  "source": { "source": "github", "repo": "refhub-io/refhub-skill" },
  "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
  "category": "Productivity"
}
```

this repo already ships its own `.agents/plugins/marketplace.json`, so cloning it locally and pointing Codex at the clone works too:

```json
{
  "name": "refhub-skill",
  "source": { "source": "local", "path": "~/plugins/refhub-skill" },
  "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
  "category": "Productivity"
}
```

manifest: `.codex-plugin/plugin.json`

### gemini cli

```sh
mkdir -p ~/.gemini/skills/refhub-skill
curl -o ~/.gemini/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/skills/refhub-skill/SKILL.md
```

### opencode

```sh
mkdir -p ~/.config/opencode/skills/refhub-skill
curl -o ~/.config/opencode/skills/refhub-skill/SKILL.md \
  https://raw.githubusercontent.com/refhub-io/refhub-skill/main/skills/refhub-skill/SKILL.md
```

restart opencode to load the skill.

### cursor, windsurf, pi, and other generic harnesses

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
- All API-key item PDF uploads use the resumable flow: `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/session`, direct `PUT` of the PDF bytes to the returned Google Drive `upload_url`, then `POST /api/v1/vaults/:vaultId/items/:itemId/pdf/complete` — at any file size, with no raw-bytes upload path. `POST /api/v1/vaults/:vaultId/items/:itemId/pdf` itself only accepts a JSON `{ source_url }` body now; a raw `application/pdf` body there returns `410 raw_pdf_upload_removed`.
- The resulting Google Drive URL is returned as `data.driveUrl` (with `data.fileId`) in the complete response — **and only there.** No GET route returns it afterward; see the field-parity note below. `driveUrl` is deliberately distinct from `pdf_url` (the publisher-hosted PDF link) — they are unrelated fields that happened to share a near-identical name before this rename.
- Browser/session JWT item PDF routes live under `/api/v1/google-drive/vaults/:vaultId/items/:itemId/pdf`, `/session`, and `/complete`. API-key agents must not call those `/google-drive/...` routes.
- Google Drive connect/disconnect, API-key lifecycle, publication-level PDF upload (`/publications/:publicationId/pdf/session` + `/complete`, same resumable-only flow, no raw-bytes variant), and global audit remain session-JWT/browser account-management flows.
- Search/list accepts canonical `per_page` and `tag`; backend also accepts compatibility aliases `limit` and `tag_id`. DOI filtering is supported.

## Publication field parity with the frontend (2026-07)

Verified against the live API that `POST/PATCH /vaults/:vaultId/items[/:itemId]` already accept the full field set from the frontend's publication dialog (`url`, `pdf_url`/`publisher_pdf`, `notes`, plus the bibtex-oriented fields) — CLI `items add`/`items update` now expose `--url` and `--pdf-url` alongside the existing `--notes`.

The one confirmed gap: the frontend's `drive_pdf` field (the Google Drive-hosted copy, from `publication_pdf_assets.stored_pdf_url`) is **not** returned by any GET route. `refhub pdf upload` performs a real Drive upload and returns the resulting URL in its response (`data.driveUrl`) — that is the only time the API surfaces it. There is no route to read it back afterward. See `docs/spec.md` §7.25 and `docs/api-mapping.md` for details.

**Naming:** `data.driveUrl` (upload response) and `pdf_url` (publisher-hosted PDF, on the publication object) are unrelated fields — pending a backend rename from the previous `data.pdfUrl` to avoid the two being confused.

**Reading PDFs:** `pdf_url` is a plain external link — fetch it like any web resource. The Drive-hosted copy has no read-back route at all, and even its view link (`https://drive.google.com/file/d/<fileId>/view`) isn't a raw download — actual byte content requires Google's Drive API with an OAuth token this skill doesn't have. See `docs/spec.md` §7.27 and `SKILL.md`'s "Reading a publication's PDF(s)" section.
