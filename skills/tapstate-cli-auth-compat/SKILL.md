---
name: tapstate-cli-auth-compat
description: Verify that a legacy Tapstate CLI works against a candidate Tapstate server using the repository's redacted local compatibility verifier. Use when validating an authentication migration, HTTP security change, CLI/server release compatibility, or machine-token revocation behavior.
---

# Tapstate CLI Authentication Compatibility

Use the verifier from the Tapstate repository. Do not copy its executable logic into this Skill.

## Prepare Isolated Inputs

1. Resolve a Tapstate repository checkout containing `tools/verify-cli-auth-compat.sh`. If it is absent, ask for the checkout path instead of creating a substitute script.
2. Use separate disposable worktrees for the legacy CLI and candidate server. The verifier runs `mvn clean`, so never give it a shared worktree.
3. Use a Mongo URI template containing the literal `{database}` placeholder. Do not point the verifier at an existing project database.

## Run The Verifier

From the Tapstate checkout, run:

```sh
tools/verify-cli-auth-compat.sh \
  --client-root <legacy-cli-worktree> \
  --server-root <candidate-server-worktree> \
  --mongo-uri '<mongo-uri-template-containing-{database}>' \
  --cleanup
```

The script starts the candidate server only on loopback, creates a uniquely named `tapstate_cli_auth_verify_*` database, and removes only that generated database when `--cleanup` is present.

## Interpret Evidence

Read the generated `report.json`. A pass records the two commits and the observed bootstrap, legacy login/read, authenticated write, and machine-token revoke HTTP results. It excludes passwords, JWTs, Mongo URIs, and bearer values.

If any case fails, retain the output directory and report the failed invariant. Do not mark a live-verification acceptance criterion complete.
