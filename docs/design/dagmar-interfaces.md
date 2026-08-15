# Dagmar — Abstract Wrapper Interfaces for Issues & Memory

**Status:** Draft for discussion (v0.2, updated for Dagger "next") · **Scope:** `.dagger/modules/dagmar-starter` + `dagmar-contract`

## 1. Goal & Non-Goals

**Goal:** An agentic software factory calls *one* stable, normalized interface
(`DagmarIssues`, `DagmarMemory`) and stays ignorant of the underlying backend
(currently: seeds/`sd`, mulch/`ml`; later e.g. GitHub Issues, Linear, beads …).
Backends are swappable without touching callers.

**Non-Goals:** Full feature parity across backends; multi-repo sync semantics;
UI/rendering (format hint only).

## 2. Layered Model

```
Agentic factory
   │  calls (CLI / GraphQL / generated clients)
   ▼
dagmar-contract (own module)         ← shared types & enums, imported by all
   │                                 consumers and implementations
   ▼
Dagmar-Starter (Dagger module)       ← DagmarIssues / DagmarMemory objects
   │  adapter layer (one file per backend)
   ▼
Backend adapters: seeds (sd) · mulch (ml) · …
   │  CLI invocation inside a container, always --json
   ▼
Container (bun image + @os-eco/*-cli) with mounted workspace state
```

Everything is Dagger-native: each operation is a module function; the CLI runs
in a container; repo state (`.seeds/`, `.mulch/`) comes from the `Workspace`.
Write operations return a **Changeset** which the caller applies (nothing hits
the host working tree until then; `dagger … changes` + `-y` or interactive).

## 3. Normalized Data Model: `DagmarRecords`

All read/write results come back as JSON with *stable* field names:

```jsonc
{
  "id": "src-7651 | mx-c4638e | 123",     // opaque string, no schema assumption
  "kind": "task|bug|feature|epic|other",   // issues
             // convention|pattern|failure|decision|reference|guide  (memory)
  "title": "…",
  "body":  "…",                            // description/content
  "status":"open|active|closed",           // issues (active ≙ in_progress)
             // live|archived              (memory)
  "tags":  ["…"],                          // labels/tags (unified as "tags")
  "meta":  { … },                          // normalized extras (priority,
                                            // assignee, tier, evidence, deps …)
  "raw":   { … },                          // unmodified backend output (audit)
  "updatedAt": "RFC3339"
}
```

Rules:
- Only `id` is guaranteed; everything else is optional. Factory code reads core
  fields only.
- The authoritative schema of this JSON is owned by the `dagmar-contract`
  module (`record-schema` + `validate-records`); implementations and consumers
  validate against it (see §8).
- Vendor specifics surface through `meta` (normalized when the adapter knows it)
  and `raw` (always present).
- Escape hatch: optional `attrs` JSON on Create/Update maps to the backend's
  extensions field (seeds supports `extensions` natively; other adapters map or
  drop it with a warning).

## 4. `DagmarIssues` — Function & Argument Model

Flat, typed arguments per the Dagger "next" type-system guidance (no options
bags; enums for closed sets; `+optional`/defaults; `-1` sentinel for ints where
0 is a valid value; empty string/list = "not set").

```
Init()                                    // bootstrap the store, idempotent
Create(title, body?, kind=task, priority=-1, assignee?, tags?, attrs?)
Get(ids []string)
List(status?, kind?, assignee?, tagsAll?, tagsAny?, priorityMax=-1, limit=50, sort=priority)
Search(query, <same filters as List>)
Update(id, title?, body?, kind?, priority=-1, assignee?, status?, attrs?, addTags?, removeTags?)
Close(ids, reason?)
Ready()                                   // open + no unresolved blockers
DepAdd(id, blockedBy) / DepRemove(id, blockedBy) / Deps(id)
Prime(format=markdown)                    // agent context assembly
Sync()                                    // stage + commit backend store
Run(args)                                 // documented raw passthrough
Capabilities()                            // machine-readable backend abilities
```

Normalized enums: `IssueKind` (task|bug|feature|epic|other), `IssueStatus`
(open|active|closed|all), `IssueSort` (priority|created|updated|id).

## 5. `DagmarMemory` — Function & Argument Model

```
Init()
Remember(scope, kind=reference, title?, content?, tags?, files?, links?,
         supersedes?, evidenceCommit?, evidenceIssue?, evidenceURL?, evidenceRef?)
Get(ids []string)
List(scope?, kind?, tier?, file?, outcome?, sortByScore?, limit=-1)
Search(query, scope?, kind?, tier?, tag?, file?, outcome?, includeArchived?, sortByScore?, limit=-1)
Update(id, title?, content?, tier?, tags?, files?, links?, supersedes?)
Archive(ids, reason) / Restore(id)
Outcome(id, status, agent?, durationMs=-1, summary?)   // feedback loop
Prime(scopes?, files?, budget=4000, format=markdown)
Sync(message?)
Run(args) / Capabilities()
```

