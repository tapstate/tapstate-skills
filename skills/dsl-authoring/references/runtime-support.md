# Current Preview Runtime Support

This matrix is pinned to Tapstate commit
`9a280177fe3267e102bbf067b84b77cb39bfd79a`. That commit is a preview
integration of the 0.8.0 line with features that are still under review
upstream: rollup fields on a nest embed, the `aggregate` step, several tables
in one nest array, `map.soft_delete`, `orOn` alternatives on a nest embed and
OR alternatives in a join, and ordering of same-type temporal values in row
expressions. A released Tapstate that does not carry those changes rejects
them; see "Released CLI" below. The generated schema describes the complete
`tapstate/v1` grammar at that commit; this file describes the narrower runtime
wired there. Update both this matrix and the repository upstream lock when the
baseline changes.

Offline validation and runtime execution are independent claims. All bundled
examples pass offline validation, but none is runnable until MCP or another
trusted live source supplies the connector-specific `source.config` members.

## Online authoring handoff

When MCP exposes `source_draft`, use it to validate a structured Source config
against the live connector contract and render canonical YAML. It does not write
an artifact or create an audit record. The authoring sequence is local file
creation, offline validation, `connection_test` and
`connection_discover_schema` for every read Source, complete-closure
`artifact_validate`, complete-closure `artifact_apply`, then optional Pipeline
start. Sink-only connection suppliers are not discovered. Fix discovery or
validation diagnostics before applying. Do not apply a standalone Source as a
substitute for workspace authoring.

### CLI equivalents

The Tapstate CLI (0.8.0 and later) drives the same server. Each command below
runs once with `tapstate -c <url> -u <user> <command>` or inside an interactive
`tapstate` session; the password comes from `TAPSTATE_PASSWORD` or a prompt.

| Step | CLI |
|---|---|
| Offline validation | `tapstate validate <workspace-or-file>` |
| Test a connection | `test <source>` |
| Discover a read Source | `discover-schema <source>` |
| Apply the workspace or one file | `apply` or `apply <file>` |
| Start, answering start checks as configured | `start <pipeline> -y` (`--checks-only` shows them) |
| Stop | `stop <pipeline> -y` clears accumulated state; `--keep-state` keeps it |
| Observe | `status <pipeline>`, `metrics <pipeline>`, `logs <pipeline>`, `explain <pipeline>` |
| Read, change, or remove a stored resource | `get <id>`, `apply <file> --if-match <hash>`, `delete <id> --if-match <hash>` |
| Check or repair aggregate groups | `aggregate <pipeline> <step> [--group k=v,...] [--repair]` |

`tapstate up` applies, discovers, applies, and starts in one step. The same
rules hold: discover before apply, and do not stop, delete, or repair without
the user's request.

MCP availability is established by the tools exposed to the current agent
session. A local `tapstate validate` result, shell command, or file inspection
does not establish Server reachability. If a required MCP tool is exposed, call
it directly and preserve its transport or structured error; only a direct
transport failure justifies reporting that the Server is unreachable.

### Changing and removing applied resources

`artifact_get` reads one applied resource of any kind by id and returns its
canonical YAML with the `contentHash` of exactly those bytes. `artifact_delete`
removes one applied resource of any kind by id and requires both `id` and
`expectedContentHash`. It is write-scoped, so it appears only in a session
permitted to write. Each draft of an `artifact_apply` closure accepts an
optional `expectedContentHash`, and one stale precondition refuses the whole
closure before any resource in it is written.

`artifact_list` is not exposed on this face at this commit. An id must already
be known, or come from a face that does list.

Removal is permanent and leaves no tombstone. Two grounds refuse it, both judged
before anything is written, so a refusal leaves storage unchanged:
`artifact.in-use` when another resource still references the id, and
`artifact.pipeline-not-stopped` when the id is a pipeline whose actual or
desired state is running or about to be. A precondition that does not match the
stored version is `artifact.version-conflict`; an id that is not stored at all
is `artifact.not-found`.

Removing a pipeline reclaims dependent state in the same operation. What is and
is not reclaimed is a boundary rather than an implementation detail, because
everything that survives is shared with something else:

