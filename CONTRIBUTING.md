# Contributing to refhub-skill

This is the process guide for changing `refhub-skill`. `AGENTS.md` and `SKILL.md` are agent-runtime instructions for operating RefHub; keep them focused on usage, safety, auth, and API behavior.

## Brand, style, and identity

Agent-facing docs still represent RefHub. Use `refhub-style-guide/refhub-identity.md` for public docs/copy, preserve concise operational wording, and keep route/auth examples concrete without adding imaginary surfaces.

Core rules:

- Prefer concise, practical wording over marketing copy.
- Preserve lowercase, `//` comment-style headings, snake_case labels, and monospace conventions where the repo surface already uses them.
- Keep examples concrete and operational: vaults, papers, tags, relations, PDFs, exports, agents, and API keys.
- Do not introduce a one-off visual, copy, naming, or interaction style for a single feature.

## Existing conventions

Before changing behavior, read:

- `AGENTS.md`
- `SKILL.md`
- `docs/spec.md`
- `docs/api-mapping.md`
- `README.md`

The public API contract comes from the backend and CLI. Do not document imaginary routes or frontend-only Supabase behavior as public API support.

## Scope and branch discipline

Do the work that was asked for. Keep unrelated refactors, formatting sweeps, and speculative API expansions out of focused PRs.

## Pull requests

Never commit directly to `main`.

Use a fresh branch from current `origin/main`:

- `fix/...` for incorrect route/auth/safety guidance.
- `feature/...` for new supported agent workflows.
- `docs/...` for documentation-only changes.
- `chore/...` for packaging or marketplace maintenance.

Open a PR for every change.

## Verification

For doc/runtime instruction changes, verify the public surface by reading the relevant backend/CLI docs or code. For packaging changes, validate JSON manifests and paths:

```sh
python -m json.tool .codex-plugin/plugin.json >/dev/null
python -m json.tool .agents/plugins/marketplace.json >/dev/null
```

When the local toolchain supports it, smoke-test plugin/skill discovery in the target harness before merging.

## Changelog and semver

Keep `CHANGELOG.md` current in the same PR as the shipped change. The file uses Keep a Changelog and Semantic Versioning.

Bump package/plugin metadata when the installable skill behavior changes:

- Patch: corrected guidance, safety fixes, route docs fixes, packaging tweaks.
- Minor: new supported workflows or newly documented API surfaces.
- Major: breaking install layout, auth assumptions, or runtime behavior.

Do not put secrets, API keys, local vault values, or user-specific credentials in examples.

## Security and credentials

Never commit API keys, bearer tokens, local env files, private vault data, source-map contents, or user-specific credentials. Examples must use placeholders only.
