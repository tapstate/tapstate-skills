---
name: dsl-authoring
description: >-
  Author, modify, explain, review, repair, and validate Tapstate tapstate/v1
  workspaces, including Source, Pipeline, Transform, View, and Serve resources.
  Use when creating or changing .tap.yml files, diagnosing DSL errors, tracing
  data-flow semantics, or preparing a pipeline for conditional online execution.
---

# Tapstate DSL Authoring

Build and maintain complete Tapstate workspaces while keeping connector-specific
configuration behind a live metadata boundary. Treat the generated schema and
Tapstate CLI behavior as authoritative contracts; never invent fields or errors.

## Load The Required Context

Read only the references needed for the current request:

- Read `references/resource-model.md` before creating resources, changing
  identities or references, or explaining workspace structure.
- Read `references/pipeline-semantics.md` before authoring or changing Pipeline
  settings, transforms, stream wiring, Serve resources, or View resources.
- Read `references/authoring-patterns.md` for create, modify, review, and repair
  procedures and for selecting a relevant example under `assets/examples/`.
- Read `references/runtime-support.md` before claiming that any resource or
  transform can run, or before attempting apply, start, or observation steps.
- Inspect `references/tapstate-v1.schema.json` when exact fields, required
  properties, types, or enums matter. Do not reproduce the grammar from memory.

Treat the schema as grammar authority and the Markdown references as semantic
guidance. If they disagree, report the mismatch instead of silently choosing a
new contract.

## Follow The Authoring Workflow

1. Determine whether the request is to create, modify, explain, review, repair,
   validate, or run a workspace. Ask only for details that materially affect the
   resource graph or data flow.
2. Inspect every existing `.tap.yml` file that participates in the workspace.
   Resolve resource identities and references before making a change.
3. Plan the complete resource graph: Source resources, the Pipeline and its
   settings, Transform resources, and any Serve or View resources. Keep all
   static DSL semantics inside this skill.
4. Author the smallest complete change. Use `tapstate new --kind <kind>` when a
   CLI-generated resource skeleton is useful. Preserve existing names and IDs
   unless the requested change requires a migration, and update every affected
   reference together.
5. Apply the connector-configuration gate below before adding or changing any
   `source.config` member.
6. Run `tapstate validate <workspace-path>` after writing. Repair errors from the
   coded CLI output, rerun validation, and report the exact remaining codes and
   locations if validation still fails. If the CLI is unavailable, state that
   validation was not run; never fabricate a result.
7. State separately what was authored, what local validation proved, what the
   current runtime supports, and whether online execution was attempted.

## Handle Each Task Mode

- **Create**: Elicit source and destination intent, capture mode, tables, stream
  shape, transforms, and serving needs, then produce the full referenced graph.
- **Modify**: Load the whole affected graph, retain unrelated content, preserve
  opaque connector configuration, and check downstream references after edits.
- **Explain**: Trace resources and streams in execution order. Distinguish field
  syntax, semantic effect, runtime support, and live connector requirements.
- **Review**: Check schema conformance, identity and reference integrity,
  transform and expression semantics, config provenance, secret handling, and
  unsupported runtime claims.
- **Repair**: Use validator codes and the semantic references to make the
  smallest justified correction. Never repair a missing connector contract by
  guessing configuration.
- **Validate**: Run the installed CLI and preserve its coded diagnostics. Do not
  replace, rename, or reinterpret an authoritative error code.

## Enforce The Source Config Boundary

Understand that `source.config` holds connector-owned fields, but do not keep a
static catalog of those member names, defaults, constraints, or secret flags.

- For a new Source, a connector change, or an explicitly requested config
  replacement, obtain current connector metadata through an available live
  Tapstate MCP/catalog capability before writing any config member.
- Never infer config keys from model memory, old examples, another connector,
  or a connector name. Never invent defaults or secret values.
- Preserve existing `source.config` by default. When editing another part of a
  file, do not normalize, reorder, or rewrite that subtree; preserve its keys
  and values exactly.
- Replace existing config only after explicit user intent and a successful live
  metadata lookup for the selected connector version.
- Never echo secret values in chat, explanations, reviews, diffs, diagnostics,
  logs, or summaries. Follow the live metadata's secret classification and the
  user's secure value mechanism.

If live Tapstate MCP/catalog metadata is unavailable, omit `source.config`
instead of synthesizing it. Produce a config-free draft, run offline validation
when the CLI is available, and state clearly that the draft is not runnable.

## Interpret Validation Narrowly

A successful `tapstate validate` result proves only that the installed CLI
accepted the local workspace under its grammar and semantic checks. It does not
prove any of the following:

- canonical serialization or formatting;
- current runtime implementation support;
- connector registration or version compatibility;
- credential validity or external connectivity;
- successful online apply, start, or data processing.

Do not call a workspace runnable until `references/runtime-support.md` permits
the used features, live connector configuration is complete, and the relevant
online checks have actually succeeded.

## Orchestrate Online Only When Available

Attempt online work only when the user requests it and the environment exposes
authoritative Tapstate online-control capabilities. Discover the available
capabilities at runtime; do not assume an MCP server name, tool name, transport,
URL, or hosting model.

When suitable capabilities exist, complete the guarded sequence: obtain live
connector metadata, finish config through the secure input path, validate
locally, apply, start, and observe status, metrics, or logs. Report the result of
each completed step and stop on authoritative coded errors without masking them.

When those capabilities do not exist, stop after the validated offline draft.
Identify the missing live step and do not describe apply, start, or observation
as completed.
