<!-- markdownlint-disable -->

# Hardening Report: Azure--webapps-deploy/v2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Azure--webapps-deploy/v2** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks in release.yml directly interpolate ${{ inputs.tag }} and ${{ steps.version.outputs.* }} expressions into shell commands (rule a). YAML template substitution occurs before the shell parses the string, so a crafted tag value could inject arbitrary shell commands. Affected steps: 'Validate tag format' (line 25: TAG="${{ inputs.tag }}"), 'Parse version' (line 34: TAG="${{ inputs.tag }}"), 'Release new tag' (line 53: MAJOR="${{ steps.version.outputs.major }}"; line 56: git tag -fa ${MAJOR} -m "Release ${{ inputs.tag }}"), 'Revert to existing tag' (line 66: BRANCH="${{ steps.version.outputs.branch }}"; line 67: git push origin ${{ inputs.tag }}:refs/heads/${BRANCH} --force). Fix: move all ${{ inputs.* }} and ${{ steps.*.outputs.* }} values into env: variables and reference them as quoted shell variables.

Locations:

- `.github/workflows/release.yml:25`
- `.github/workflows/release.yml:34`
- `.github/workflows/release.yml:53`
- `.github/workflows/release.yml:56`
- `.github/workflows/release.yml:66`
- `.github/workflows/release.yml:67`

### github-env-injection (severity: high)

In the 'Parse version' step of release.yml, the variable MAJOR is derived from ${{ inputs.tag }} (an untrusted workflow_dispatch input) and written to $GITHUB_OUTPUT without sanitization (no printf '%s' ... | tr -d '\n\r' applied). A newline character in the input could inject additional key=value pairs into the output file, poisoning downstream step outputs. Line 35: MAJOR=$(echo "$TAG" | cut -d. -f1) where TAG comes from ${{ inputs.tag }}. Line 36: echo "major=${MAJOR}" >> $GITHUB_OUTPUT (unsanitized). Line 37: echo "branch=releases/${MAJOR}" >> $GITHUB_OUTPUT (unsanitized).

Locations:

- `.github/workflows/release.yml:36`
- `.github/workflows/release.yml:37`

### unpinned-uses (severity: high)

All uses: references across every workflow file use mutable tag or branch refs instead of immutable 40-character commit SHAs, exposing workflows to supply-chain attacks if any referenced action is compromised or its tag is moved. Failing references: ci.yml: actions/checkout@v2; defaultLabels.yml: actions/stale@v3 (x2); github_actions_test_v2.yml: actions/checkout@v4, actions/setup-dotnet@v4, actions/upload-artifact@v4 (x2), actions/setup-node@v3, azure/webapps-deploy@releases/v2 (x2); github_actions_test_v3.yml: actions/checkout@v4, actions/setup-dotnet@v4, actions/upload-artifact@v4 (x2), actions/setup-node@v3, azure/login@v2 (x2), azure/webapps-deploy@releases/v3 (x2); pr_check_webapp_dotnet_windows.yml: actions/checkout@v2 (x2), actions/checkout@master, nuget/setup-nuget@v1, microsoft/setup-msbuild@v1.0.2, azure/login@v2; pr_check_windows_container_pubprofile.yml: actions/checkout@master, azure/login@v2, azure/docker-login@v1, Azure/appservice-settings@v1, actions/checkout@v2; release.yml: actions/checkout@v4.

Locations:

- `.github/workflows/ci.yml:16`
- `.github/workflows/defaultLabels.yml:14`
- `.github/workflows/defaultLabels.yml:25`
- `.github/workflows/github_actions_test_v2.yml:20`
- `.github/workflows/github_actions_test_v3.yml:20`
- `.github/workflows/pr_check_webapp_dotnet_windows.yml:33`
- `.github/workflows/pr_check_windows_container_pubprofile.yml:29`
- `.github/workflows/release.yml:40`

### missing-permissions (severity: medium)

Two workflow files have no permissions: key at the top level and no job-level permissions: blocks on any of their jobs. Without an explicit permissions declaration the workflow inherits the repository default token permissions which may be overly broad. Affected: ci.yml (no permissions key anywhere) and defaultLabels.yml (no permissions key anywhere).

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/defaultLabels.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across 6 workflow files:

1. script-injection (release.yml): Moved all ${{ inputs.tag }} and ${{ steps.version.outputs.* }} expressions into step-level env: blocks. Shell scripts now reference plain environment variables ($INPUT_TAG, $MAJOR, $VERSION_BRANCH, $BRANCH) with proper double-quoting.

2. github-env-injection (release.yml): Added sanitization in the 'Parse version' step using `printf '%s' "$MAJOR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT to prevent newline injection.

3. unpinned-uses: Pinned all mutable tag/branch references to immutable 40-character commit SHAs across all 6 affected workflow files (ci.yml, defaultLabels.yml, github_actions_test_v2.yml, github_actions_test_v3.yml, pr_check_webapp_dotnet_windows.yml, pr_check_windows_container_pubprofile.yml, release.yml). All SHAs were resolved via lookup_action_sha.

4. missing-permissions: Added `permissions: contents: read` to ci.yml and `permissions: issues: write / pull-requests: write` to defaultLabels.yml (minimum permissions needed for the stale action to function).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/pr_check_windows_container_pubprofile.yml. The `${{ env.CONTAINER_REGISTRY }}` expression was moved out of the `run:` shell commands (lines 58-59) and into a step-level `env:` block. The docker tag and docker push commands now reference the value as the quoted shell variable `"$CONTAINER_REGISTRY"` instead of directly interpolating the `${{ }}` expression.

