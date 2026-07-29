<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.74.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.74.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of pinned 40-character SHA digests, making them vulnerable to supply-chain attacks if the referenced tag or branch is moved.

lint.yml: actions/checkout@v4 (×3), reviewdog/action-shellcheck@v1, reviewdog/action-hadolint@v1, reviewdog/action-actionlint@v1
main.yml: actions/checkout@v4, anothrNick/github-tag-action@master, softprops/action-gh-release@v2.0.0
test.yml: actions/checkout@v4

Locations:

- `.github/workflows/lint.yml:20`
- `.github/workflows/lint.yml:21`
- `.github/workflows/lint.yml:31`
- `.github/workflows/lint.yml:33`
- `.github/workflows/lint.yml:41`
- `.github/workflows/lint.yml:43`
- `.github/workflows/main.yml:17`
- `.github/workflows/main.yml:22`
- `.github/workflows/main.yml:28`
- `.github/workflows/test.yml:20`

### script-injection (severity: high)

Sub-rule (a): The 'Check if the tag would have been created' run: block in test.yml directly interpolates ${{ steps.*.outputs.* }} expressions into shell variable assignments without any quoting. For example: `MAIN1_OUTPUT_TAG=${{ steps.test_main1.outputs.old_tag }}`, `MAIN1_OUTPUT_NEWTAG=${{ steps.test_main1.outputs.new_tag }}`, `PRE1_OUTPUT_TAG=${{ steps.test_pre1.outputs.old_tag }}`, and many more. These step outputs can contain attacker-controlled values (e.g. from a PR that manipulates the tag output), and the expressions are substituted directly into the shell script before execution, enabling command injection.

Locations:

- `.github/workflows/test.yml:75`

### github-env-injection (severity: high)

In entrypoint.sh, the setOutput() function writes values to $GITHUB_OUTPUT via `echo "${1}=${2}" >> "${GITHUB_OUTPUT}"` without any newline sanitization. Several calls pass values derived directly from inherited environment variables set by the calling workflow:
- `setOutput "new_tag" "$new"` where `$new` can be set to `$custom_tag` (= `${CUSTOM_TAG:-}`), a raw env var from the calling workflow.
- `setOutput "part" "$default_semvar_bump"` where `$default_semvar_bump` = `${DEFAULT_BUMP:-minor}`, another inherited env var.
Neither value is sanitized with `printf '%s' ... | tr -d '\n\r'` before the write, allowing a newline in CUSTOM_TAG or DEFAULT_BUMP to inject additional key=value pairs into GITHUB_OUTPUT.

Locations:

- `entrypoint.sh:6`
- `entrypoint.sh:10`
- `entrypoint.sh:56`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

1. unpinned-uses: Pinned all action references to full SHA digests in lint.yml (actions/checkout@11d596..., reviewdog/action-shellcheck@4c0745..., reviewdog/action-hadolint@1b2cfa..., reviewdog/action-actionlint@508422...), main.yml (actions/checkout@11d596..., anothrNick/github-tag-action@4ed449..., softprops/action-gh-release@a6c748...), and test.yml (actions/checkout@11d596...). Original tags preserved as comments.

2. script-injection: In test.yml's 'Check if the tag would have been created' step, moved all 21 ${{ steps.*.outputs.* }} expressions from the run: block into the step's env: block. The shell script now references plain environment variables ($MAIN1_OUTPUT_TAG, etc.) instead of directly interpolating GitHub expressions.

3. github-env-injection: Fixed the setOutput() function in entrypoint.sh to sanitize values before writing to $GITHUB_OUTPUT. Added `local safe_value; safe_value=$(printf '%s' "${2}" | tr -d '\n\r')` to strip newlines/carriage returns, preventing injection of additional key=value pairs into GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansions in the 'Check if the tag would have been created' step in .github/workflows/test.yml. All five `verlt` function calls now properly double-quote their arguments (MAIN1, PRE1, MAIN2, PRE2, MAIN5 tag variables). Also fixed the internal `verlte $1 $2` call inside `verlt` to use `verlte "$1" "$2"` to prevent word splitting on positional parameters.

