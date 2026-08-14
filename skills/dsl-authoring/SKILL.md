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
2. Resolve the local Tapstate workspace from the request or the Tapstate CLI
   workdir settings, then inspect every participating `.tap.yml` file. Never
   resolve it relative to this installed Skill. For a new resource, create the
   required kind directory under that workspace: `<workspace-root>/source/`,
   `<workspace-root>/pipeline/`, `<workspace-root>/transform/`,
   `<workspace-root>/view/`, or `<workspace-root>/serve/`. This is the Tapstate
   workspace layout that `tapstate new` initializes, not a project-binding or
   installer destination. Do not move existing files merely to change layout.
3. Plan the complete resource graph: reusable Source connection definitions,
   the Pipeline's table selection and settings, Transform resources, and any
   Serve or View resources. Keep all static DSL semantics inside this skill.
4. Apply the connector-configuration gate below before authoring a new Source
   or changing any `source.config` member.
5. Author the remaining smallest complete change. For config-free static
   resources, use `tapstate new --kind <kind>` only when its output can be
   placed in the correct kind directory without an extra skeleton-and-rewrite
   cycle. Preserve existing names and IDs unless the requested change requires a
   migration, and update every affected reference together.
6. Run `tapstate validate <workspace-path>` after writing. Repair errors from the
   coded CLI output, rerun validation, and report the exact remaining codes and
   locations if validation still fails. If the CLI is unavailable, state that
   validation was not run; never fabricate a result.
7. State separately what was authored, what local validation proved, what the
   current runtime supports, and whether online execution was attempted.

## Keep The First Actions Deterministic

Workspace resolution is a blocking gate for create, modify, and run requests:

1. Derive `<workspace-root>` from the path supplied by the user or from the
   Tapstate CLI workdir settings. If the user identifies a current directory,
   verify that directory itself; never substitute the installed Skill directory
   or the agent process working directory.
2. Verify the resolved root before creating files or calling MCP. If no root can
   be verified, ask for its path instead of probing unrelated directories.
3. After verification, inspect only participating workspace files and then use
   the shortest applicable operation sequence:
   `resolve -> inspect -> source_draft -> write -> author -> validate -> test/discover -> artifact_validate -> artifact_apply -> start/observe`.
4. Do not run generic repository discovery, `git status`, generic CLI help,
   no-op shell commands, or connector list/get calls as preflight for a known
   connector. After a successful `source_draft`, write its returned YAML to the
   final Source path immediately before continuing.
5. Never run `echo`, `printf`, `true`, `:`, or `sleep` only to narrate progress.
   Treat the shell as an implementation channel and put progress updates in the
   assistant message.
6. For multiple Sources, complete one atomic unit at a time:
   `source_draft -> write source/<id>.tap.yml -> continue`. Do not interleave
   source creation with connector discovery, pipeline authoring, or status
   narration.
7. When the user asks to run, call `connection_test` and then
   `connection_discover_schema` for every read Source before `artifact_validate`.
   Do not discover a sink-only connection supplier. After discovery succeeds,
   call `artifact_validate` immediately with the complete YAML closure. Fix its
   diagnostics in the workspace and call it again before `artifact_apply`; do
   not insert generic shell or CLI exploration between these calls.
8. Determine online capability from the current session's exposed MCP tool
   inventory. If a required tool is exposed, call it before reporting any
   server problem. Do not infer MCP reachability from a local CLI result, shell
   probe, file inspection, or a previous message.

This gate keeps authoring work inside the requested workspace and prevents an
exploratory command from becoming a substitute for a required authoring step.

## Handle Each Task Mode

- **Create**: Elicit source and destination intent, capture mode, the tables
  this Pipeline should consume, stream shape, transforms, and serving needs,
  then produce the full referenced graph. Keep Source definitions reusable:
  omit `source.tables` unless the user explicitly restricts that Source to a
  subset of tables; put the requested table selection in Pipeline `from`
  wiring.
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
  replacement, use the available live `source_draft` capability with the chosen
  connector and the user-supplied structured config. It validates against the
  current connector contract and returns canonical Source YAML without creating
  an artifact or audit record.
- A request to synchronize particular tables is Pipeline intent, not Source
  scope. Do not add those table names to the new Source just because they occur
  in the request; leave `source.tables` absent unless the user explicitly says
  that the Source itself must expose only those tables.
- Treat a coded `source_draft` contract error as an authoritative stop for that
  Source. Report the exact missing or invalid field and wait for corrected input;
  do not guess a connector key, create a config-free fallback, or continue to
  that Source's pipeline or run phase.
- When the request supplies a connector identifier, use it directly in one
  `source_draft` request. Do not first call connector list or connector get to
  confirm an already known id. Query the catalog only when the id is unknown,
  the draft operation reports a connector-contract error, or the user asks to
  compare connectors.
