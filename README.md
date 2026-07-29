# Tapstate Skills

Installable Agent Skills for authoring and operating Tapstate resources.

## Install

Install every Skill in this repository:

```sh
npx skills add tapstate/tapstate-skills
```

Install only the DSL authoring Skill:

```sh
npx skills add tapstate/tapstate-skills --skill dsl-authoring
```

List the Skills that the installer can discover:

```sh
npx skills add tapstate/tapstate-skills --list
```

The `skills` command fetches this Git repository. This repository is not an npm
package and intentionally has no `package.json`. Git commits and tags identify
published revisions.

## DSL authoring boundary

`dsl-authoring` understands the complete `tapstate/v1` resource model. It can
draft, modify, explain, review, repair, and validate Sources, Pipelines,
Transforms, Views, and Serve resources, including references and data-flow
semantics.

Connector-specific members inside `source.config` are the only dynamic DSL
boundary. They must come from the live connector catalog through MCP. The Skill
does not infer configuration keys, defaults, or secret values from model memory
or examples. When editing an existing Source, it preserves `source.config` by
default. Replacing it requires explicit user intent and a current contract from
MCP.

Without MCP, the Skill can produce a config-free draft and validate its offline
DSL semantics. That draft is not runnable. Offline validation also does not
prove current runtime support, connector availability, credential validity, or
online Pipeline state.

## Compatibility

[`upstream.lock`](upstream.lock) records the Tapstate commit, DSL version,
generated schema checksum, CLI version, and verified release-archive checksum
used to test the Skill. The vendored schema is copied byte-for-byte from that
Tapstate commit and remains the machine-readable grammar authority.

## Contributing

Keep all tracked files, comments, commit messages, and pull request text in
English with zero CJK characters. Never add connector configuration or secrets
to examples. Run the validation workflow locally where practical before opening
a pull request.
