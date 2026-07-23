# `scanners/`

Scanners produce **Findings / snapshots / time-series / inventory** on a cadence, into the Observatory.

## Layout

```
scanners/<org-slug>/<name>/
  manifest.yaml   # axis: scanner
  README.md       # optional, per scanner
```

`<org-slug>` must equal your org slug exactly — see [CONTRIBUTING.md → Folder shape](../CONTRIBUTING.md#folder-shape).

- **Acceptance criteria:** [CONTRIBUTING.md → `scanners/`](../CONTRIBUTING.md#scanners)
- **Working example:** [`lace-cloud/registry`](https://github.com/lace-cloud/registry)

This README documents the directory and keeps it in git. Drop your first scanner alongside it.
