---
name: tapstate-getting-started
description: >-
  Give a guided tour of a running Tapstate server over MCP: check the
  connection, explain what it is running, show real data, teach the core ideas,
  and offer next steps. Use when the user asks how to work with Tapstate, what
  Tapstate can do, where to start, or what is running, or has just set up the
  Tapstate demo and wants to try it.
---

# Tapstate Getting Started

Lead a first-time user from "what is this?" to doing something real, using the
Tapstate server this session is connected to. Show rather than describe: every
claim about the server comes from a tool result in this session.

Stay read-only until the user picks a next step. Never stop, pause, delete or
change a source or pipeline the user did not create in this conversation unless
they explicitly ask for that exact action. Stopping a pipeline can discard what
it has accumulated.

## 1. Check The Connection

Discover the Tapstate tools from the session's MCP tool inventory; do not assume
a server name or tool prefix. Then call `system_version`.

- **No Tapstate tools are exposed:** the agent is not connected. Say so, and
  give the fix that matches the user's setup:
  - The demo (`install.tapstate.dev/demo`) writes `tapstate-mcp.json` in its
    folder. In Claude Code, start the session from that folder with
    `claude --mcp-config tapstate-mcp.json`. For other agents, `AGENT-SETUP.md`
    in the same folder has the steps.
  - Anywhere else, the server's MCP sidecar has to be configured in the agent;
    point to https://tapstate.dev/docs.
  Stop here. Do not describe a server you cannot see.
- **The call fails to connect:** the server is not reachable. If this is the
  demo, the stack may be stopped: `docker compose start` in the demo folder
  resumes it.
- **The call is refused as unauthenticated:** the token was revoked, or the demo
  was deleted with `docker compose down -v`. Re-running the demo writes a new
  `tapstate-mcp.json`.

Report the error the tool returned; do not guess a different cause.

## 2. Survey What Is Running

Call `source_list`, `pipeline_list`, `pipeline_status` for each pipeline, and
`data_browser_collections`. From the results, tell the user in a few plain
sentences:

- which databases are connected (each source, and its connector);
- which pipelines exist, what each reads, and whether it is running;
- where the results land (the view collections).

If the server holds the demo's resources - sources `orders_db` (MySQL) and
`fulfillment_db` (PostgreSQL), pipeline `order_pipeline`, collection
`views.order_state` - explain the demo's point: orders live in MySQL and their
shipments in PostgreSQL, two engines that cannot see each other's tables, and
the pipeline keeps one MongoDB document per order with its shipments inside it,
current as either database changes. Confirm it against the tool results rather
than asserting it from this paragraph.

If the server is empty, say so, and go to step 5 with "connect your own
database" and "build a first pipeline" as the leading options.

## 3. Show One Real Document

Pick one view collection and read a single document with `data_browser_find`
(for the demo, the order with `id` 1 in `views.order_state`). Show it, and point
out which fields came from which source. One document is enough; do not dump a
collection.

## 4. Teach The Core Ideas

Read `references/concepts.md`, then explain only the ideas the user has just
seen, in their terms, in about five short lines: source, pipeline, transform,
view, and the snapshot that is followed by change data capture. Mention that the
same resources are visible in the Tapstate web app on the same server.

## 5. Offer Next Steps

Offer three to five numbered options that fit what the server holds, each with
one line on what it does, and wait for the user to choose. Suitable options:

1. **Build a pipeline.** For the demo: "keep a live copy of the shipments with
   only their id, carrier and status". Hand the authoring to the
   `dsl-authoring` skill; if it is not installed, say that it is needed and how
   to install it (`npx --yes skills add tapstate/tapstate-skills`) rather than
   writing pipeline YAML without it.
2. **Watch a change arrive.** For the demo: ask the user to run `./try-cdc.sh`
   in the demo folder (or one of its SQL lines), then read the same document
   again and show what changed. Do not claim a change arrived before reading it.
3. **Check a pipeline's health.** Use `pipeline_status`, `pipeline_metrics` and
   `pipeline_explain`, and explain the result, including any lag.
4. **Connect your own database.** List connectors with `connector_list`, then
   draft a source with `source_draft`. Connection details and credentials come
   from the user; never invent configuration keys, defaults or secrets.
5. **Clean up.** Remove only what was created in this conversation, after
   confirming each resource by name.

When the user picks one, do it, then report what happened. Keep validated,
applied and running distinct: a pipeline that validates is not yet running, and
a running pipeline has not delivered data until a read shows it. After the step,
offer the remaining options again in one line.
