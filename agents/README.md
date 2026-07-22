# `agents/`

Agents read a producer and emit an **Advisory** at one of the four Manifestations (preview, draft, triage, trace). They never return a Verdict and never gate.

## Layout

```
agents/<org-slug>/<name>/
  manifest.yaml   # axis: agent
  README.md       # optional, per agent
```

`<org-slug>` must equal your org slug exactly — see [CONTRIBUTING.md → Folder shape](../CONTRIBUTING.md#folder-shape).

- **Acceptance criteria:** [CONTRIBUTING.md → `agents/`](../CONTRIBUTING.md#agents)
- **Egress consent** (bind-time, enforced): [CONTRIBUTING.md → Egress consent](../CONTRIBUTING.md#egress-consent-readsscope-decides-how-hard-your-artifact-is-to-install)
- **Working example:** [`lace-cloud/registry`](https://github.com/lace-cloud/registry)

This README documents the directory and keeps it in git. Drop your first agent alongside it.
