# Issue tracker: seeds (via the dagmar wrapper)

Issues and specs for this repo live in the git-native `.seeds/` store (JSONL),
managed through the dagmar wrapper module — a vendor-agnostic interface over
the `sd` CLI. Call `sd` directly only when the wrapper is unavailable.

## Conventions

- Creating an issue: `dagger -y call dagmar-starter issues create --title "…" \
  --kind BUG --priority 1 --tags <labels> changes`
  (the trailing `changes` applies the pending changeset to the working tree;
  without it nothing is written to the host)
- Reading issues: `dagger -q call dagmar-starter issues list --status OPEN json`,
  `… issues get --ids <id> json`, `… issues search --query <text> json`
- IDs are opaque strings (e.g. `src-d111`); never assume a numeric format
- Triage state is carried by issue labels (see `triage-labels.md` for the role
  strings) — pass them via `--tags` on create or `--add-tags` on update
- Lifecycle statuses: `OPEN`, `ACTIVE`, `CLOSED` (active ≙ in_progress)
- Committing the store: `dagger -y call dagmar-starter issues sync changes`
  (explicit; writes run through the lefthook/betterleaks pre-commit chain)
- `--kind` / `--status` / `--sort` flags take enum constant names (`BUG`,
  `OPEN`, `PRIORITY`) — the CLI lists allowed values per flag

## When a skill says "publish to the issue tracker"

Create an issue via the wrapper (command above). Put the spec body in
`--body`. Record triage state as labels via `--tags`.

## When a skill says "fetch the relevant ticket"

Run `dagger -q call dagmar-starter issues get --ids <id> json`. The user will
normally pass the id directly; ids surface in `issues list` output.

## Wayfinding operations

Used by `/wayfinder`. The **map** is an `EPIC`-kind issue; **children** are
ordinary issues blocked by each other.

- **Map**: an issue of kind `EPIC` whose body holds the Notes /
  Decisions-so-far / Fog sections; reference it by id.
- **Child ticket**: an ordinary issue; its type is a `type:<t>` label
  (`type:research`, `type:prototype`, `type:grilling`, `type:task`).
- **Blocking**: `dagger -y call dagmar-starter issues dep-add --id <blocked> \
  --blocked-by <blocker> changes`. A ticket is unblocked when every blocker is
  `CLOSED`.
- **Frontier**: `dagger -q call dagmar-starter issues ready json` — open issues
  with no unresolved blockers, sorted by priority (native fit).
- **Claim**: update status to `ACTIVE` before any work:
  `dagger -y call dagmar-starter issues update --id <id> --status ACTIVE changes`.
- **Resolve**: append the answer to the issue body, then close:
  `dagger -y call dagmar-starter issues update --id <id> --body "<answer>" changes`
  followed by `… issues close --ids <id> --reason resolved changes`; finally
  record the context pointer on the map issue body (update the epic).