| On removing a pipeline | Current preview runtime |
|---|---|
| Its desired, state, and observation documents | Reclaimed in the same operation, so the converge side stops reconciling the id. |
| Its own consumer entry on a shared SRS mining chain | Detached. A departed consumer's cursor is otherwise folded into two independent minimums that pin the chain's durable frontier and its cdc write headroom for every other pipeline on that chain, raising no error. |
| The mining chain record itself | Never removed. Other pipelines may still be consuming it. |
| Nest state | Never removed. |
| Target tables and target database contents | Never removed. Removing a pipeline removes the declaration, not the data it already wrote. |

A reclaim that fails does not put the artifact back: the removal has been judged
and applied by then. Every reclaim step is attempted regardless, and the failures
are reported together as `artifact.reclaim-incomplete`, carrying `id`, `reason`
and the `residue` it did not clear. This is the one failure here that is not a
refusal: the resource is already gone, so repeating the removal can only answer
`artifact.not-found`. Report it; do not retry it.

Its `reason` decides the next step, and the two differ:

| `reason` | What happened | Next step |
|---|---|---|
| `step-failed` | The removal stands and the steps that succeeded stand; the named `residue` was not cleared. | Clear the named residue by hand. |
| `pipeline-live` | The pipeline was started again after the refusal was judged and before the reclaim ran, so nothing was reclaimed and all four records are named as residue. | Stop the pipeline first. Clearing by hand while its job still runs discards the fencing epoch, which is what keeps that job from colliding with a later pipeline applied under the same id. |

## Capability matrix

| Surface | Grammar and offline validation | Current preview runtime |
|---|---|---|
| Read Source | All source modes, literal/object/regex table selectors, and multiple tables are authorable. | Supported, including several tables and regex selectors. An object-form table entry with `filter` or `pk` fails at assembly with `actuation.source-table-spec-unsupported`. |
| Target Source | A source may omit mode and tables when used only as a connection supplier. | Supported by inline sync; connector and config are used to open the sink. |
| Source references | Table, step, source-qualified table, and regex references validate. | Literal table names, step names, and quoted regex patterns are resolved. A regex that matches nothing fails with `actuation.from-regex-empty`. |
| Inline filter | CEL boolean expressions validate. | Supported. Row events are filtered; an update that stops matching becomes a delete of the previous row; DDL bypasses. |
| Inline map | Rename, drop, literal, `=CEL` rules, and `soft_delete` validate. | Supported over `after`; deletes and DDL bypass unless `soft_delete` is set, which turns a delete into a marked update. Unlisted fields pass through. |
| Inline JavaScript | Script bodies validate structurally. | Supported by the stateless transform adapter. |
| Inline union | Multi-input list wiring validates. | Supported as a passthrough fan-in. |
| Inline unwind | Path, locators, and options validate. | Supported. The target must upsert. |
| Inline nest | Alias wiring and recursive array, object, flat, and rollup children validate, with `orOn` and `rollup`. | Supported. Flat accepts one-to-one and many-to-one; a live one-to-many match stops with `nest.flat-cardinality-violation`. Rollup fields are recomputed on every send. `orOn` is many-to-one only. |
| Inline aggregate | Grouping, `where`, and rollup functions validate, as does the target `on_full_load` rule. | Supported on one cluster member, with one input that is a source table or a filter/map/union over source tables. A pipeline with an aggregate step cannot be previewed. Embedding a group in a nest has no end-to-end coverage. |
| Inline join | Alias wiring, engine, and SQL validate; a step or pattern alias is refused. | Supported for the SQL subset in `pipeline-semantics.md`, including LEFT JOIN with OR alternatives. Column names are checked at start. |
| Transform `use` | Reusable transform definitions and use sites validate. | Rejected when the pipeline is assembled; inline the transform body. |
| Inline `serve.sync` | Sync elements, modes, DDL, rename, and options validate. | Supported as sink vertices, including `rename.map`. Multiple sync elements fan out from the same upstream. |
| Serve `use` | Reusable serve definitions and use sites validate. | Supported; the definition is expanded under the use site's wiring. |
| `serve.query` | REST, GraphQL, and MCP query declarations validate. | Not executed by the DAG builder. |
| `serve.push` | Push source, topic, format, and options validate. | Not executed by the DAG builder. |
| Inline or reusable view | View wiring, tiered storage, and schema policy validate. | Materialized into the server's configured view store; a reusable view is expanded under the use site's wiring. Unsupported storage tiers fail with `actuation.view-storage-tier-unsupported`. Querying a view through `serve.query` is not executed. |
| `execution` | Parallelism and batch shape validate on every node kind. | Accepted. One processor reads a source whatever is written. |

