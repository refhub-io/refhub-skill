# Changelog

All notable changes to `refhub-skill` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this
project uses [Semantic Versioning](https://semver.org/). History prior to
1.1.0 was not tracked in this file.

## [1.3.0] - 2026-09-08

### Added
- Documented curated vault sections (`GET/POST /vaults/:vaultId/sections`, `PATCH/DELETE .../sections/:sectionId`) across `AGENTS.md`, `SKILL.md`, `docs/spec.md`, and `docs/api-mapping.md`, matching `.netlify` v2.7.0 (#196). List is viewer-level; writes require vault **owner** permission specifically, not just editor.
- Documented `section_id`/`section_position`/`featured`/`featured_note` as accepted fields on `PATCH /vaults/:vaultId/items/:itemId`, and the owner-only access check the backend runs for them separately from the rest of that endpoint.
- Documented citation-based relationship-suggestion scanning as now supported: `refhub relations scan --vault <id> [--item <id>] [--dry-run] [--limit <n>]` (`refhub-cli` v1.6.0) — pure client-side orchestration on the existing Semantic Scholar and relation endpoints, no dedicated backend route, mirroring `refhub enrich`'s pattern. Only ever proposes `cites` relations.

### Fixed
- `AGENTS.md`'s Vaults route list was missing `POST /vaults/:vaultId/archive` even though the rest of the file already documented the workflow — added.
- Corrected `403` error-code documentation in `AGENTS.md` and `SKILL.md`: the real codes are `insufficient_vault_access` and `vault_not_allowed`, not the previously-documented `vault_access_denied` (which doesn't exist); `vault_not_found` is `404`, not `403`.

### Changed
- Removed relationship-suggestion scanning from every "not yet implemented" / "do not apply" / non-goals list across all four docs — manual relation CRUD was already supported, and the automated scanning step now is too, via the CLI.

## [1.2.0] - 2026-09-08

### Added
- Documented vault archiving (`POST /vaults/:vaultId/archive`, `vaults:admin` + owner) across `AGENTS.md`, `SKILL.md`, `docs/spec.md`, and `docs/api-mapping.md`, matching refhub.io #152 and `.netlify` v2.6.0. Includes the `409 vault_archived` error code and the `archived_at` vault field.

### Changed
- Corrected every prior "vault archiving has no API route" claim — it now does. Unarchiving remains explicitly documented as never supported, by design (not a deferred gap).
- Added relationship-suggestion scanning (the frontend's citation-matching workflow) to the not-yet-implemented lists that previously only mentioned vault archiving — manual relation CRUD was already, and remains, fully supported.

## [1.1.1] - 2026-07-21

### Added
- Documented `reading_state` (`unread`/`skimmed`/`read`, defaults to
  `unread`) and `important` (boolean, defaults to `false`) as accepted
  item fields in `docs/spec.md` and `SKILL.md`, matching refhub.io v1.7.0
  (issue #94) and the `.netlify` v2.5.0 API update. Like `notes`, both are
  vault-local per item and never propagate to sibling copies of the same
  paper in other vaults.

## [1.1.0] - 2026-07-08

### Added
- When the `refhub` CLI isn't found, the skill now asks the user upfront
  (with the install/setup commands included) whether to set it up,
  instead of silently falling back to direct API calls. Declining (or
  not responding) still falls back to direct HTTP calls — this is a
  nudge, not a hard requirement.
- Documented `url`/`pdf_url` fields on item add/update, and the full
  bibtex-oriented field set, matching the frontend's publication dialog
  one-for-one.
- Documented that the stored Drive PDF link is now readable back as
  `drive_pdf_url` on item reads and the refreshed row from item update
  (previously only ever returned once, in the upload response).
- Documented that publication-level PDF upload
  (`POST /publications/:publicationId/pdf/session` + `/complete`, for
  library-only papers with no vault) is API-key compatible
  (`vaults:write`) — corrected from six places across this repo that
  previously and incorrectly grouped it with JWT-only management routes.
- Synced `docs/spec.md`'s "exists now" list with `AGENTS.md`: added the
  `/related` and `/cited-by` Semantic Scholar routes and the
  publication-level PDF upload route, which were missing from `spec.md`
  but already correct in `AGENTS.md`.

### Changed
- Renamed the PDF upload response field `pdfUrl` → `driveUrl` across
  `SKILL.md`, `docs/spec.md`, `README.md`, `docs/api-mapping.md` — it
  collided in name with the unrelated `pdf_url` (publisher-hosted PDF)
  field despite being a different concept (the Google Drive-hosted copy).
- Documented that PDF upload always uses the resumable Drive flow now,
  regardless of file size — the raw-bytes upload path is gone, matching
  the backend's resumable-only pivot.
