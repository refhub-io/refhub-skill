# RefHub Skill Roadmap

## Phase 0: Spec foundation

Deliverables:

- repo-level purpose statement
- skill workflow spec
- API dependency mapping
- roadmap and alignment rules

Status:

- this repository now covers Phase 0

## Phase 1: Skill v1

Goal:

Ship the first agent-facing RefHub skill using only the currently implemented public API.

Include:

- `vaults.list`
- `vaults.get`
- `items.add`
- `items.update`
- `vaults.export`

Requirements:

- API-key-based runtime auth
- clear structured success and error outputs
- no direct Supabase dependency
- explicit handling of scope errors, vault restrictions, and export formats

Do not include:

- vault creation
- tag mutation
- relation mutation
- sharing flows

## Phase 2: CLI alignment

Goal:

Create a CLI whose verbs mirror the skill operations instead of inventing a separate conceptual model.

Target command shape:

```text
refhub vaults list
refhub vaults get <vault-id>
refhub items add --vault <vault-id> --file items.json
refhub items update --vault <vault-id> --item <item-id> --file patch.json
refhub vaults export <vault-id> --format json
```

Alignment rules:

- CLI verbs should correspond 1:1 to skill operations where possible
- response fields should be structurally similar
- CLI-specific behavior should mainly concern I/O formatting, files, and exit codes

## Phase 3: MCP alignment

Goal:

Expose the same stable operation set as MCP tools for model-runtime integration.

Target tool family:

- `refhub_vaults_list`
- `refhub_vault_get`
- `refhub_items_add`
- `refhub_item_update`
- `refhub_vault_export`

Alignment rules:

- MCP tool names can differ stylistically, but semantics should match the skill and CLI
- tool inputs should stay close to the API contract plus light normalization
- unsupported workflows should be omitted rather than weakly approximated

## Phase 4: Expansion after API growth

Expand only after canonical API routes exist for:

- vault creation and editing
- tag management
- relation management
- sharing and access requests

At that point:

- extend the skill spec first
- then align CLI and MCP additions to the updated spec

## Strategic rule

RefHub should continue to grow in this order:

`API` -> `Skill` -> `CLI` / `MCP`

That keeps the architecture honest:

- one canonical backend contract
- one agent-facing workflow adapter
- multiple delivery surfaces aligned to the same semantics