All examples use the grammar-valid reusable Source shape with Pipeline-level
table selection. `nested-document` combines an array child with a many-to-one
flat child. `grouped-totals` uses an aggregate step and `player-ledger` uses
`soft_delete`, several tables in one array, rollup fields, and `orOn`. Each is
executable by the pinned source runtime once connector config, discovery,
target support, and online validation have succeeded.

### Released CLI

The independently pinned released CLI is v0.8.0. It validates
`filter-and-map`, `nested-document`, and `straight-cdc`, while CI builds the
exact Tapstate source commit above to validate every example. v0.8.0 rejects
`aggregate`, `rollup`, `as: rollup`, `orOn`, and `soft_delete` as unknown
grammar, so `grouped-totals` and `player-ledger` need a build containing the
pinned commit or a later release. Do not reinterpret that version mismatch as
an error in the workspace; report which build validated the workspace.

## Settings and options

The current assembly uses only part of the accepted surface:

| Field group | Runtime status |
|---|---|
| `settings.read_mode` | Used to choose snapshot, tail, or both. |
| `settings.start_from` | Used for a shared incremental tail. Runtime accepts exactly `earliest`, `latest`, or an ISO-8601 instant. Other strings pass offline shape checks but fail when the run is assembled. |
| Omitted `settings.start_from` | The current assembly uses `earliest`, while the generated schema documents `latest` as the grammar default. Set it explicitly when cursor position matters. |
| `settings.error_policy`, `batch_size`, `parallelism`, `schedule` | Accepted and mode-checked where applicable, but ignored by the current execution path. Use node `execution` for parallelism and batching. |
| `source.srs.key`, `retention`, `enabled` | Used by capture and replay-store assembly. |
| `source.srs.schema_evolution`, `queryable` | Accepted but ignored by the current execution path. |
| `source.options` and table `options` | Accepted but ignored by the current capture translation. |
| Table `filter` and `pk` | Refused at assembly with `actuation.source-table-spec-unsupported`. Filter in a pipeline step instead. |
| Transform `options` and `experimental` | Accepted but ignored by the current stateless transform binding. |
| `serve.sync.source`, `write_mode`, `ddl`, `on_full_load`, `rename` | Used by sink assembly. |
| `serve.sync.options` | Accepted but ignored by the current sink binding. |
| View `schema` policy and serve query/push options | Accepted but no runtime surface consumes them. |

Do not convert an ignored field into a run claim. Preserve it as authored intent
and state that the pinned preview does not enforce it.

## CEL runtime gaps

Offline CEL compiles against `op`, `ts`, `src`, `before`, `after`, and `schema`
with dynamic row fields. Apply and `artifact_validate` type-check again against
discovered source models; see "Row expressions" in `pipeline-semantics.md`.

- `now()` is declared by the compiler and therefore validates, but the current
  runtime registers no implementation. Evaluating it fails with a coded
  expression error.
- Arithmetic and comparison with literals on DECIMAL columns are refused at
  apply. Summing one decimal column in a rollup or aggregate is exact.
- Same-type temporal ordering is supported; any other temporal operation is
  refused at apply, and two values with no common order fail at runtime with
  `transform.expression-failed`.
- A missing dynamic field, a missing before image, or an incompatible runtime
  type can still fail during evaluation with `transform.expression-failed`.
  Offline validation cannot infer a connector's row schema.

These limitations do not change map syntax: every computed map rule must still
start with `=`. Without that marker, the authored string is a literal.
