# Pipeline Semantics

Use the vendored schema for exact YAML shapes. Use this reference for dataflow
and execution meaning, then consult `runtime-support.md` before making a run
claim.

## Contents

- Read settings
- Wiring forms
- Reusable `use`
- Transform bodies
- Row expressions
- Node execution
- Serve
- View

## Read settings

Pipeline `settings` apply to the task, not to a source:

| Field | Semantic contract |
|---|---|
| `error_policy` | Requested record-error behavior: `fail`, `skip`, or `dead_letter`. |
| `batch_size` | Requested records per batch. |
| `parallelism` | Requested parallel workers. |
| `schedule` | Cron-style schedule, legal only for a bounded read. |
| `read_mode` | `snapshot_and_cdc`, `cdc_only`, or `snapshot_only`; meaningful for CDC sources. |
| `start_from` | Cursor for an incremental tail: `earliest`, `latest`, or an ISO-8601 instant. |

`read_mode` defaults in the grammar to `snapshot_and_cdc`. `snapshot_only` is
invalid for a pure stream. `schedule` requires a bounded read. `start_from`
requires an incremental tail and therefore cannot accompany a bounded-only
read. Runtime defaults and restrictions may be narrower than these grammar
rules.

## Wiring forms

Streaming transforms (`filter`, `map`, `js`, `union`, `unwind`, and
`aggregate`) use scalar or list `from` wiring. The normalized meaning is always a non-empty list:

```yaml
from: orders
```

```yaml
from: [orders, returns]
```

A quoted `/.../` regular expression in a streaming `from` selects every source
table whose name matches, for example `from: "/^PlayerSession[0-9]$/"`. Quote
the pattern in YAML; inside a flow sequence the brackets of `[0-9]` are
otherwise read as YAML.

`nest` and `join` require an explicit alias map. Body fields refer to aliases,
not directly to workspace resources:

```yaml
from:
  customer: customers
  order: orders
```

A `join` alias must name one source table: never another step, and never a
`/.../` pattern (`dsl.join-input-not-a-table`, `dsl.join-input-is-a-pattern`).
A `nest` alias may name a source table or a step.

The table names in these Pipeline `from` clauses are the Pipeline's selection
for this synchronization. Keep the Source reusable by omitting `source.tables`
unless the user explicitly requires a Source-level allowlist; a Source-level
allowlist constrains every Pipeline that references that Source.

For streaming steps after the first, omitted `from` means the preceding step.
An omitted `from` on an inline view or serve block also means the preceding
transform, or the view for a serve block that follows one. Prefer explicit
wiring when editing or when a future reorder could change the natural source.

## Reusable `use`

A reusable definition separates logic from wiring:

```yaml
version: tapstate/v1
kind: transform
id: public_rows
type: filter
expr: "op != 'd'"
```

```yaml
transforms:
  - id: current_rows
    use: public_rows
    from: orders
```

The same rule applies to view and serve definitions: definitions never carry
`from`; Pipeline use sites always do. Scalar entries are sugar for a use
reference with natural-order wiring, but the first transform must provide an
explicit upstream.

## Transform bodies

### Filter

`filter.expr` is a CEL boolean predicate over the event envelope. Row events
whose result is false are dropped. Non-row DDL events bypass the filter so a
predicate cannot silently swallow schema evolution.

A filter keeps a live subset, not only a stream of matches:

- an update that stops matching is evaluated against the previous row as
  `after`; if that row matched, the filter sends a delete of the previous row
  so the downstream copy is removed;
- an update whose previous and current rows both fail produces nothing;
- a source delete keeps its envelope: the removed row is in `before` and
  `after` is empty. To keep matching deletes, write
  `"op == 'd' ? before.status == 'x' : after.status == 'x'"`.

Both behaviours need before images on update and delete. A predicate that
reads a missing before-image field fails with `transform.expression-failed`;
the filter never fetches old rows from the source. `op != 'd'` deliberately
suppresses deletes, so such a filter no longer describes a live copy.

### Map

`map.fields` is an ordered mapping of output names to field rules:

| Authored value | Meaning |
|---|---|
| `$field` | Rename `field` to the output name and consume the old name. |
| `false` | Drop the same-named field. |
| `=expression` | Compute the output with CEL. The leading `=` is mandatory for a computed map rule. |
| Any other YAML value | Add or replace the output with that literal value. |

Rules run in declaration order. Declared outputs come first, followed by
unlisted input fields in input order. An output wins over a pass-through field
with the same name. A rename consumes its source name. A rename whose source is
missing produces nothing and does not suppress an unrelated same-named input.
Map applies to the `after` row image; events with no `after` image, including
deletes and DDL, pass through unchanged.

