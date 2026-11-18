# Authoring Patterns

Apply these patterns after reading `resource-model.md`. Read
`pipeline-semantics.md` for a field's meaning and `runtime-support.md` before
describing a workspace as executable.

## Contents

- Create a workspace
- Modify safely
- Repair a workspace
- Select a pattern
- Lessons from live runs
- Interpret validation honestly

## Create a workspace

1. Capture the intended source connector, read mode, Pipeline table set,
   transforms, output connector, and delivery surface. Treat the Source as a
   reusable connection and capture a separate Source-level table restriction
   only if the user explicitly requests one.
2. Resolve and verify the requested Tapstate workspace before any file or MCP
   operation. If the path cannot be verified, ask for it rather than probing the
   agent's current directory or an installed Skill directory. Then create kind
   directories under it as resources are added. Place each new Source in
   `<workspace-root>/source/<id>.tap.yml` and the Pipeline in
   `<workspace-root>/pipeline/<id>.tap.yml`. Use the equivalent kind directory
   for reusable Transform, View, and Serve resources. These are workspace
   directories initialized by Tapstate authoring, never paths relative to the
   installed Skill.
3. Create one read `source` and one target connection-supplier `source` when a
   sync or push output is required.
4. When live MCP exposes `source_draft`, invoke it directly for every Source
   with a known connector id and the user-supplied structured config. Write its
   canonical YAML response directly to the Source file before any other
   exploration. Do not create an empty Source first, and do not preflight with
   connector list or connector get.
5. Treat a coded `source_draft` contract error as a stop for that Source. Report
   its exact diagnostic and wait for corrected input instead of guessing keys,
   creating a config-free fallback, or moving into pipeline authoring.
6. Omit `source.config` only when that live contract is unavailable. Mark the
   resulting config-free draft as not runnable.
7. Create the Pipeline resource graph with explicit ids and `from` references.
   Put the tables to synchronize in the Pipeline wiring. Do not copy a
   Pipeline's table list into the Source unless the user said the Source itself
   must expose only that subset. If the Source has no `tables`, use the literal
   table names in the Pipeline's first upstream `from` (or an explicit regex
   when the user requested dynamic selection).
8. Choose only grammar-supported fields from `tapstate-v1.schema.json`.
9. Check the requested graph against `runtime-support.md`.
10. Run `tapstate validate path/to/workspace` and fix each coded error. When the
    user asked to run, call `connection_test` and then
    `connection_discover_schema` for every read Source, not for a sink-only
    connection supplier.
11. After discovery succeeds, call `artifact_validate` with the complete YAML
    closure and fix its diagnostics before `artifact_apply`.
12. For an online request, use the exposed MCP tool inventory as the capability
    probe. If a required tool is listed, call it directly; do not report an
    unreachable Server before that call. If it is absent, report the missing
    MCP capability rather than claiming a transport failure.
13. Report separately: offline validation, missing dynamic config, current
   runtime support, and online run state.

For a configured Source, the live draft response is the only starting document.
Use a CLI scaffold only for a config-free static resource and place its generated
file in the resource's kind directory before validation.

## Modify safely

Read the entire workspace before editing. Build a reference map for top-level
ids, Pipeline source ids, step ids, `from` tokens, use references, and serve
target sources.

- Preserve every existing `source.config` mapping byte-for-byte unless the user
  explicitly requests a connector configuration change.
- For a requested config replacement, require current MCP catalog metadata and
  obtain secret values from the user or a secret provider. When `source_draft`
  is available, use it to validate and render the replacement. Never infer a key
  or print a secret.
- Preserve unrelated YAML and metadata. Rename an id only after updating every
  reference to it.
- When inserting a transform, wire its `from` explicitly and update the next
  consumer if the new step is intended to replace an edge.
- Keep reusable bodies pure: change wiring at the Pipeline use site.
- Revalidate the whole workspace, not only the edited file.

If byte preservation of opaque config is important, avoid parse-and-reserialize
workflows that rewrite the source file. Patch only the intended YAML region.

## Repair a workspace

Repair in dependency order:

1. YAML shape and strict field names.
2. Top-level id uniqueness and the no-dot rule.
3. Missing source and reusable-definition references.
4. Pipeline-internal id collisions and `from` closure.
5. Source mode, SRS, schedule, and read-axis compatibility.
6. CEL syntax and result shape.
7. Runtime-support mismatch.

Do not remove an advanced construct merely because the preview runtime lacks
it. Keep a valid requested design and label its runtime status unless the user
asks for a preview-compatible rewrite.

## Select a pattern

### Straight CDC

Use one CDC source, a target connection supplier, and an inline `serve.sync`.
The `straight-cdc` asset is config-free and matches the current static topology,
but it still needs MCP-provided config and live connector checks before a run.

### Filter and map

Put filter before map when the predicate needs original field names. Use `$name`
for rename, `false` for drop, `=CEL` for computation, and an ordinary YAML value
for a literal. Remember that unlisted fields pass through; explicitly drop any
field that must not reach the sink.

The `filter-and-map` asset uses only string comparisons and string-producing CEL
to avoid the current numeric overload limitation.

### Nested documents

