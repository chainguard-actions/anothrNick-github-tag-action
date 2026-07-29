<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.73.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.73.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files use mutable tag or branch refs instead of full 40-character SHA commit digests, making them vulnerable to supply-chain attacks if the referenced action is compromised or the tag is moved.

Failing references:
- lint.yml: `actions/checkout@v4`, `reviewdog/action-shellcheck@v1`, `reviewdog/action-hadolint@v1`, `reviewdog/action-actionlint@v1`
- main.yml: `actions/checkout@v4`, `anothrNick/github-tag-action@master`, `softprops/action-gh-release@v2.0.0`
- test.yml: `actions/checkout@v4`

All should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/lint.yml:19`
- `.github/workflows/lint.yml:20`
- `.github/workflows/lint.yml:30`
- `.github/workflows/lint.yml:36`
- `.github/workflows/lint.yml:42`
- `.github/workflows/main.yml:19`
- `.github/workflows/main.yml:24`
- `.github/workflows/main.yml:31`
- `.github/workflows/test.yml:19`

### script-injection (severity: high)

Rule (a) violation: The 'Check if the tag would have been created' run: block in test.yml directly interpolates `${{ steps.*.outputs.* }}` expressions into shell variable assignments. Any `${{ ... }}` expression inside a run: script is evaluated by the YAML template engine before the shell sees it, allowing an attacker who can influence step outputs to inject arbitrary shell commands.

Offending lines (examples):
  MAIN1_OUTPUT_TAG=${{ steps.test_main1.outputs.old_tag }}
  MAIN1_OUTPUT_NEWTAG=${{ steps.test_main1.outputs.new_tag }}
  PRE1_OUTPUT_TAG=${{ steps.test_pre1.outputs.old_tag }}
  (and 18 more similar assignments)

Fix: move the values into an env: block and reference them as quoted shell variables, e.g.:
  env:
    MAIN1_OUTPUT_TAG: ${{ steps.test_main1.outputs.old_tag }}
  run: |
    echo "$MAIN1_OUTPUT_TAG"

Locations:

- `.github/workflows/test.yml:119`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all unpinned action references in lint.yml (5 refs), main.yml (3 refs), and test.yml (1 ref) by resolving each to its full 40-character SHA via lookup_action_sha. Fixed script injection in test.yml's 'Check if the tag would have been created' step by moving all 21 ${{ steps.*.outputs.* }} template expressions out of the run: block and into an env: block, then referencing them as plain shell environment variables.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the setOutput() function in entrypoint.sh to sanitize values before writing to $GITHUB_OUTPUT. Changed the function body from a direct `echo "${1}=${2}"` to first strip newlines/carriage returns via `safe_value=$(printf '%s' "${2}" | tr -d '\n\r')` and then write `${1}=${safe_value}`. This single change protects all call sites (lines 196-199 and others) since they all route through the same function. Attacker-controlled env vars like $CUSTOM_TAG, $DEFAULT_BUMP, $INITIAL_VERSION, $TAG_PREFIX, and $PRERELEASE_SUFFIX can no longer inject newline characters to poison $GITHUB_OUTPUT with arbitrary key-value pairs.

