# Tapstate Skills

Installable Agent Skills for authoring and operating Tapstate resources.

## Install

Install every Skill globally for agents that support global Skill installation:

```sh
npx --yes skills add tapstate/tapstate-skills -a claude-code codex cursor antigravity-cli -g -y
```

Install only the DSL authoring Skill globally:

```sh
npx --yes skills add tapstate/tapstate-skills --skill dsl-authoring -a claude-code codex cursor antigravity-cli -g -y
```

List the Skills that the installer can discover:

```sh
npx --yes skills add tapstate/tapstate-skills --list
```

The named agents use the shared `~/.agents/skills` global store. The explicit
agent list avoids installers that cannot install Skills globally, including
PromptScript. The `skills` command fetches this Git repository. This repository
is not an npm package and intentionally has no `package.json`. Git commits and
tags identify published revisions.

## Native plugin

Use the native marketplace when host-specific plugin updates are preferred:

```sh
# Claude Code: install, then `claude plugin update tapstate-skills`
/plugin marketplace add tapstate/tapstate-skills
/plugin install tapstate-skills@tapstate-skills

# Codex: install
codex plugin marketplace add tapstate/tapstate-skills
codex plugin add tapstate-skills@tapstate-skills

# Codex: refresh and reinstall after a marketplace release
codex plugin marketplace upgrade tapstate-skills
codex plugin add tapstate-skills@tapstate-skills
```

The Codex marketplace resolves the repository-native
`.agents/plugins/marketplace.json` entry and the root
`.codex-plugin/plugin.json` manifest. The plugin uses the repository's existing
`skills/` directory; it does not maintain a second Skill copy.

## DSL authoring boundary

`dsl-authoring` understands the complete `tapstate/v1` resource model. It can
draft, modify, explain, review, repair, and validate Sources, Pipelines,
Transforms, Views, and Serve resources, including references and data-flow
semantics.

Connector-specific members inside `source.config` are the only dynamic DSL
boundary. For a known connector, the Skill uses live MCP `source_draft` to
validate structured config and return canonical Source YAML without persisting an
artifact. The returned document is written to
`<workspace-root>/source/<id>.tap.yml`. This is a Tapstate workspace convention,
not an installed-Skill path or a repository-relative path. The first resource
creates its workspace and kind directory as needed, just as `tapstate new` does.
Only a later complete-workspace apply creates online artifacts. The Skill does
not infer configuration keys, defaults, or secret values from model memory or
examples. When editing an existing Source, it preserves `source.config` by
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
