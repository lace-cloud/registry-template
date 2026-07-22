# `handlers/`

Handlers subscribe to Run-lifecycle hooks and return a **Verdict**. Only `pre_apply` gates a Run; every other hook is observing-only.

## Layout

```
handlers/<org-slug>/<name>/
  manifest.yaml   # axis: handler
  README.md       # optional, per handler
```

`<org-slug>` must equal your org slug exactly — see [CONTRIBUTING.md → Folder shape](../CONTRIBUTING.md#folder-shape).

- **Acceptance criteria:** [CONTRIBUTING.md → `handlers/`](../CONTRIBUTING.md#handlers)
- **Working example:** [`lace-cloud/registry`](https://github.com/lace-cloud/registry)

This README documents the directory and keeps it in git. Drop your first handler alongside it.
