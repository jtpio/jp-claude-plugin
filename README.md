# jp-claude-plugin

Claude Code plugin for maintaining Jupyter ecosystem repositories.

## Installation

```bash
/plugin marketplace add jtpio/jp-claude-plugin
/plugin install jupyter-maintainer@jp-claude-plugin
```

## Commands

| Command | Description |
|---------|-------------|
| `/update-playwright-snapshots` | Update Playwright test snapshots from failed CI test reports |

## `/update-playwright-snapshots`

Extracts "actual" images from snapshot mismatch failures in CI and updates the baseline snapshots in the repository.

- Finds failed UI test workflow runs for the current PR
- Downloads test report artifacts
- Matches report PNGs to snapshot files by MD5 hash
- Copies updated snapshots to the correct locations

Works with JupyterLab, Jupyter Notebook, and other Jupyter repos using Playwright for UI testing.