`map.soft_delete` names a boolean column that turns deletes into updates. With
it set, an insert or snapshot row leaves with the column `false`, an update's
before and after images both carry `false`, a delete leaves as an **update** of
the last row image with the column `true`, and DDL is unchanged. `fields` may
then be omitted; when present, its rules run on the marked row too, and the
`soft_delete` column may not also be declared under `fields`
(`dsl.illegal-value`). Place the step exactly where the marking should apply:

- just before one target's `serve.from` or view `from`, to mark rows for that
  target only;
- on the stream feeding a nest root, to keep a deleted root's document with the
  marker at its top level and its children still embedded;
- on the stream feeding one embed, to keep deleted children in their array as
  marked elements. An embed fed directly by its table still loses elements.

The marked row holds only what the delete carried, so a soft-delete step that
feeds a nest needs full before images on delete; otherwise the element or root
is replaced by its key and the marker. A fresh full load has no deleted rows to
mark, and nothing ages marked rows out.

### JavaScript

`js.script` is the GraalVM escape hatch and sees all events, including DDL. The
script declares `process(record, ctx)` and may declare `filter(record)`.
`record` has the same `op`, `ts`, `src`, `before`, `after`, and `schema` fields as
the envelope. `ctx.emit(record)` fans out in call order and `ctx.log(message)`
logs; there is no state or lookup surface. Emitted records are followed by the
non-null return value. Return null with no emissions to drop the event. Use JavaScript
only when filter and map cannot express the change.

### Union

`union` explicitly merges multiple upstream streams. It has no body fields; its
multi-item list `from` defines the fan-in.

### Unwind

`unwind` turns one parent row into one row per element of the array at `path`
(one level; no dotted paths). Each output keeps the parent's fields and
replaces the array with one element. At least one locator is required:
`element_key` copies an identity field from each element into a top-level
column of the same name, and `include_array_index` adds a zero-based ordinal
column. The output key is the parent key plus the locator. Generated names must
not collide with parent columns or the array path
(`dsl.unwind-column-already-exists`). The target must upsert
(`dsl.unwind-needs-an-upsert-target`). `preserve_null_and_empty_arrays`,
`element_type`, and `source_id_field` are optional; the last one only applies
when a target such as MongoDB needs a separate generated identity.

### Nest

`nest` materializes related streams into documents. Its alias-map `from` names
the available streams. `root.from` selects the parent alias; `root.key` defines
the parent upsert key; `root.mode` declares its write shape. Each `embed`:

- selects a child alias with `from`;
- maps child fields to parent fields through `on`;
- chooses `array`, `object`, or `flat` with `as`;
- writes array/object at `path`; flat must omit both `path` and `arrayKey` and merges one row into the parent;
- may define `key`, `arrayKey` for arrays, `ignoreUpdates`, and `trackKeyChanges`;
- may recursively contain more `embed` entries.

Flat supports one-to-one children and many-to-one shared referenced rows. Several flat siblings may
merge into one parent when their output fields are disjoint. A second live row under one parent is a
one-to-many relationship and stops with `nest.flat-cardinality-violation`; use `array` for that shape.
Fields may not overlap the parent, another flat child, or an object/array path. Discovered models catch
known conflicts before start and actual rows catch dynamic or model-less conflicts at runtime with
`nest.flat-field-conflict`. Rename or drop fields in an upstream map, retaining any join/key fields until
after Nest has used them.

`primary_key`, `order: main_first|sub_first`, capacity settings, and key-tracking switches provide
additional stateful materialization controls.

#### Embed identity

An embed over a step's output (a `map`, `union`, or `js`) has no table key to
fall back on and must declare `arrayKey`, or start fails with
`nest.array-key-unresolvable`. An embed whose rows are themselves parents of
deeper embeds also declares `key` with the same fields. A many-to-one object or
flat embed whose `on` does not cover the referenced table's discovered key must
declare `key` naming the column it joins on; MongoDB discovery knows only `_id`,
so `key: [AttractionNumber]` on an embed joined by `AttractionNumber` avoids
`nest.embed-target-not-parent-key` at start.

#### Several tables in one array

An embed reads one stream. To feed one array from several tables, tag each
table's rows in its own `map` (`fields: { kind: points }`), merge the tagged
streams with `union`, embed the union, and set `arrayKey: [kind, id]`. For one
table split into shards, one `map` with a regex `from` and `fields: { shard:
"=src" }` both merges and tags. Leaving the tag out of `arrayKey` is not
reported: rows with equal ids in different tables silently become one element.
Keep the root's `from` a source table, or a `map`/`filter` over one; a root fed
by a `union` or `js` step reaches the target without a key and appends a new
document on every change.

#### Matching on one column or another: `orOn`

`orOn` lists further alternatives to `on`, each a child-field to parent-field
map written like `on`:

