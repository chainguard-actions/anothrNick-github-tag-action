<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.55.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.55.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All three workflow files reference actions using mutable tags or branch names instead of pinned full-length SHA commit hashes, making the workflow vulnerable to supply-chain attacks if those tags are moved or compromised.

- lint.yml: `actions/checkout@v3` (line 21), `reviewdog/action-shellcheck@v1` (line 22), `actions/checkout@v3` (line 33), `reviewdog/action-hadolint@v1` (line 35), `actions/checkout@v3` (line 43), `reviewdog/action-actionlint@v1` (line 45)
- main.yml: `actions/checkout@v3` (line 14), `anothrNick/github-tag-action@master` (line 19), `marvinpinto/action-automatic-releases@v1.2.1` (line 25)
- test.yml: `actions/checkout@v3` (line 18)

Locations:

- `.github/workflows/lint.yml:21`
- `.github/workflows/lint.yml:22`
- `.github/workflows/lint.yml:33`
- `.github/workflows/lint.yml:35`
- `.github/workflows/lint.yml:43`
- `.github/workflows/lint.yml:45`
- `.github/workflows/main.yml:14`
- `.github/workflows/main.yml:19`
- `.github/workflows/main.yml:25`
- `.github/workflows/test.yml:18`

### missing-permissions (severity: medium)

The workflow file main.yml has no top-level `permissions:` key and none of its jobs define a `permissions:` block. Without explicit permissions, the workflow inherits the default repository permissions (which may include broad write access), violating the principle of least privilege.

Locations:

- `.github/workflows/main.yml:1`

### script-injection (severity: high)

Sub-rule (a): The `run:` block in test.yml directly interpolates `steps.*.outputs.*` expressions inside shell commands. These values are workflow-controllable and are substituted by the YAML template engine before the shell sees them, allowing an attacker to inject arbitrary shell commands.

Offending lines:
  MAIN_OUTPUT_TAG=${{ steps.test_main.outputs.old_tag }}
  MAIN_OUTPUT_NEWTAG=${{ steps.test_main.outputs.new_tag }}
  MAIN_OUTPUT_PART=${{ steps.test_main.outputs.part }}
  PRE_OUTPUT_TAG=${{ steps.test_pre.outputs.old_tag }}
  PRE_OUTPUT_NEWTAG=${{ steps.test_pre.outputs.new_tag }}
  PRE_OUTPUT_PART=${{ steps.test_pre.outputs.part }}

Fix: move these values into `env:` variables and reference them as `$ENV_VAR` (double-quoted) inside the script.

Locations:

- `.github/workflows/test.yml:44`

### github-env-injection (severity: high)

entrypoint.sh defines a `setOutput()` helper that writes values directly to `$GITHUB_OUTPUT` without sanitization:

  setOutput() { echo "${1}=${2}" >> "${GITHUB_OUTPUT}" }

This function is called with values derived from inherited process environment variables set by the calling workflow — notably `$custom_tag` (from `CUSTOM_TAG`), `$new` (which is overwritten with `$custom_tag` when set), `$tag`, and `$part`. None of these values are sanitized with `printf '%s' "$VAR" | tr -d '\n\r'` before being written to `$GITHUB_OUTPUT`. A newline embedded in any of these values (e.g. via `CUSTOM_TAG`) would allow injection of arbitrary key=value pairs into the GitHub output environment.

Locations:

- `entrypoint.sh:43`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings:
1. unpinned-uses: Pinned all 10 action references across lint.yml, main.yml, and test.yml to full SHA commit hashes with original tags preserved as comments.
2. missing-permissions: Added `permissions: contents: write` to main.yml (needed for pushing tags and creating releases).
3. script-injection: Moved all ${{ steps.*.outputs.* }} expressions in test.yml's run block into an env: block, referencing them as plain shell variables.
4. github-env-injection: Updated setOutput() in entrypoint.sh to sanitize values with `printf '%s' | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

