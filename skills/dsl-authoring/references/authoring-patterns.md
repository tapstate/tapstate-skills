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
