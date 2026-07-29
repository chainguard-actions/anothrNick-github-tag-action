<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.72.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.72.0** was hardened automatically. 2 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions by mutable tag or branch names instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the referenced tag or branch is moved.

.github/workflows/lint.yml:
  - uses: actions/checkout@v4 (lines 21, 32, 40)
  - uses: reviewdog/action-shellcheck@v1 (line 22)
  - uses: reviewdog/action-hadolint@v1 (line 34)
  - uses: reviewdog/action-actionlint@v1 (line 42)

.github/workflows/main.yml:
  - uses: actions/checkout@v4 (line 18)
  - uses: anothrNick/github-tag-action@master (line 23) — branch ref
  - uses: softprops/action-gh-release@v2.0.0 (line 29)

.github/workflows/test.yml:
  - uses: actions/checkout@v4 (line 22)

Locations:

- `.github/workflows/lint.yml:21`
- `.github/workflows/lint.yml:22`
- `.github/workflows/lint.yml:32`
- `.github/workflows/lint.yml:34`
- `.github/workflows/lint.yml:40`
- `.github/workflows/lint.yml:42`
- `.github/workflows/main.yml:18`
- `.github/workflows/main.yml:23`
- `.github/workflows/main.yml:29`
- `.github/workflows/test.yml:22`

### script-injection (severity: high)

Sub-rule (a): The 'Check if the tag would have been created' run: block in test.yml directly interpolates ${{ steps.*.outputs.* }} expressions inside shell commands. These values flow through YAML template substitution before the shell parses them, allowing an attacker-controlled tag value (e.g. containing shell metacharacters) to execute arbitrary commands. Offending lines include:
  MAIN1_OUTPUT_TAG=${{ steps.test_main1.outputs.old_tag }}
  MAIN1_OUTPUT_NEWTAG=${{ steps.test_main1.outputs.new_tag }}
  MAIN1_OUTPUT_PART=${{ steps.test_main1.outputs.part }}
  (and similar for PRE1, MAIN2, PRE2, MAIN3, MAIN4, MAIN5 outputs)
All step outputs should be passed via env: variables and then referenced as quoted shell variables (e.g. "$MAIN1_OUTPUT_TAG") instead of being interpolated directly.

Locations:

- `.github/workflows/test.yml:111`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all unpinned action references by pinning to full 40-character commit SHAs: actions/checkout@v4 → 11d5960a326750d5838078e36cf38b85af677262, reviewdog/action-shellcheck@v1 → 4c07458293ac342d477251099501a718ae5ef86e, reviewdog/action-hadolint@v1 → 1b2cfa6ba72072ad35158d7ff3aa49bbdc03506d, reviewdog/action-actionlint@v1 → 50842263c20a7c46bd0065b9e624d3c569db061e, anothrNick/github-tag-action@master → 4ed44965e0db8dab2b466a16da04aec3cc312fd8, softprops/action-gh-release@v2.0.0 → a6c7483a42ee9d5daced968f6c217562cd680f7f. Fixed script injection in test.yml by moving all 15 ${{ steps.*.outputs.* }} expressions from inline shell assignments in the run: block into an env: block, so the shell script references them as plain environment variables.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection finding in entrypoint.sh: updated the setOutput() function to sanitize values before writing to $GITHUB_OUTPUT by using `printf '%s' "${2}" | tr -d '\n\r'` to strip embedded newlines and carriage returns. This single fix covers all 16 call sites that pass caller-controlled values (custom_tag, default_semvar_bump, part, new, tag, pre_tag) to $GITHUB_OUTPUT. The script-injection finding in .github/workflows/test.yml was intentionally not fixed per the instructions that state security fixes should only be applied to action.yml and supporting scripts, not to test harness files.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable references in .github/workflows/test.yml at the 'Check if the tag would have been created' step. All calls to verlt() and verlte() now properly double-quote their arguments: `verlt "$MAIN1_OUTPUT_TAG" "$MAIN1_OUTPUT_NEWTAG"` etc. Also fixed the internal `verlte $1 $2` call within the `verlt` function to `verlte "$1" "$2"`. This prevents shell metacharacter injection if action outputs contain special characters.

