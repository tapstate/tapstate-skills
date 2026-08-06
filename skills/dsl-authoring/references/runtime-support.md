# Current Preview Runtime Support

This matrix is pinned to Tapstate commit
`1ff03dce10af298cb3f1925e352050b36dcdeb55`. The generated schema describes the
complete `tapstate/v1` grammar; this file describes the narrower runtime wired at
that commit. Update both this matrix and the repository upstream lock when the
baseline changes.

Offline validation and runtime execution are independent claims. All three
bundled examples pass offline validation, but none is runnable until MCP or
another trusted live source supplies the connector-specific `source.config`
members.

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

## Capability matrix

| Surface | Grammar and offline validation | Current preview runtime |
|---|---|---|
| Read Source | All source modes, literal/object/regex table selectors, and multiple tables are authorable. | The runtime path expects one read Source selecting exactly one non-regex table. Literal and object-form names reach capture; omitted, multiple, and regex tables fail during assembly. |
| Target Source | A source may omit mode and tables when used only as a connection supplier. | Supported by inline sync; connector and config are used to open the sink. |
| Source references | Table, step, source-qualified table, and regex references validate. | The linear builder resolves simple literal table or step names. Qualified and regex `from` references are not carried. |
| Inline filter | CEL boolean expressions validate. | Supported. Row events are filtered; DDL bypasses. |
| Inline map | Rename, drop, literal, and `=CEL` rules validate. | Supported over `after`; deletes and DDL bypass. Unlisted fields pass through. |
| Inline JavaScript | Script bodies validate structurally. | Supported by the stateless transform adapter. |
| Inline union | Multi-input list wiring validates. | Supported as a single-lane passthrough fan-in. |
| Inline nest | Alias wiring and nest trees validate. | Rejected by the linear DAG builder as a stateful transform. |
| Inline join | Alias wiring, engine, and SQL validate. | Rejected by the linear DAG builder as a stateful transform. |
| Transform `use` | Reusable transform definitions and use sites validate. | Unresolved use steps are rejected; no definition-resolution phase is wired. |
| Inline `serve.sync` | Sync elements, modes, DDL, rename, and options validate. | Supported as sink vertices. Multiple sync elements fan out from the same upstream. |
| Serve `use` | Reusable serve definitions and use sites validate. | Rejected; no definition-resolution phase is wired. |
| `serve.query` | REST, GraphQL, and MCP query declarations validate. | Not executed by the DAG builder. |
| `serve.push` | Push source, topic, format, and options validate. | Not executed by the DAG builder. |
| Inline or reusable view | View wiring, tiered storage, and schema policy validate. | Not materialized or queried by the current data plane. Reusable view resolution is therefore also unavailable. |

The `straight-cdc` and `filter-and-map` examples use the supported static shape.
The `nested-document` example deliberately proves that grammar-valid nest
authoring is distinct from runtime support and must be reported as unavailable
on this preview.

## Settings and options

The current assembly uses only part of the accepted surface:

| Field group | Runtime status |
|---|---|
| `settings.read_mode` | Used to choose snapshot, tail, or both. |
| `settings.start_from` | Used for a shared incremental tail. Runtime accepts exactly `earliest`, `latest`, or an ISO-8601 instant. Other strings pass offline shape checks but fail when the run is assembled. |
| Omitted `settings.start_from` | The current assembly uses `earliest`, while the generated schema documents `latest` as the grammar default. Set it explicitly when cursor position matters. |
| `settings.error_policy`, `batch_size`, `parallelism`, `schedule` | Accepted and mode-checked where applicable, but ignored by the current execution path. |
| `source.srs.key`, `retention`, `enabled` | Used by capture and replay-store assembly. |
| `source.srs.schema_evolution`, `queryable` | Accepted but ignored by the current execution path. |
| `source.options`, table `filter`, `pk`, and table `options` | Accepted but ignored by the current capture translation. Only the selected table name is carried. |
| Transform `options` and `experimental` | Accepted but ignored by the current stateless transform binding. |
| `serve.sync.source`, `write_mode`, `ddl` | Used by sink assembly. |
| `serve.sync.rename` and `options` | Accepted but ignored by the current sink binding. |
| View storage/schema and serve query/push options | Accepted but no runtime surface consumes them. |

Do not convert an ignored field into a run claim. Preserve it as authored intent
and state that the pinned preview does not enforce it.

## CEL runtime gaps

Offline CEL compiles against `op`, `ts`, `src`, `before`, `after`, and `schema`.
Nested row fields are dynamic, so some overload mistakes surface only at
evaluation.

- Keep preview-safe examples string-only. Numeric values from row maps can fail
  CEL numeric overload dispatch even when the expression passed offline checks.
- `now()` is declared by the compiler and therefore validates, but the current
  runtime registers no implementation. Evaluating it fails with a coded
  expression error.
- A missing dynamic field or incompatible runtime type can still fail during
  evaluation. Offline validation cannot infer a connector's row schema.

These limitations do not change map syntax: every computed map rule must still
start with `=`. Without that marker, the authored string is a literal.