- Write the returned YAML directly to
  `<workspace-root>/source/<id>.tap.yml`, creating that workspace kind directory
  when needed. Do not first write a config-free Source and then revise it. The
  returned document may contain user-supplied secret values; write it to the
  requested local workspace but do not reproduce those values in chat, logs,
  diffs, or summaries.
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

If the required live Tapstate MCP/catalog tool is not exposed in this session,
omit `source.config` instead of synthesizing it. Produce a config-free draft, run
offline validation when the CLI is available, and state clearly that the draft
is not runnable. Never use this fallback while the MCP tool is exposed but
uncalled.

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

The exposed MCP tool inventory is the availability probe. A direct MCP call that
returns a transport error is the only basis for reporting that the Server is
unreachable. If a required tool is listed, invoke it directly and preserve its
structured result or error. If the tool is absent, report the missing MCP
capability, not a guessed Server outage.

When suitable capabilities exist, complete the guarded sequence: draft each
configured Source through `source_draft`, write the returned YAML into the local
workspace, validate locally, test and discover every read Source, call
`artifact_validate` with the complete resource closure, apply that same closure
through `artifact_apply`, then start and observe the requested Pipeline.
`source_draft` and `artifact_validate` do not persist an artifact;
`connection_discover_schema` persists only the latest observed source model;
`artifact_apply` is the artifact persistence handoff in this flow. Report the
result of each completed step and stop on authoritative coded errors without
masking them.

When those capabilities do not exist, stop after the validated offline draft.
Identify the missing live step and do not describe apply, start, or observation
as completed.

## Change Or Remove An Already-Applied Resource

Every sequence above creates resources. A resource that has already been applied
differs in one way that governs each step here: the stored version can have moved
since this session last read it, and whoever moved it is not in this
conversation. Editing and removal therefore both name the version they act on.

Availability is judged exactly as for any other online capability, from the
current session's exposed MCP tool inventory. When `artifact_get` or
`artifact_delete` is absent, report the missing capability and stop; never
describe an edit or a removal as attempted.

`artifact_get` takes an `id` and returns that resource's canonical YAML together
with the `contentHash` of exactly those bytes. That hash is the only supported
precondition value. Never compute one locally, carry one over from an earlier
session, or reuse one across resources. The two sequences are:

- edit: `artifact_get -> author -> validate -> artifact_validate -> artifact_apply`
- removal: `artifact_get -> artifact_delete`

Three rules govern them:

1. **Changing a resource that already exists requires `expectedContentHash` on
   that resource's draft**, set to the `contentHash` returned by the
   `artifact_get` just made on it. An apply without it overwrites whatever is
   stored, whoever wrote it and whenever. A draft for a resource that does not
   exist yet carries no precondition. Carrying one on an unchanged draft of the
   same closure is also correct: if that resource has moved, refusing the batch
   is the check working rather than a malfunction.
2. **`artifact_delete` requires `expectedContentHash`.** It is a required
   argument, not an optional one. The removal is permanent and leaves no
   tombstone, and the only way back is applying the resource again.
3. **Never answer `artifact.version-conflict` by dropping the precondition and
   retrying.** That code means the stored version moved after this session read
   it. An apply without the check would then overwrite precisely the change
   somebody else had just made, leaving nothing behind to say it ever existed.
   Call `artifact_get` again, redo the intended change on the version it
   returns, and send that version's hash. If the change cannot be re-expressed
   on the new version, stop and report the conflict.

### Report A Refused Change Or Removal

Stop on each code below without masking it, exactly as for any other
authoritative coded error. Each one names a next step that belongs to the user.

- **`artifact.in-use`**: another resource still references this id. Read the
  `referrers` parameter and report which resources they are. Do not remove a
  referrer to clear the way; which of the two to keep is the user's decision.
- **`artifact.pipeline-not-stopped`**: the id is a pipeline that is running or
  is about to. Read `actual` and `desired` and report both. Do not issue a stop
  as part of the removal; stopping a pipeline carries its own consequences and
  is the user's decision.
- **`artifact.not-found`**: the id is not stored. Report that the target no
  longer exists and stop. Never rewrite a precondition-bearing apply into one
  without a precondition and send it again. The first request means change the
  resource that is there, the rewritten one means create this resource, and an
  author who asked for an edit is never told it became a creation.

The last of those and rule 3 are one mistake wearing two faces: answering a
failed precondition by removing the precondition. The codes differ only in what
became of the resource meanwhile, `artifact.version-conflict` meaning somebody
changed it and `artifact.not-found` meaning somebody removed it. Neither is
repaired by retrying without the check.
