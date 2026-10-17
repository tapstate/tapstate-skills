# Tapstate Concepts For A First Explanation

Use these to explain what the user has just seen. Explain only the ideas that
appear in the current server, in plain words, and tie each to a name from the
tool results.

- **Source:** a connection to one database, through a connector such as `mysql`,
  `postgres` or `mongodb`, naming the tables Tapstate reads. A source does not
  move data by itself.
- **Pipeline:** reads one or more sources and keeps a result up to date. It runs
  on the Tapstate server; starting it is a separate step from defining it.
- **Transform:** a step inside a pipeline that reshapes rows - keeping some
  fields, filtering rows, or nesting rows from one table inside the documents of
  another. Nesting is how the demo puts each order's shipments inside the order.
- **View:** the result a pipeline maintains, materialized as documents in the
  server's own store, where it can be read like any collection. Declaring a view
  is what tells the pipeline to keep it.
- **Snapshot, then change data capture:** a pipeline first copies what the
  sources already hold, then follows every insert, update and delete as it
  happens, so the view stays current without being re-run.

What Tapstate is not, when the user compares it to something familiar:

- It is not a database view or a SQL join: the sources can be different
  engines, and the result is stored and kept current, not computed per query.
- It is not a one-off copy: after the snapshot it keeps following changes.

Resources are written as `tapstate/v1` YAML files (`*.tap.yml`). The
`dsl-authoring` skill covers writing and changing them.
