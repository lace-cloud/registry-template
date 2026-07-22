# `modules/`

Terraform modules — the **desired state** an artifact contributes. The Lace runner applies them.

## Layout

```
modules/<org-slug>/<name>/
  manifest.yaml   # axis: module
  main.tf         # required
  ...             # variables.tf, outputs.tf, etc.
```

`<org-slug>` must equal your org slug exactly — see [CONTRIBUTING.md → Folder shape](../CONTRIBUTING.md#folder-shape).

- **Acceptance criteria:** [CONTRIBUTING.md → `modules/`](../CONTRIBUTING.md#modules)
- **Working example:** [`lace-cloud/registry`](https://github.com/lace-cloud/registry)

This README documents the directory and keeps it in git. Drop your first module alongside it.