Normalized enums: `MemoryKind` (convention|pattern|failure|decision|reference|
guide|note — note maps to reference), `MemoryTier` (foundational|tactical|
observational), `OutcomeStatus` (success|failure|partial), `RecordFormat`
(markdown|compact|xml|plain).

## 6. Capability Model & Degradation

No backend can do everything — hence:

| Capability            | seeds (`sd`)        | mulch (`ml`)          | On ✗ |
|-----------------------|---------------------|-----------------------|------|
| opaque ids            | ✓                   | ✓                     | — |
| full-text search      | ✓ (substring)       | ✓ (BM25 + score boost)| — |
| tags/labels           | ✓                   | ✓                     | — |
| priority              | ✓ P0–P4             | ✗                     | ignore + warn |
| status lifecycle      | open/progress/closed| live/archived         | domain mapping |
| dependencies          | ✓ dep add/remove    | relates-to/supersedes | link fallback |
| evidence/provenance   | extensions          | ✓ (rich)              | → attrs/meta |
| outcomes/feedback     | ✗                   | ✓                     | ignore + warn |
| prime (context budget)| ✓ sd prime          | ✓ budget tokens       | — both! |
| JSON output           | ✓                   | ✓                     | prerequisite |

- `Capabilities()` makes this machine-readable; the factory can drive adaptive paths.
- Unknown fields: normalized into `meta`, visible in `raw` — never an error.
- `Run(args)` is the documented passthrough for expert cases (explicitly breaks
  swappability, by design).

## 7. Workspace, Changeset & Container Semantics

- **Reads:** mount workspace state read-only, run CLI in container, normalize JSON.
- **Writes:** run CLI against the mounted state, then produce a **Changeset**
  (diff of workspace before/after). The host working tree only changes when the
  caller applies the changeset. No implicit `git commit`.
- **Sync:** explicit function; runs `sd sync` / `ml sync`; its changeset carries
  the resulting `.git` state (commit objects + refs) so applying it lands the
  commit on the host.
- **Container:** single bun base image (`oven/bun:1.3.13`) + `bun install -g`
  of the pinned `@os-eco/*-cli` packages. Versions are pinned centrally in the
  adapter layer. mise stays host-side for developers.
- **Engine constraint (object round-trip):** the engine stores module objects
  as JSON and reconstitutes them per call — only *exported* fields survive.
  Adapter objects therefore carry `Source`/`Repo` as exported `+private`
  fields and construct their `cliSession` per call; the session itself is
  never stored on an object.

## 8. `dagmar-contract` — Shared Types Module

The contract lives in its own module so every application that consumes
dagmar-issues/dagmar-memory can import the same definitions. It owns:

1. **Enums** (backend, kind, status, sort, tier, outcome, format) — exposed as
   typed enums in generated clients, plus `validate-*` functions and value
   listings.
2. **The normalized record schema** — `record-schema` returns the authoritative
   JSON schema of `DagmarRecords.json`; `validate-records` checks any records
   JSON against it and reports per-record violations.

Why the record *type* itself stays module-local in each implementation: the
Dagger engine forbids referencing external types from dependency modules in
function signatures. What crosses module boundaries is the JSON string — so the
contract is enforced where it is real:

- **Enum values**: bound at compile time (`IssueKindTask = IssueKind(string(
  dagger.DagmarContractIssueKindTask))`) — a contract change breaks the build.
- **Record shape**: every implementation's `DagmarRecords` exposes `validate`
  and `validate-details`, which run the contract validator over the actual
  output. Consumers can do the same with raw JSON.
- Contract evolution is explicit: a change there is a breaking change for every
  consumer, decided once.

## 9. Resolved Decisions

1. **Return type:** typed `DagmarRecords` object with `Json()`, `Raw()` and
   `Changes()` accessors. ✔
2. **Id level:** opaque strings only. ✔
3. **Filters:** flat typed args (Dagger "next" style), no options structs. ✔
4. **Status vocabulary:** normalized `open|active|closed` (active ≙ in_progress). ✔
5. **Prime:** on both interfaces separately (issues: workflow planning;
   memory: expertise context). ✔
6. **Sync autonomy:** manual only, no auto-sync on writes. ✔
7. **Host writes:** via Changeset, applied explicitly by the caller. ✔ (new)
