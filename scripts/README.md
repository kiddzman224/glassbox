# Scripts Directory

This directory contains utility scripts for testing, verification, and automation.

## Available Scripts

### `verify_issues.sh`

Automated verification script for GitHub issues and local `issues.md` files.

**Purpose**: Verifies that all issues are created correctly with proper labels,
required content sections, and no duplicate titles.  The expected issue count
is configurable to support different backlog wave sizes (default: **120**).

**Prerequisites**:
1. Install GitHub CLI: https://cli.github.com/
   ```bash
   # macOS
   brew install gh
   
   # Linux
   sudo apt install gh
   ```

2. Authenticate with GitHub:
   ```bash
   gh auth login
   ```

**Usage**:

```bash
# Basic verification (expects 120 issues with label new_for_wave)
./scripts/verify_issues.sh

# Override expected count at runtime (e.g. a 40-item wave)
./scripts/verify_issues.sh --count 40

# Override via environment variable (useful in CI)
GLASSBOX_ISSUE_COUNT=40 ./scripts/verify_issues.sh

# Verify and export fetched issues to JSON
./scripts/verify_issues.sh --export

# Validate only a local issues.md without network access
./scripts/verify_issues.sh --local-only --file issues.md

# Combine: check GitHub AND validate a local file
./scripts/verify_issues.sh --file issues.md
```

**Options**:

| Flag | Description | Default |
|------|-------------|---------|
| `--count N` | Expected issue count | `120` (or `$GLASSBOX_ISSUE_COUNT`) |
| `--label LABEL` | GitHub label to filter on | `new_for_wave` |
| `--repo OWNER/REPO` | Target repository | `pugsley76/glassbox` |
| `--file PATH` | Local issues.md to parse | auto-detected if `issues.md` exists |
| `--export` | Write fetched issues to `issues_export.json` | off |
| `--local-only` | Skip GitHub API; only validate `--file` | off |

**What it checks**:
- Issue count matches expected value
- All issues have the `new_for_wave` label
- No duplicate issue titles
- Required sections present in every issue (no checkbox syntax required):
  - `Description`
  - `Work to be done`
  - `Implementation procedure`
  - `Acceptance criteria`

**Troubleshooting**:

- **Error: GitHub CLI not installed**
  ```
  Install from: https://cli.github.com/
  ```

- **Error: Not authenticated**
  ```bash
  gh auth login
  ```

- **Error: Issue count mismatch**
  - Check if all issues were created
  - Override with `--count N` or `GLASSBOX_ISSUE_COUNT=N` for a different wave size
  - Verify the label name is correct

## Release scripts

The following scripts are used in the release pipeline and can be run locally
to verify or reproduce release artifacts.

### `verify-release.sh`

Smoke-tests release artifacts in `dist/release/` (or a directory you supply).

**Checks performed:**
1. All five expected binaries exist and are non-empty:
   `glassbox-linux-amd64`, `glassbox-linux-arm64`,
   `glassbox-darwin-amd64`, `glassbox-darwin-arm64`,
   `glassbox-windows-amd64.exe`
2. Each binary is exercised with five offline commands (`--version`, `--help`,
   `debug --demo`, `debug --dry-run --help`, `version --json`) on the native
   platform. Cross-platform binaries are run via `wine` (Windows on Linux) or
   `qemu-*-static` (cross-arch Linux) when available; otherwise reported as
   `[SKIP-UNAVAILABLE]` — not a failure.
3. `checksums.sha256` exists and all entries pass `sha256sum --check`.
4. `version.txt` contains non-empty `version`, `commit`, and `build_date` fields.

**Usage:**
```bash
# Run against the default dist/release directory
bash scripts/verify-release.sh

# Run against a custom directory
bash scripts/verify-release.sh /path/to/dist
```

**Called by:** `make verify-release` and the `package` job in
`.github/workflows/release.yml`.

**Expected exit codes:** `0` all checks passed, `1` one or more checks failed.

---

### `check-reproducibility.sh`

Builds a release binary twice in fully isolated `GOPATH`/`GOCACHE`
environments and compares SHA-256 hashes to confirm byte-identical output.

**Supported targets:** `linux-amd64` (default), `linux-arm64`, `darwin-amd64`,
`darwin-arm64`, `windows-amd64`.

**Usage:**
```bash
# Default: linux-amd64
make reproducibility-check
bash scripts/check-reproducibility.sh

# Specific target
bash scripts/check-reproducibility.sh darwin-arm64

# Keep temp dirs for diffoscope inspection
KEEP_BUILD_DIRS=1 bash scripts/check-reproducibility.sh linux-amd64
```

**Expected output (success):**
```
Reproducibility check
  Target             : linux-amd64 (linux/amd64)
  SOURCE_DATE_EPOCH  : 1700000000
  ...
  [PASS] Binary identical: 4f3e2a1b...
  [PASS] Archive identical: 9c8d7e6f...
Result: builds are reproducible for target linux-amd64.
```

See [docs/reproducible-builds.md](../docs/reproducible-builds.md) for the full
troubleshooting guide.

---

### `verify-manifest.sh`

Offline verification of a signed release `manifest.json`. Requires only
`bash`, `python3`, and optionally `pip install cryptography` for the Ed25519
signature step.

