# Tapstate Resource Model

Use `tapstate-v1.schema.json` as the exact grammar authority. This reference
explains identity, ownership, and reference semantics that are easy to miss when
reading the schema alone.

## Contents

- Project and identity
- Source
- Pipeline
- Reusable definitions
- Reference semantics
- Validation boundary

## Project and identity

- A Tapstate project is a folder of `*.tap.yml` files. Its identity is declared
  by `project.tap.yml` at the project root:

  ```yaml
  version: tapstate/v1
  kind: project
  id: bank_c360
  metadata:
    description: Customer 360 on the core banking and cards systems
  ```

  It allows only `version`, `kind`, `id`, and optional `metadata` (`labels`,
  `description`). Never put deployment targets, connection settings, or other
  configuration in it. `tapstate new` writes this file for every recipe; create
  it when starting a project by hand. A folder without it still works: the
  project id then defaults to the folder name, and `tapstate up` prints a
  hint suggesting adding the file. Do not add it to an existing
  project unless the user asks.
- The CLI finds the project root from the `-w/--workdir` flag, then
  `TAPSTATE_WORKDIR`, then the nearest enclosing folder that contains
  `project.tap.yml`, then the legacy default `tap-work`, so CLI commands work
  from any subfolder of a project.
- Store one top-level resource in each `*.tap.yml` file. For new projects,
  create kind directories under the requested project root: `source/`,
  `pipeline/`, `transform/`, `view/`, and `serve/`. These directories are a
  Tapstate project convention, not paths within the installed Skill. The
  loader discovers files recursively, so a legacy flat project remains valid
  and does not need a layout-only migration.
- Set `version: tapstate/v1` and one of the five resource kinds: `source`,
  `pipeline`, `transform`, `view`, or `serve`. `project` is the kind of the
  project identity file only, never of a resource.
- Give every top-level resource an `id` that is unique across all kinds in the
  project. A dot is forbidden because `source_id.table` uses it as an
  addressing separator.
- Treat `metadata.labels` and `metadata.description` as annotations only. They
  never establish identity. `metadata.labels.project` is reserved: `tapstate up`
  sets it to the project id on every resource the project declares,
  and writing a different value by hand is a validation error. Do not author it.
- One server can run many projects, and a project can be deployed to several
  servers, but resource ids stay unique per server. If another project on that
  server already owns an id, the apply is refused with
  `artifact.project-id-taken`, naming the owning project; rename the resource in
  this project or remove it from the other project first. Sources a project only
  references, rather than declares, are not labeled and stay shared.
- Keep unknown stable fields out of the document. Use `experimental` only for
  explicitly experimental data.

Pipeline-internal ids share one private namespace across transform steps, the
inline view, the inline serve block, and sync or push elements. Transform step
ids must also not shadow a Pipeline source id or a literal table name.

## Source

A `source` owns a connector relationship. It has two roles:

1. A read source declares `mode`; omit `tables` by default so the connection can
   be reused by multiple Pipelines. Add `tables` only when the user explicitly
   says that this Source itself is restricted to a subset of tables (or when
   preserving an existing Source-level allowlist).
2. A connection supplier for `serve.sync` or `serve.push` may omit both.

`connector` is required. `config` is the only dynamic DSL member boundary: its
member names, types, defaults, visibility rules, and secret flags must come from
the live connector catalog through MCP. When available, `source_draft` validates
a complete Source and renders its canonical YAML without persisting it. Write
that result to the local Source file, then apply the complete project only when
online execution is requested. Omit `config` when the live contract is not
available. When editing an existing source, preserve its complete `config`
mapping by default and never reveal secret values.

The static source semantics are:

| Field | Meaning |
|---|---|
| `mode` | `cdc`, `snapshot`, `stream`, `file`, or `api`; connector capability validation decides which pairings are legal. |
| `tables` | An optional Source-level allowlist: literal names, `/regular-expression/` selectors, or object-form table specifications. It is not the place to record which tables one particular Pipeline synchronizes. |
| `tables[].filter` | A source-row CEL predicate. Its column environment is discovered online, so offline validation does not type-check it. |
| `tables[].pk` | An explicit primary-key override for a literal table. |
| `tables[].options` | Per-table extension options. |
| `options` | Source-level extension options. `snapshot_mode` and `start_from` are forbidden here because the read axis belongs to Pipeline settings. |
| `srs` | Shared Record Store behavior; legal only for `mode: cdc`. |

`srs.key` asserts a shared mining-chain identity. `retention` controls retained
changes, `schema_evolution` is `track` or `ignore`, `queryable` declares direct
query intent, and `enabled: false` requests a direct single-consumer tail with no
shared replay buffer. The default for `enabled` is true.

## Pipeline

A `pipeline` is the composing runnable unit. `source` accepts one source id or a
list and always refers to top-level `source` resources in the same project.
The Pipeline wires inline or reusable transforms to an optional view and serve
surface. It must produce an output through `view` or `serve`; merely naming a
source is not a complete composition.

Table scope belongs to the Pipeline. For each read stream, put the tables this
Pipeline consumes in the first transform's `from`, the view/serve `from`, or a
later transform's explicit `from` as appropriate. A Source with omitted
`tables` supplies an open table universe; a Pipeline can then select different
tables from that same Source without changing the Source definition. A
Source-level `tables` list narrows that universe for every Pipeline that uses
the Source and therefore requires explicit user intent.

See `pipeline-semantics.md` for settings, stream addressing, transforms, views,
and serving.

## Reusable definitions

`transform`, `view`, and `serve` resources are pure reusable bodies:

- A `transform` contains one of the six transform bodies and optional transform
  options.
- A `view` contains materialization identity, storage, and schema policy.
- A `serve` contains sync, query, and push declarations.

Never put `from` in a reusable definition. The Pipeline use site supplies
wiring. A Pipeline step uses `use: transform_id`; a Pipeline `view` or `serve`
block uses `use: definition_id`. The use site's `id` defaults to the referenced
id, and transform use sites may override only `options`, not the body.

## Reference semantics

Literal `from` tokens resolve in this order-sensitive project model:

- a preceding Pipeline step id;
- a literal table name selected by one of the Pipeline's sources;
- `source_id.table` when an unqualified table name would be ambiguous;
- a source id when the source has no addressed table.

A literal table selector is a frozen link. A `/regular-expression/` table or
`from` reference is dynamic: newly matching tables can enter its universe.
Offline validation accepts an unresolved literal only when an open table
universe prevents proving that it is missing. Use qualified names to remove
cross-source ambiguity, then check `runtime-support.md` before presenting them
as executable.

References to top-level source, transform, view, and serve ids must resolve in
the same project. `serve.sync[].source` and `serve.push[].source` always name
a top-level `source` used as a target connection supplier.

## Validation boundary

Run:

```sh
tapstate validate path/to/project
```

Validation loads every `*.tap.yml` file recursively, rejects malformed or
unknown fields, checks ids and reference closure, compiles envelope-rooted CEL,
and applies the bundled connector capability projection. Success proves offline
validity against that installed CLI. It does not prove canonical serialization,
live connector registration, credentials, connectivity, or runtime support.
