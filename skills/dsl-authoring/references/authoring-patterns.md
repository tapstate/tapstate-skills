# Authoring Patterns

Apply these patterns after reading `resource-model.md`. Read
`pipeline-semantics.md` for a field's meaning and `runtime-support.md` before
describing a workspace as executable.

## Contents

- Create a workspace
- Modify safely
- Repair a workspace
- Select a pattern
- Interpret validation honestly

## Create a workspace

1. Capture the intended source connector, read mode, table set, transforms,
   output connector, and delivery surface.
2. Create one read `source` and one target connection-supplier `source` when a
   sync or push output is required.
3. Omit `source.config` unless live MCP catalog data supplies its exact members.
4. Create the Pipeline resource graph with explicit ids and `from` references.
5. Choose only grammar-supported fields from `tapstate-v1.schema.json`.
6. Check the requested graph against `runtime-support.md`.
7. Run `tapstate validate path/to/workspace` and fix each coded error.
8. Report separately: offline validation, missing dynamic config, current
   runtime support, and online run state.

Start from a CLI scaffold when useful:

```sh
tapstate new --kind source --id src_orders --out ./workspace
tapstate new --kind pipeline --id orders_to_archive --out ./workspace
```

The command uses `--kind`; `tapstate new source` is not equivalent.

## Modify safely

Read the entire workspace before editing. Build a reference map for top-level
ids, Pipeline source ids, step ids, `from` tokens, use references, and serve
target sources.

- Preserve every existing `source.config` mapping byte-for-byte unless the user
  explicitly requests a connector configuration change.
- For a requested config replacement, require current MCP catalog metadata and
  obtain secret values from the user or a secret provider. Never infer a key or
  print a secret.
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
fields. Use `arrayKey` for stable array element identity. The
`nested-document` asset is grammar-valid but is not executable by the pinned
preview runtime.

### Reusable logic

Promote repeated transform, view, or serve bodies to top-level resources. Keep
all upstream wiring at each use site. Use explicit ids when two instances of the
same definition appear in one Pipeline.

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
