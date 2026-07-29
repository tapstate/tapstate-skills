# AGENTS.md

## Scope

This public repository contains installable Tapstate Agent Skills. Keep product
Skills under `skills/<skill-name>/`, with one required `SKILL.md` in each Skill
directory. Repository-level guidance belongs in the root `README.md`; do not add
a Skill-local README.

## Language and repository hygiene

- Write every tracked file, comment, commit message, and pull request in English.
- Do not add CJK characters, including CJK punctuation or full-width forms.
- Do not commit credentials, tokens, connection strings, private keys, or other
  secrets.
- Do not add a root `package.json` unless repository-owned Node tooling actually
  requires one. npm package metadata is not a Skill version contract.
- Keep generated files and local installer output out of version control.

## Authoring contracts

- Treat the vendored `tapstate/v1` JSON Schema as generated, authoritative
  grammar. Copy it byte-for-byte from the Tapstate commit recorded in
  `upstream.lock`; never edit it by hand.
- Update the Tapstate commit, schema checksum, CLI version, release-archive
  checksum, semantic references, runtime-support notes, and validated examples
  together when the upstream contract changes.
- Treat connector-specific members inside `source.config` as the only dynamic
  DSL member boundary. Obtain them from the live connector catalog through MCP.
- Never invent configuration keys, defaults, secret values, or connector
  constraints. Examples must omit `source.config` entirely.
- Preserve an existing `source.config` by default. Replace it only when the user
  explicitly asks and MCP supplies the current connector contract.
- Distinguish grammar validity, offline validation, runtime support, and online
  execution state. Do not describe one as proof of another.
- Keep `SKILL.md` procedural and concise. Put detailed semantics in
  `references/` and reusable output fixtures in `assets/`.

## Validation

Use the versions pinned in `upstream.lock` and the validation workflow. At a
minimum, verify:

```sh
npx --yes skills@1.5.20 add . --list
tapstate validate skills/dsl-authoring/assets/examples/<example>
```

Also verify the schema checksum, scan all tracked files for CJK characters, and
confirm that examples contain neither `config` members nor secret material.
