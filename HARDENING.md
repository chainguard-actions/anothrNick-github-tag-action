<!-- markdownlint-disable -->

# Hardening Report: anothrNick--github-tag-action/1.75.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **anothrNick--github-tag-action/1.75.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tags or branch names instead of immutable full 40-character SHA digests, making them vulnerable to supply-chain attacks if the referenced action is compromised or its tag is moved.

lint.yml:
  - uses: actions/checkout@v4
  - uses: reviewdog/action-shellcheck@v1
  - uses: reviewdog/action-hadolint@v1
  - uses: reviewdog/action-actionlint@v1

main.yml:
  - uses: actions/checkout@v4
  - uses: anothrNick/github-tag-action@master  (branch ref!)
  - uses: softprops/action-gh-release@v2.0.0

test.yml:
  - uses: actions/checkout@v4
  - uses: bats-core/bats-action@3.0.1

Locations:

- `.github/workflows/lint.yml:19`
- `.github/workflows/lint.yml:20`
- `.github/workflows/lint.yml:30`
- `.github/workflows/lint.yml:36`
- `.github/workflows/lint.yml:38`
- `.github/workflows/main.yml:18`
- `.github/workflows/main.yml:23`
- `.github/workflows/main.yml:29`
- `.github/workflows/test.yml:22`
- `.github/workflows/test.yml:80`
- `.github/workflows/test.yml:86`

### script-injection (severity: high)

Sub-rule (a): The 'Check if the tag would have been created' run: block in test.yml directly interpolates ${{ steps.*.outputs.* }} expressions into shell variable assignments (e.g., `MAIN1_OUTPUT_TAG=${{ steps.test_main1.outputs.old_tag }}`). The `steps.*.outputs.*` context is workflow-controllable — a malicious tag value containing shell metacharacters (`;`, `|`, `$(...)`, etc.) would be injected directly into the shell script before the shell ever parses it, enabling command injection. All such assignments in this run: block are affected.

Locations:

- `.github/workflows/test.yml:57`

### github-env-injection (severity: high)

In entrypoint.sh, the setOutput() function writes values to $GITHUB_OUTPUT using `echo "${1}=${2}" >> "${GITHUB_OUTPUT}"` without any newline sanitization. The values written — including $custom_tag (from CUSTOM_TAG env var), $tag, $new, $part, $default_semvar_bump (from DEFAULT_BUMP), and $suffix (from PRERELEASE_SUFFIX) — are all derived from environment variables set by the calling workflow. These are inherited process env vars that a calling workflow can set to arbitrary values, including values containing newlines that could inject additional key=value pairs into GITHUB_OUTPUT. The required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`) is absent before every write.

Locations:

- `entrypoint.sh:68`
- `entrypoint.sh:69`
- `entrypoint.sh:70`
- `entrypoint.sh:71`
- `entrypoint.sh:72`
- `entrypoint.sh:73`
- `entrypoint.sh:74`
- `entrypoint.sh:75`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings:

1. unpinned-uses: Pinned all 7 action references across lint.yml, main.yml, and test.yml to full 40-character SHA digests with original tag/branch preserved as comments.

2. script-injection: In test.yml's 'Check if the tag would have been created' step, moved all 21 ${{ steps.*.outputs.* }} expressions from direct shell interpolation in the run: block into the step's env: block. The shell script now references them as plain environment variables (e.g., $MAIN1_OUTPUT_TAG instead of ${{ steps.test_main1.outputs.old_tag }}).

3. github-env-injection: Updated the setOutput() function in entrypoint.sh to sanitize values before writing to $GITHUB_OUTPUT. Added `local safe_value; safe_value=$(printf '%s' "${2}" | tr -d '\n\r')` before the echo, preventing newline injection attacks that could inject additional key=value pairs into GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansions in verlt() function calls in .github/workflows/test.yml. All five verlt call sites now properly quote their arguments (MAIN1, PRE1, MAIN2, PRE2, MAIN5), and the internal verlte call within verlt() also now quotes $1 and $2. This prevents attacker-controlled tag values containing shell metacharacters from causing command injection.