```yaml
- from: loc
  on: { locn_code: locn_code }
  orOn:
    - { hub_locn_id: hub_locn_id }
  as: object
  path: location
```

Alternatives are tried in written order and the first one with a live match
supplies the row. A NULL in an alternative's fields matches nothing. `orOn` is
only for an embed that points at one shared row (many-to-one); an alternative
that matches on the parent's own key is refused with
`nest.or-on-not-a-reference`. Moving documents when a referenced row's matching
columns change needs before images on update. Row-level data validation does
not cover `orOn` embeds yet.

#### Rollup fields

`rollup` on an `as: array` embed, or on an `as: rollup` embed, computes fields
over the embed's live elements and writes them on the embed's parent:

```yaml
- from: item
  on: { order_id: id }
  as: array
  path: items
  arrayKey: [id]
  rollup:
    item_count: { count: true }
    total_qty: { sum: $qty, default: 0 }
```

Each field names exactly one function: `count` (`true`, or an expression
counting non-null values), `count_distinct`, `sum`, `avg`, `min`, `max`, or
`first`/`last` with `{ value: <expr>, by: <expr> | [<expr>, ...] }`. Optional
qualifiers are `filter` (an element contributes only while it holds), `type`
(`int64`, `decimal`, or `double`, converted exactly or refused), and `default`
(written when nothing contributed; it never replaces the null of a missing
reference). Arguments are `$column` or `=expression` over the element as
`after`, including fields its flat embeds merge in, references it resolves to,
and rollup fields of its own child levels.

`as: rollup` keeps the level's rows only to compute rollup fields: it takes
`arrayKey`, omits `path`, and must declare at least one field
(`dsl.rollup-level-without-fields`). Rollup on `as: object` or `as: flat` is
refused (`dsl.rollup-on-single-row-embed`). A rollup name shares one namespace
per level with columns, embed paths, and flat fields
(`nest.rollup-field-conflict`).

Numbers are exact or refused: `sum`/`avg` need a discovered numeric type
(`nest.rollup-type-unknown`); integer sums stay 64-bit, decimal sums keep their
scale, and only `avg` rounds (half-even). A field that reads a reference that
is missing for any live element is null, meaning incomplete, not zero. Every
send recomputes rollups from the live elements, so keep a document with rollup
fields under about 1 MB of state; `nest.rollup-level-wide` warns at apply and at
runtime. A total over a group that is not a document belongs in `aggregate`.

### Aggregate

`aggregate` groups the rows of one input and keeps one current output row per
group, deleting a group when its last row leaves:

```yaml
- id: sales_by_region_day
  type: aggregate
  from: [orders]
  key: [id]
  where: "=after.status != 'cancelled'"
  groupBy:
    region: $region
    day: "=after.created_day"
  fields:
    order_count: { count: true }
    revenue: { sum: $amount, default: 0 }
    latest_status: { last: { value: $status, by: $updated_at } }
```

- `from`: exactly one input, a source table or a `filter`, `map`, or `union`
  over source tables. A nest or join input is refused
  (`aggregate.input-not-supported`).
- `key`: columns identifying one input row; defaults to the input's primary key
  and is required when it has none (`aggregate.key-required`).
- `where`: prefer it over an upstream `filter`. It is evaluated by the step, so
  a row that stops matching leaves its group whatever the source sends. An
  upstream filter on a source without guaranteed before images is refused
  (`aggregate.upstream-filter-unsafe`).
- `groupBy`: output key fields in declared order, each `$column` or
  `=expression`. Absent gives one total row keyed by a column named `aggregate`
  holding the step id; such a step cannot name a field `aggregate`.
- `fields`: the rollup functions and qualifiers above, unchanged.

Every target reading an aggregate must declare `on_full_load: clear` (or
`fail`), because a full load rebuilds the state
(`aggregate.rebuild-needs-clear`). After changing `from`, `key`, `where`,
`groupBy`, or `fields`, start refuses the old state with
`aggregate.definition-changed`; rebuild from a full load. A group that cannot
be computed exactly keeps its last good values and carries a
`__tap.aggregate.<step>` marker with the failing code. State is at most
`2 + m` entries per input row (m = number of `min`, `max`, `first`, `last`,
`count_distinct` fields) and holds input values. Status and metrics report
`aggregateGroups.<step>` and related keys. `tapstate aggregate <pipeline>
<step> [--group k=v,...] [--repair]` checks or repairs groups against stored
rows.

### Join

`join` materializes a flat wide table. Its alias-map keys become SQL relation
names. `engine` selects the query engine and `sql` defines the result.

