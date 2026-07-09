<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.55.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **anothrNick--github-tag-action/1.55.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in a run: block. In .github/workflows/test.yml, the step 'Check if the tag would have been created' directly interpolates ${{ steps.test_main.outputs.old_tag }}, ${{ steps.test_main.outputs.new_tag }}, ${{ steps.test_main.outputs.part }}, ${{ steps.test_pre.outputs.old_tag }}, ${{ steps.test_pre.outputs.new_tag }}, and ${{ steps.test_pre.outputs.part }} inside shell variable assignments in a run: block. These steps.*.outputs.* values flow through YAML template substitution before the shell processes them, allowing shell metacharacter injection.

Locations:

- `.github/workflows/test.yml:47`

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of full 40-character SHA commit hashes. Failing references: lint.yml: actions/checkout@v3, reviewdog/action-shellcheck@v1, reviewdog/action-hadolint@v1, reviewdog/action-actionlint@v1. main.yml: actions/checkout@v3, anothrNick/github-tag-action@master, marvinpinto/action-automatic-releases@v1.2.1. test.yml: actions/checkout@v3.

Locations:

- `.github/workflows/lint.yml:20`
- `.github/workflows/lint.yml:21`
- `.github/workflows/lint.yml:32`
- `.github/workflows/lint.yml:33`
- `.github/workflows/lint.yml:43`
- `.github/workflows/lint.yml:45`
- `.github/workflows/main.yml:16`
- `.github/workflows/main.yml:22`
- `.github/workflows/main.yml:28`
- `.github/workflows/test.yml:20`

### missing-permissions (severity: medium)

The workflow file main.yml has no top-level permissions: key and its only job (bump-version) also has no job-level permissions: key. This means the workflow runs with the default (broad) GITHUB_TOKEN permissions.

Locations:

- `.github/workflows/main.yml:1`

### github-env-injection (severity: high)

In entrypoint.sh, the setOutput() function writes values to $GITHUB_OUTPUT using: echo "${1}=${2}" >> "${GITHUB_OUTPUT}". This function is called with values like $new, $tag, and $part that are derived from git commit messages (via $log) and repository data, which can be controlled by PR authors via commit message content. No sanitization (printf '%s' ... | tr -d '\n\r') is applied before writing to $GITHUB_OUTPUT, allowing newline injection that could add arbitrary entries to the output file.

Locations:

- `entrypoint.sh:44`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, missing-permissions, github-env-injection

**Notes:**

Fixed all four findings: (1) script-injection in test.yml: moved all ${{ steps.*.outputs.* }} expressions from inline shell variable assignments into the step's env: block; (2) unpinned-uses: pinned all actions to full 40-char SHAs with tag comments - actions/checkout@v3 -> f43a0e5ff2bd294095638e18286ca9a3d1956744, reviewdog/action-shellcheck@v1 -> 4c07458293ac342d477251099501a718ae5ef86e, reviewdog/action-hadolint@v1 -> 1b2cfa6ba72072ad35158d7ff3aa49bbdc03506d, reviewdog/action-actionlint@v1 -> 6fb7acc99f4a1008869fa8a0f09cfca740837d9d, anothrNick/github-tag-action@master -> 4ed44965e0db8dab2b466a16da04aec3cc312fd8, marvinpinto/action-automatic-releases@v1.2.1 -> 919008cf3f741b179569b7a6fb4d8860689ab7f0; (3) missing-permissions in main.yml: added top-level 'permissions: contents: write' (needed for pushing tags and creating releases); (4) github-env-injection in entrypoint.sh: updated setOutput() to sanitize values with printf '%s' | tr -d '\n\r' before writing to $GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable usage in .github/workflows/test.yml at the 'Check if the tag would have been created' step. Specifically: (1) Quoted $MAIN_OUTPUT_TAG, $MAIN_OUTPUT_NEWTAG, $PRE_OUTPUT_TAG, and $PRE_OUTPUT_NEWTAG in the verlt() function calls (lines ~72-73); (2) Also quoted $1 and $2 in the verlte call within the verlt() function body to prevent injection when values are passed through the function chain. The env: block already correctly isolates the ${{ steps.*.outputs.* }} expressions from the shell script.

