# dagmar-starter

Dagger workspace providing vendor-agnostic wrappers (`dagmar-issues`,
`dagmar-memory`) over git-native tooling, plus the shared type contract
consumed by agentic software factories. See `README.md` for the quick start.

## Agent skills

### Issue tracker

Issues live in the git-native `.seeds/` store, managed through the dagmar
wrapper (`dagger call dagmar-starter issues …`). See
`docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles map 1:1 to seeds labels
(`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`).
See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` + `docs/adr/` at the repo root. See
`docs/agents/domain.md`.