Use one driving fact table with dimensions joined directly to it by equality.
Each dimension must be unique on its complete join key; there is no fan-out and
no uniqueness guard. INNER and LEFT JOIN with `AND`-ed qualified equalities are
supported; RIGHT JOIN is normalized to LEFT. A `LEFT JOIN ... ON a = b OR c = d`
onto one table tries alternatives in order and the first match wins; read that
table only through qualified columns. WHERE, GROUP BY, DISTINCT, subqueries,
ORDER BY, FULL OUTER, CROSS, NATURAL, USING, and non-equality joins are refused.
For upsert sync, project every primary-key column of the driving table as a
direct column (`actuation.join-output-key-not-published`). Column names are
checked against discovered schemas at start (`actuation.join-sql-invalid`).

## Row expressions

Filter `expr`, computed map rules, aggregate `where`/`groupBy`/arguments,
rollup arguments, and push-format expressions are CEL row expressions. Where a
field takes either a column or an expression (map rules, `groupBy`, rollup and
aggregate arguments, `where`, rollup `filter`), write `$column` for a bare read
or `=` followed by the expression; filter `expr` is always an expression and
takes no `=`.

Envelope-rooted expressions bind:

| Binding | Type and meaning |
|---|---|
| `op` | String operation symbol: `i`, `u`, `d`, `r`, or `ddl`. |
| `ts` | Integer event timestamp. |
| `src` | String source stream name. |
| `before` | String-keyed dynamic map; empty when absent. |
| `after` | String-keyed dynamic map; empty when absent. |
| `schema` | String-keyed dynamic map; empty when absent. |

The compile environment includes the CEL standard library, standard macros
such as `has`, `exists`, `all`, `map`, and `filter`, plus declared `now()`.
Offline (`tapstate validate`) field types remain dynamic, so validation catches
unknown envelope names and syntax but cannot prove every field access or
overload will evaluate.

Apply and `artifact_validate` check expressions again against the discovered
source models, which is why discovery comes first
(`dsl.row-expression-needs-discovery`). A typed column then fails to compile
when the operation cannot hold its type exactly
(`dsl.row-expression-type-unsupported`), and an operation on a column whose type
nothing resolved can be refused with `dsl.row-expression-type-unknown`. For
example, a filter `before.Tags != after.Tags` over a field that a map computed
on a nest output is refused at apply. Columns a sampled model has not seen yet stay
dynamic and compile.

- Decimal and temporal columns are opaque: they pass through unchanged, but
  arithmetic on them is refused. Two values of the same temporal type (DATETIME,
  DATE, or TIME) may be ordered with `<`, `<=`, `>`, `>=`, for example
  `after.updated_at >= before.updated_at`; mixed temporal types, strings, and
  numbers are refused.
- `has()` guards a field that may be absent. A nest drops an empty array field,
  so a downstream `after.emails.exists(...)` fails when the last element goes;
  write `has(after.emails) && after.emails.exists(...)`.
- Comparing `before` and `after` needs before images from the source.

`tables[].filter` is not a row expression and is not supported by the runtime;
see `runtime-support.md`.

## Node execution

Sources, inline and reusable transforms, view blocks, and `serve.sync` elements
accept `execution`:

```yaml
execution:
  parallelism: 4
  batch:
    max_records: 1024
    max_wait: 50ms
```

`parallelism` is the target total processors for the node across the cluster
(1 to 1024; default 1 for sources and transforms, 4 for view and sync sinks).
`batch.max_records` is 1 to 65536 (default 1024) and `batch.max_wait` is a
whole number of `ms`, `s`, or `m`, at most 60s (default `0ms`). One processor
reads a source today whatever is written.

## Serve

An inline Pipeline serve block or reusable `serve` body can declare three
surfaces:

- `sync`: table-model writes. `source` names a target connection supplier;
  `write_mode` is `upsert` or `append`; `ddl` is `apply`, `ignore`, or `fail`.
  Rename precedence is explicit `rename.map`, otherwise case conversion, then
  literal prefix and suffix. A sync `id` is required when a query backend names
  it.
- `query`: pull endpoints of type `rest`, `graphql`, or `mcp`. With no backend,
  the query is parallel egress from a view store; `backend` names a sync element
  when the sink serves the API.
- `push`: event-stream egress. `source` names a target connection supplier;
  `topic` selects the destination. `format` is either a whole-payload CEL value
  or an ordered field-rule map. Do not add a connector `type` field.

## View

A view is a materialized, queryable output. Inline views add `from`; reusable
view definitions omit it. `primary_key` identifies records. `storage` can
declare hot memory (`ttl`), warm database (`collection`, `indexes`), and cold
lake (`partition_by`) tiers. `schema.enforce` requests strict enforcement and
`schema.evolution` declares the evolution policy. Views accept `write_mode`
(default `upsert`), `on_full_load` (default `append`), and `execution`.
