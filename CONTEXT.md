# Domain Glossary

The vocabulary of the dagmar workspace. Skills and agents use these terms
exactly; see `docs/agents/domain.md` for consumption rules.

- **Dagmar interface** — one of the two vendor-agnostic surfaces a factory
  calls: `DagmarIssues` (issue tracking) or `DagmarMemory` (expertise/memory).
  Callers never name the backend technology.
- **Session** — the module (`cliSession`) concentrating the container-exec /
  changeset / error protocol behind a small interface (`Run`, `Mutate`,
  `Fail`, `NoChanges`). Both dagmar interfaces are adapters over a session.
- **Normalized record** — the stable JSON shape (`id`, `kind`, `title`,
  `body`, `status`, `tags`, `meta`, `updatedAt`) every operation returns; the
  authoritative schema lives in `dagmar-contract` (`record-schema`).
- **Normalize** — the pure layer (`internal/normalize`) mapping backend JSON
  to normalized records and normalized filters to CLI flags. No Dagger
  dependency; table-tested.
- **Contract** — the `dagmar-contract` module: enums and record schema shared
  by implementations and consumer applications.
- **Changeset** — the pending workspace diff a write operation returns; host
  state changes only when the caller applies it.
- **Backend** — the CLI tool behind an interface (today seeds/`sd` for
  issues, mulch/`ml` for memory). Swappable without touching callers.
- **Escape hatch** — `Run(args)`: verbatim backend command inside the same
  session, explicitly breaking vendor-agnosticism for expert cases.
