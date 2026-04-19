# refhub-skill

> // agent_skill for [refhub.io](https://refhub.io)

agent skill for operating refhub through its public api (v2). agents load `SKILL.md` to discover workflows and consult `docs/` for exact route contracts and behavioral rules. no implementation code — the skill is the contract.

---

## // capabilities

with a scoped refhub api key, an agent can:

- manage vaults (create, update, delete, visibility, collaborators)
- add, update, delete, search, and bulk-upsert papers
- import references from a doi, bibtex string, or url
- manage tags and relations as first-class objects
- sync incrementally via the changes feed
- export vaults as json or bibtex
- read audit logs

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

agents use a pre-issued refhub api key (`rhk_<publicId>_<secret>`). key creation and revocation are human-managed through the refhub ui — not part of the agent runtime.

---

## // execution layer

the [`refhub` cli](https://github.com/refhub/refhub-cli) is the recommended execution layer:

```sh
npm i -g @refhub/cli
```

when available, agents use it instead of making http calls directly. the cli handles authentication, error formatting, and consistent output.

```sh
export REFHUB_API_KEY=rhk_<publicId>_<secret>

refhub vaults list
refhub --help               # discover commands
refhub vaults --help        # group-level help
```

exit codes: `0` success · `1` api error · `2` bad arguments · `3` auth error.

agents without the cli fall back to direct http as documented in `docs/`.

---

## // install

### claude code

```sh
claude plugin marketplace add refhub-io/refhub-marketplace
claude plugin install refhub-skill@refhub-marketplace
```

available in the next session. automatically invoked when you ask claude to work with refhub vaults or papers.

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

---

## // stack

```
refhub api  →  refhub-skill  →  refhub cli  →  mcp server (planned)
```

the api is canonical. the skill adapts its surface into agent-friendly workflows without inventing behavior the api does not support.