**Usage:**
```bash
# Verify manifest and artifacts in dist/release
bash scripts/verify-manifest.sh dist/release/manifest.json dist/release

# Verify after downloading a release
curl -LO https://github.com/dotandev/glassbox/releases/download/v1.2.3/manifest.json
pip install cryptography
bash scripts/verify-manifest.sh manifest.json .
```

**Expected exit codes:** `0` all checks passed, `1` one or more checks failed.

---

### `verify_protocol_registration_windows.ps1`

Smoke-tests the `glassbox://` custom URL protocol handler registry entries on
Windows. Verifies that `HKCU\Software\Classes\glassbox` and its
`\shell\open\command` sub-key exist, that the `URL Protocol` value is present,
and that the command string invokes `protocol-handler`.

Set `GLASSBOX_BINARY` to verify the command references a specific binary path.

**Usage:**
```powershell
powershell -ExecutionPolicy Bypass -File scripts\verify_protocol_registration_windows.ps1

# With binary path check
$env:GLASSBOX_BINARY = "C:\Program Files\Glassbox\glassbox.exe"
powershell -ExecutionPolicy Bypass -File scripts\verify_protocol_registration_windows.ps1
```

---

### `verify_protocol_registration_macos.sh`

Smoke-tests the `glassbox://` protocol handler on macOS. Checks the `.app`
bundle plist, the helper executable, and (when `lsregister` is available) the
Launch Services database.

**Usage:**
```bash
bash scripts/verify_protocol_registration_macos.sh
GLASSBOX_BINARY=/usr/local/bin/glassbox bash scripts/verify_protocol_registration_macos.sh
```

---

### `verify_protocol_registration_linux.sh`

Smoke-tests the `glassbox://` protocol handler on Linux. Checks the
`.desktop` file, the helper script, and the `xdg-mime` default handler.

**Usage:**
```bash
bash scripts/verify_protocol_registration_linux.sh
GLASSBOX_BINARY=/usr/local/bin/glassbox bash scripts/verify_protocol_registration_linux.sh
```

---

### `index_regression_fixtures.sh`

Builds and validates a repository-wide regression fixture index at
`test/regression/fixture_index.json`.

**Purpose**: Provides a single authoritative catalogue of every fixture across
all seven regression layers (`rpc`, `trace`, `sourcemap`, `session`, `audit`,
`replay`, `cli`).  The index records each fixture's path, layer, failure class,
issue/PR reference, schema version, and test name.  CI fails on orphaned
entries, duplicates, or missing issue references.

**Usage**:

```bash
# Regenerate the index
./scripts/index_regression_fixtures.sh

# Validate an existing index without regenerating (CI mode)
./scripts/index_regression_fixtures.sh --check-only

# Use a non-default fixtures directory
./scripts/index_regression_fixtures.sh --fixtures-dir my/fixtures

# Write index to a custom location
./scripts/index_regression_fixtures.sh --output /tmp/index.json
```

**What it checks**:
- Every fixture has a recognised layer name
- Every fixture filename contains an issue or PR reference (e.g. `issue123`)
- No duplicate fixture IDs within the same layer
- All indexed files exist on disk

---



If you prefer manual verification using GitHub API:

```bash
# Check issue count
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/dotandev/glassbox/issues?labels=new_for_wave&per_page=100" \
  | jq 'length'

# Get all issues with label
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/dotandev/glassbox/issues?labels=new_for_wave&per_page=100" \
  | jq '.[] | {number, title, labels: [.labels[].name]}'

# Export to file
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/dotandev/glassbox/issues?labels=new_for_wave&per_page=100" \
  > issues.json
```

## Using GitHub CLI Directly

```bash
# List all issues with label
gh issue list --repo dotandev/glassbox --label new_for_wave

# Count issues
gh issue list --repo dotandev/glassbox --label new_for_wave --json number --jq 'length'

# View specific issue
gh issue view 123 --repo dotandev/glassbox

# Export issues to JSON
gh issue list --repo dotandev/glassbox --label new_for_wave --limit 100 \
  --json number,title,labels,body > issues_export.json
```

## Quick Start

1. **Install GitHub CLI** (if not already installed):
   ```bash
   brew install gh  # macOS
   ```

2. **Authenticate**:
   ```bash
   gh auth login
   ```

3. **Run verification**:
   ```bash
   cd /path/to/glassbox
   ./scripts/verify_issues.sh
   ```

4. **Check results**:
   - Green checkmarks ([OK]) = passed
   - Red X ([FAIL]) = failed
   - Script exits with code 0 on success, 1 on failure

## CI/CD Integration

To use this script in CI/CD pipelines:

```yaml
# GitHub Actions example
- name: Verify Issues
  run: |
    ./scripts/verify_issues.sh
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Adding New Scripts

When adding new scripts to this directory:

1. Make them executable:
   ```bash
   chmod +x scripts/your_script.sh
   ```

2. Add a shebang line:
   ```bash
   #!/bin/bash
   ```

3. Add copyright header:
   ```bash
   # Copyright 2026 Glassbox Users
   # SPDX-License-Identifier: Apache-2.0
   ```

4. Update this README with usage instructions

5. Add error handling:
   ```bash
   set -e  # Exit on error
   ```

## Support

For issues with scripts:
1. Check prerequisites are installed
2. Verify authentication
3. Check script permissions (`chmod +x`)
4. Review error messages
5. Open an issue on GitHub if problems persist
