# refhub-skill

Spec-first foundation for the RefHub skill.

This repository does not contain an implementation yet. It defines the initial contract for an agent-facing skill that sits on top of the RefHub API.

## Why this repo exists

RefHub already has:

- a frontend and product model centered on vaults, publications, tags, notes, relations, sharing, and export
- an initial versioned API for API keys, vault listing, vault reads, item writes, and export

RefHub does not yet have:

- a stable, exhaustive public API for all frontend capabilities
- a dedicated CLI
- an MCP server

The skill is therefore the next layer, not the first layer:

`RefHub API` -> `refhub-skill` -> later `CLI` / `MCP`

The API remains canonical. The skill is an agent adapter with opinionated workflows, guardrails, and predictable outputs.

## Scope of this foundation

This foundation documents:

- what the skill is for
- which workflows should exist in v1
- which current API endpoints those workflows depend on
- which workflows are blocked by missing API surface
- how the skill should align later with a CLI and MCP server

It intentionally does not add fake code, placeholder commands, or speculative SDK structure.

## Product grounding

Current RefHub repos show that the product already centers on:

- vaults as the main organizational unit
- vault-scoped publications stored in `vault_publications`
- canonical publication records in `publications`
- vault-scoped tags in `tags`
- item-tag links in `publication_tags`
- citation/relationship links in `publication_relations`
- collaborative access via `vault_shares`
- notes and rich bibliographic metadata on publications
- export to JSON and BibTeX

The current public API is intentionally narrower than the frontend's direct Supabase usage.

## Documents

- [`docs/spec.md`](/opt/openclaw/projects/r2d2/refhub-skill/docs/spec.md)
- [`docs/api-mapping.md`](/opt/openclaw/projects/r2d2/refhub-skill/docs/api-mapping.md)
- [`docs/roadmap.md`](/opt/openclaw/projects/r2d2/refhub-skill/docs/roadmap.md)

## Recommended initial repo structure

```text
refhub-skill/
├── README.md
└── docs/
    ├── spec.md
    ├── api-mapping.md
    └── roadmap.md
```

Why so small:

- the repo is empty today
- the API surface is still evolving
- the right first artifact is a contract, not scaffolding
- later implementation can add manifests, templates, tests, and examples against a settled spec

## Current status

- `implemented in RefHub product`: yes
- `implemented in RefHub public API`: partial
- `implemented in this repo`: spec only

## Immediate next step

Use the docs in this repo as the approval point for the first real skill implementation, keeping the first release limited to workflows the current API already supports cleanly.
