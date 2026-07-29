<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.71.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.71.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tags or branch names instead of full 40-character commit SHAs. This exposes the workflow to supply-chain attacks where a tag is moved to point to malicious code.

.github/workflows/lint.yml:
  - actions/checkout@v4
  - reviewdog/action-shellcheck@v1
  - reviewdog/action-hadolint@v1
  - reviewdog/action-actionlint@v1

.github/workflows/main.yml:
  - actions/checkout@v4
  - anothrNick/github-tag-action@master  (branch reference — highest risk)
  - softprops/action-gh-release@v2.0.0

.github/workflows/test.yml:
  - actions/checkout@v4

All of these should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/lint.yml:19`
- `.github/workflows/lint.yml:20`
- `.github/workflows/lint.yml:29`
- `.github/workflows/lint.yml:33`
- `.github/workflows/lint.yml:40`
- `.github/workflows/lint.yml:42`
- `.github/workflows/main.yml:18`
- `.github/workflows/main.yml:24`
- `.github/workflows/main.yml:30`
- `.github/workflows/test.yml:22`

### script-injection (severity: high)

Sub-rule (a): The 'Check if the tag would have been created' run: block in test.yml directly interpolates ${{ steps.*.outputs.* }} expressions inside shell variable assignments. For example:

  MAIN1_OUTPUT_TAG=${{ steps.test_main1.outputs.old_tag }}
  MAIN1_OUTPUT_NEWTAG=${{ steps.test_main1.outputs.new_tag }}
  MAIN1_OUTPUT_PART=${{ steps.test_main1.outputs.part }}
  PRE1_OUTPUT_TAG=${{ steps.test_pre1.outputs.old_tag }}
  ... (and many more)

These step outputs are derived from git tag names and commit messages, which can be attacker-controlled (e.g., via a crafted branch name or commit message in a pull request). Injecting these directly into the shell script without quoting or sanitization allows shell metacharacter injection. The values should be passed via env: variables and then double-quoted in the script.

Locations:

- `.github/workflows/test.yml:99`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all unpinned action references by pinning to full commit SHAs: actions/checkout@v4 → 11d5960a326750d5838078e36cf38b85af677262 (used in lint.yml ×3, main.yml, test.yml), reviewdog/action-shellcheck@v1 → 4c07458293ac342d477251099501a718ae5ef86e, reviewdog/action-hadolint@v1 → 1b2cfa6ba72072ad35158d7ff3aa49bbdc03506d, reviewdog/action-actionlint@v1 → 50842263c20a7c46bd0065b9e624d3c569db061e, anothrNick/github-tag-action@master → 4ed44965e0db8dab2b466a16da04aec3cc312fd8, softprops/action-gh-release@v2.0.0 → a6c7483a42ee9d5daced968f6c217562cd680f7f. Fixed script injection in test.yml by moving all 15 ${{ steps.*.outputs.* }} expressions from inline shell variable assignments in the run: block into a step-level env: block, so the values are passed as environment variables rather than being interpolated directly into the shell script.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

1. entrypoint.sh: Updated the `setOutput` function to sanitize both key and value using `printf '%s' | tr -d '\n\r'` before writing to $GITHUB_OUTPUT, preventing newline injection from attacker-controlled values like CUSTOM_TAG. 2. .github/workflows/test.yml: Added double-quotes around all unquoted variable expansions passed to `verlt` and `verlte` functions (MAIN1/2/5_OUTPUT_TAG/NEWTAG, PRE1/2_OUTPUT_TAG/NEWTAG), fixed the internal `verlte $1 $2` call to `verlte "$1" "$2"`, and replaced backtick command substitution with $() in verlte.