Model parent and child streams as aliases in `from`, select the parent with
`root.from`, and make each `embed.on` mapping point from child fields to parent
fields. Use `arrayKey` for stable array element identity. Use `as: flat`
without `path` or `arrayKey` when one child row should contribute fields
directly to the parent. Flat accepts one-to-one and many-to-one relationships,
refuses one-to-many at runtime, and never resolves field collisions by
declaration order. Map child fields to disjoint public names before Nest and
drop internal join/key names after it. The `nested-document` asset combines an
array child with a many-to-one flat child.

Give every many-to-one object or flat embed a `key` naming the column it is
looked up by when that column is not the referenced table's discovered key.

### One array from several tables

Tag each table in its own `map`, merge the tagged streams with `union`, embed
the union, and include the tag in `arrayKey` (`[kind, id]`). For shards, one
`map` with a quoted regex `from` and `fields: { shard: "=src" }` does both. Keep
the root fed by a source table or a map/filter over one. The `player-ledger`
asset shows the tagged union.

### Totals inside a document

Add `rollup:` to the array embed whose rows are being summarized, or use
`as: rollup` when the rows should not appear in the document. Use `default: 0`
only where an empty level should read as zero; a missing reference still gives
null. Guard optional references inside the expression with `has()`. Keep the
document under about 1 MB of state. The `player-ledger` asset counts movements,
sums a filtered bucket, and takes a maximum.

### Totals per group

Use an `aggregate` step when the group is not a document (per region, per day,
or one grand total). Put the predicate in `where:` rather than an upstream
filter, declare `key` when the input has no primary key, and set
`on_full_load: clear` on every target that reads the step. Plan a definition
change as a rebuild. The `grouped-totals` asset shows the full shape.

### Kept deletes

Add a `map` with `soft_delete: <column>` on exactly the stream whose deletes
should be kept: before one target, before a nest root, or before one embed.
Require full before images on delete from the source when it feeds a nest. The
`player-ledger` asset marks deleted players.

### A reference named two ways

Use `orOn` on a many-to-one nest embed, or `LEFT JOIN ... ON a = b OR c = d` in
a join, when one row is named by one column on some parents and another column
on the rest. Order alternatives by preference; the first live match wins.

### Reusable logic

Promote repeated transform, view, or serve bodies to top-level resources. Keep
all upstream wiring at each use site. Use explicit ids when two instances of the
same definition appear in one Pipeline.

## Lessons from live runs

These came from rehearsing a multi-table MongoDB nest end to end. Apply them
when the same shape appears.

- **Declare `key` on a shared-row embed.** MongoDB discovery only knows `_id`,
  so an object embed looked up by another column (`on: { AttractionNumber:
  AttractionNumber }`) needs `key: [AttractionNumber]`, or start fails with
  `nest.embed-target-not-parent-key`.
- **Turn on before images when anything reads `before`.** For MongoDB this is
  the connector's pre-image setting plus collection pre-images; for MySQL
  `binlog_row_image=FULL`; for PostgreSQL `REPLICA IDENTITY FULL`. Without them
  an update carries no `before`, and a filter such as
  `before.RankingID != after.RankingID` fails with
  `transform.expression-failed`. The connector key for the setting comes from
  the live connector contract, never from memory.
- **Guard list macros with `has()`.** A nest drops an empty array field, so
  `after.emails.exists(...)` fails once a document's last element is deleted.
  Write `has(after.emails) && after.emails.exists(...)`.
- **Change a rule on a running pipeline with stop, apply, start.** Applying an
  edit to a running or failed pipeline is accepted, but the running job keeps
  the old definition, and `stop` or `pause` is then refused with
  `io.workload-claim-fenced` until the previous version is applied again. Stop
  first (a plain stop clears accumulated state, so the restart is a full
  reload), apply the change, then start. Tell the user before stopping; it is
  their decision.
- **Do not promise that validation catches every typo.** An alias typo in
  `from` is refused offline with `dsl.missing-reference`, but a misspelled
  column in `on:` is not caught for a source without a column model such as
  MongoDB: the pipeline runs and the embed stays empty.
- **`write_mode: append` is not an event log on every target.** MongoDB keeps
  the key index, so an append target still holds the latest row per key.
- **Change events on computed nest fields are not available yet.** A filter
  comparing `before` and `after` of a field computed on a nest output is
  refused at apply with `dsl.row-expression-type-unknown`.

## Interpret validation honestly

`tapstate validate` success means the installed CLI accepted the workspace's
offline grammar, references, CEL compilation, and bundled capability checks. It
does not prove:

- that YAML is already in canonical serialized form;
- that omitted connector config exists or is correct;
- that credentials connect;
- that the server has the same connector catalog;
- that the preview runtime implements every accepted resource;
- that apply, start, status, metrics, logs, or stop succeeded.

Only report those later states after their own authoritative operation returns
success.

## Handoff to online control

`source_draft` is local-authoring support, not persistence. After all resources
are written and local validation passes, call `artifact_validate` with the
complete Source/Pipeline closure. For an online run, test and discover every
read Source first; discovery is the source-model input used by the preview sink
target resolver and by row-expression checks. Fix `artifact_validate`
diagnostics, then apply that same closure through `artifact_apply` only when the
user asks for online execution. Start the Pipeline only after that apply
succeeds. A partial Source apply creates an online state that no longer matches
the workspace and is not an authoring workflow. A local CLI result is not an MCP
availability check; call exposed MCP tools before declaring the Server unreachable.
