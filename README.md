# dagmar-starter

Dagger workspace providing **vendor-agnostic wrappers** for issue tracking and
agent memory, so an agentic software factory can work against one stable
interface without knowing the underlying technology.

## Modules

| Module | Purpose |
|---|---|
| `dagmar-starter` | Implements `DagmarIssues` and `DagmarMemory` (currently backed by [seeds](https://github.com/jayminwest/seeds) `sd` and [mulch](https://github.com/jayminwest/mulch) `ml` from the os-eco ecosystem). |
| `dagmar-contract` | The shared type contract: enums (backend, kind, status, tier, outcome, format), the authoritative normalized-record schema (`record-schema`) and its validator (`validate-records`). Every consumer of dagmar-issues/dagmar-memory imports this module. `dagmar-starter` binds enum values at compile time and validates record output against the schema, so contract and implementation cannot drift silently. |

## Quick start

```bash
# Issues (backed by seeds/sd)
dagger -y call dagmar-starter issues init changes
dagger -y call dagmar-starter issues create --title "Fix login" --kind BUG --priority 1 changes
dagger -q call dagmar-starter issues list --status OPEN json
dagger -q call dagmar-starter issues ready json
dagger -y call dagmar-starter issues sync changes   # stage + commit .seeds/

# Memory (backed by mulch/ml)
dagger -y call dagmar-starter memory init changes
dagger -y call dagmar-starter memory remember --scope coding --kind CONVENTION \
        --title "no-any" --content "Avoid any in Go" --tags style changes
dagger -q call dagmar-starter memory search --query dagger json
dagger -y call dagmar-starter memory outcome --id <id> --status OUTCOME_SUCCESS --agent factory changes
dagger -q call dagmar-starter memory prime --budget 2000 --format FORMAT_COMPACT

# Validate outputs against the contract
dagger -q call dagmar-starter issues list validate          # true
dagger -q call dagmar-contract validate-records \
        --records-json '[{"id":"x1","tags":["a"],"meta":{}}]' valid

# Contract
dagger -q call dagmar-contract version
dagger -q call dagmar-contract record-schema | head
dagger -q call dagmar-contract validate-issue --kind TASK --status OPEN --sort PRIORITY
```

## Architecture

```
DagmarIssues / DagmarMemory   ← thin adapters (flag translation + result shaping)
        │
   cliSession                  ← deep module: Run / Mutate / Fail / NoChanges
        │                        (container-exec, changeset, error protocol)
   internal/normalize          ← pure layer: backend JSON → records,
        │                        filters → flags (table-tested, no engine)
   seeds (sd) · mulch (ml)     ← backends in pinned bun containers
```

`go test ./internal/normalize/` pins the record-schema mapping without a
Dagger engine.

## Semantics

- **Normalized records**: every operation returns a `DagmarRecords` object with
  `json` (stable core fields: `id`, `kind`, `title`, `body`, `status`, `tags`,
  `meta`, `updatedAt`), `raw` (unmodified backend output, for audit) and
  `changes`.
- **Changesets**: write operations never touch the working tree directly.
  Select `changes` at the end of a call chain and apply it (`-y` or interactive
  confirmation). Nothing lands on the host before you approve.
- **Sync**: explicit, never automatic. `sync` runs the backend's commit step
  and its changeset carries the resulting git state.
- **Opaque ids**: ids are strings; never assume a format.
- **Enum args**: pass enum constant names (e.g. `--kind BUG`,
  `--status OUTCOME_SUCCESS`); the CLI lists allowed values per flag.
- **Escape hatch**: `issues run --args sd --args stats --args --json` executes
  backend commands verbatim inside the same container.

Design details: [docs/design/dagmar-interfaces.md](docs/design/dagmar-interfaces.md)

## Toolchain (host-side, for developers)

Tools are pinned in `mise.toml` (bun, lefthook, betterleaks, seeds/sd,
mulch/ml). Git hooks run betterleaks on staged files (see `lefthook.yml`).
