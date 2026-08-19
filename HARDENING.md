<!-- markdownlint-disable -->

# Hardening Report: Azure--webapps-deploy/v3.0.8

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **Azure--webapps-deploy/v3.0.8** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks in release.yml directly interpolate ${{ inputs.tag }} and ${{ steps.version.outputs.* }} expressions inside shell commands, enabling script injection. An attacker-controlled workflow_dispatch input value is substituted into the shell before quoting takes effect. Offending lines include: `TAG="${{ inputs.tag }}"` (lines 22, 30), `git tag -fa ${MAJOR} -m "Release ${{ inputs.tag }}"` (line 50), `git push origin ${{ inputs.tag }}:refs/heads/${BRANCH} --force` (line 60), `MAJOR="${{ steps.version.outputs.major }}"` (line 47), and `BRANCH="${{ steps.version.outputs.branch }}"` (line 59). Sub-rule (a) violated.

Locations:

- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:30`
- `.github/workflows/release.yml:47`
- `.github/workflows/release.yml:50`
- `.github/workflows/release.yml:59`
- `.github/workflows/release.yml:60`

### github-env-injection (severity: high)

In release.yml, the 'Parse version' step writes values derived from the untrusted input ${{ inputs.tag }} to $GITHUB_OUTPUT without sanitization. The value is assigned to TAG via `TAG="${{ inputs.tag }}"`, then MAJOR is derived from TAG and written as `echo "major=${MAJOR}" >> $GITHUB_OUTPUT` and `echo "branch=releases/${MAJOR}" >> $GITHUB_OUTPUT`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, allowing newline injection into the output file.

Locations:

- `.github/workflows/release.yml:30`

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of pinned 40-character commit SHAs, making them vulnerable to supply-chain attacks if the referenced tag or branch is moved.

ci.yml: `actions/checkout@v2`

defaultLabels.yml: `actions/stale@v3` (×2)

github_actions_test_v2.yml: `actions/checkout@v4`, `actions/setup-dotnet@v4`, `actions/upload-artifact@v4`, `actions/setup-node@v3`, `azure/webapps-deploy@releases/v2` (×2)

github_actions_test_v3.yml: `actions/checkout@v4`, `actions/setup-dotnet@v4`, `actions/upload-artifact@v4`, `azure/login@v2`, `azure/webapps-deploy@releases/v3`, `actions/setup-node@v3` (×2)

pr_check_webapp_dotnet_windows.yml: `actions/checkout@v2`, `actions/checkout@master`, `nuget/setup-nuget@v1`, `microsoft/setup-msbuild@v1.0.2`, `actions/checkout@v2`, `azure/login@v2`

pr_check_windows_container_pubprofile.yml: `actions/checkout@master`, `azure/login@v2`, `azure/docker-login@v1`, `Azure/appservice-settings@v1`, `actions/checkout@v2`

release.yml: `actions/checkout@v4`

Locations:

- `.github/workflows/ci.yml:14`
- `.github/workflows/defaultLabels.yml:13`
- `.github/workflows/github_actions_test_v2.yml:20`
- `.github/workflows/github_actions_test_v3.yml:20`
- `.github/workflows/pr_check_webapp_dotnet_windows.yml:30`
- `.github/workflows/pr_check_windows_container_pubprofile.yml:22`
- `.github/workflows/release.yml:36`

### missing-permissions (severity: medium)

ci.yml has no top-level `permissions:` key and its only job (`build_test_job`) also has no `permissions:` key, so the workflow runs with the default (potentially broad) token permissions.

defaultLabels.yml has no top-level `permissions:` key and its only job (`build`) also has no `permissions:` key, so the workflow runs with the default token permissions.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/defaultLabels.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across multiple workflow files:

1. **script-injection** (release.yml): Moved all `${{ inputs.tag }}` and `${{ steps.version.outputs.* }}` expressions into step-level `env:` blocks (INPUT_TAG, MAJOR, BRANCH, VERSION_BRANCH) and referenced them as plain shell variables in run scripts.

2. **github-env-injection** (release.yml): In the 'Parse version' step, sanitized the derived MAJOR value with `printf '%s' "$MAJOR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

3. **unpinned-uses**: Pinned all unpinned action references to full 40-char SHAs with original tag in comment:
   - ci.yml: actions/checkout@v2 → @0717577d...
   - defaultLabels.yml: actions/stale@v3 (×2) → @98ed4cb5...
   - github_actions_test_v2.yml: actions/checkout@v4, actions/setup-dotnet@v4, actions/upload-artifact@v4 (×2), actions/setup-node@v3, azure/webapps-deploy@releases/v2 (×2)
   - github_actions_test_v3.yml: actions/checkout@v4 (×2), actions/setup-dotnet@v4, actions/upload-artifact@v4 (×2), azure/login@v2 (×2), azure/webapps-deploy@releases/v3 (×2), actions/setup-node@v3
   - pr_check_webapp_dotnet_windows.yml: actions/checkout@v2 (×2), actions/checkout@master, nuget/setup-nuget@v1, microsoft/setup-msbuild@v1.0.2, azure/login@v2
   - pr_check_windows_container_pubprofile.yml: actions/checkout@master, azure/login@v2, azure/docker-login@v1, Azure/appservice-settings@v1, actions/checkout@v2
   - release.yml: actions/checkout@v4

4. **missing-permissions**: Added `permissions: {}` to ci.yml and `permissions: { issues: write, pull-requests: write }` to defaultLabels.yml.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/pr_check_windows_container_pubprofile.yml: moved `${{ env.CONTAINER_REGISTRY }}` out of the `run:` shell block in the 'Pull and tag an image and push it to ACR' step. Added an `env:` block to the step with `CONTAINER_REGISTRY: ${{ env.CONTAINER_REGISTRY }}` and updated the docker tag and docker push commands to reference `"$CONTAINER_REGISTRY"` as a shell variable instead of using the template expression directly.

