# Update Playwright Snapshots from CI

Update Playwright test snapshots from failed CI test reports. This skill extracts the "actual" images from snapshot mismatch failures and updates the baseline snapshots in the repository.

## Usage

Invoke with `/update-playwright-snapshots` or ask "update playwright snapshots from CI".

## Process

### 1. Identify the PR and Failed Tests

First, get the current PR information and find failed UI test workflow runs:

```bash
# Get PR info and check for failed UI test jobs
gh pr view --json number,headRefName,statusCheckRollup
```

Look for checks with:
- `conclusion: "FAILURE"`
- Names containing "ui-test", "playwright", "e2e", or "snapshot"

### 2. Get Workflow Artifacts

For each failed workflow run, list available artifacts:

```bash
gh api repos/{owner}/{repo}/actions/runs/{run_id}/artifacts
```

Look for artifacts matching these patterns:
- `*-test-report` - HTML test reports
- `*-updated-snapshots` - Pre-extracted actual images (if available)
- `*-test-assets` - Test output assets

### 3. Download and Analyze Reports

Download the test report artifacts:

```bash
gh run download {run_id} -n {artifact-name} -D /tmp/jp-claude-plugin-{run_id}-{artifact-name}
```

### 4. Identify Snapshot Mismatches

Snapshot mismatch failures are identified by:
- PNG files in the report's `data/` directory
- These come in sets of 3: expected, actual, and diff images
- The "actual" image is what the test produced and should become the new baseline

To match report PNGs to snapshot files:

```bash
# For each PNG in the report data folder
for f in /tmp/jp-claude-plugin-{run_id}-{report}/data/*.png; do
  md5_report=$(md5 -q "$f" 2>/dev/null || md5sum "$f" | cut -d' ' -f1)

  # Compare against updated-snapshots artifact (if available)
  for s in /tmp/jp-claude-plugin-{run_id}-{updated-snapshots}/*-snapshots/*.png; do
    md5_snap=$(md5 -q "$s" 2>/dev/null || md5sum "$s" | cut -d' ' -f1)
    if [ "$md5_report" = "$md5_snap" ]; then
      echo "MATCH: $f -> $s"
    fi
  done
done
```

### 5. Copy Updated Snapshots

Only copy files that:
1. Are "actual" images (match files in updated-snapshots artifact)
2. Correspond to snapshot mismatch failures (not other test failures)

Find the snapshot directory in the repo (common patterns):
- `ui-tests/test/*.spec.ts-snapshots/`
- `e2e/*.spec.ts-snapshots/`
- `tests/*.spec.ts-snapshots/`
- `**/*-snapshots/`

```bash
# Find snapshot directories
find . -type d -name "*-snapshots" | head -10
```

Copy matched files:
```bash
cp -v /tmp/jp-claude-plugin-{run_id}-{report}/data/{hash}.png {repo}/path/to/snapshots/{snapshot-name}.png
```

### 6. Verify Changes

```bash
git status
```

## Important Notes

- Only update snapshots from **snapshot mismatch** failures, not from other test failures
- The HTML report `data/` folder contains 3 images per failed snapshot test:
  - Expected (the current baseline)
  - Actual (what the test produced - this is what you want)
  - Diff (visual difference highlighting)
- Match by MD5 hash to identify which report PNG corresponds to which snapshot
- If an `updated-snapshots` artifact exists, it contains pre-extracted actual images with proper filenames
- Always verify the changes make sense before committing

## Artifact Naming Conventions

Common artifact name patterns by project:
- JupyterLab/Notebook: `notebook-{browser}-test-report`, `notebook-{browser}-updated-snapshots`
- Generic: `playwright-report`, `test-results`, `snapshots`

## Browser Variants

Snapshots are typically browser-specific:
- `*-chromium-linux.png`
- `*-firefox-linux.png`
- `*-webkit-linux.png`

Process each browser's report separately.
