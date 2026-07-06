# Contributing

This repo holds your org's private Lace registry manifests. PRs are reviewed by the team(s) named in `.github/CODEOWNERS`.

## Folder shape

```
<axis>/<author>/<name>/manifest.yaml
<axis>/<author>/<name>/README.md
```

`<axis>` ∈ {`modules`, `handlers`, `scanners`, `agents`}. `<author>` is your org's slug or a team-internal namespace (e.g. `acme`, `acme-platform`, `acme-security`). The `(axis, author, name)` tuple must be unique within this repo.

Modules carry `*.tf` files alongside `manifest.yaml` and may nest under a cloud-system tier (e.g. `modules/aws/<name>/main.tf`). Non-module axes are manifest + README only.

## Manifest envelope

Every artifact ships a `manifest.yaml` with this base envelope:

```yaml
apiVersion: '1'
axis: module | handler | scanner | agent
author: acme                       # kebab-case (1-64 chars)
name: internal-vpc                 # kebab-case (1-64 chars), unique within axis+author
version: v1.0.0                    # semver, must be 'v'-prefixed
displayName: "Acme Internal VPC"   # human-readable (1-96 chars)
description: "..."
categories: [networking, internal]
configSchema: { ... }              # JSON-Schema for per-install config
runtime:
  location: customer-hosted        # private manifests CANNOT use 'lace-managed' (ADR 0004)
  dispatch: lace-pull | customer-push | customer-poll
  ...
authors: ["Acme Platform <platform@acme.example>"]
# Axis-specific fields layered on top — see per-axis acceptance criteria below.
```

The deep validation runs server-side. The CI envelope check is fail-fast.

## Per-axis acceptance criteria

### `modules/`

- `axis: module`. Modules may declare `runtime.location: lace-managed` (Terraform applied by the Lace runner) or `customer-hosted`; ADR 0004 only restricts in-tree handler/agent dispatch, not modules.
- `main.tf` exists. `terraform fmt -check` passes. `terraform validate` passes after `terraform init -backend=false`.
- `version` bumped on any `*.tf` change.
- A `bundle:` block is **required**. Its fields:

  | Field | Required | Type | Meaning |
  |---|---|---|---|
  | `bundle.system` | **yes** | string (1–32) | Cloud/IaC system tag — `aws`, `gcp`, `cloudflare`, `docker`, … |
  | `bundle.gitUrl` | **yes** | URL (≤512) | Fetchable source git URL. Terraform fetches the module from here; inline HCL alone is not enough, so a `/source` read fails with HTTP 400 when it's absent. |
  | `bundle.modulePath` | **yes** | string (1–512) | Path to the module under the repo root (e.g. `modules/aws/internal-vpc`). |
  | `bundle.commitSha` | no | string (7–64) | Source commit pin. |
  | `bundle.gitTag` | no | string (1–128) | Source tag pin. |

  Example:

  ```yaml
  bundle:
    system: aws
    gitUrl: https://github.com/acme-corp/lace-registry.git
    modulePath: modules/acme/internal-vpc
    commitSha: '1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b'
  ```

### `handlers/`

- `axis: handler`. `runtime.location: customer-hosted` (private manifests cannot use `lace-managed`).
- `hooks.available` ⊆ {`pre_plan`, `post_plan`, `pre_apply`, `post_apply`, `post_destroy`}, ≥ 1 entry. `hooks.default` ⊆ `hooks.available`.
- `endpointSource` ∈ {`manifest`, `install`}. `endpointSource: 'in_tree'` is reserved for `author: lace` in the public registry (ADR 0004).
  - `manifest`: declare `endpointUrl` (single URL every install dispatches to).
  - `install`: omit `endpointUrl`; each org supplies its own URL at install time.
- `signing: { kind: hmac_sha256 }` for any handler carrying secrets, or `{ kind: none }`.
- `verdictMode` ∈ {`sync`, `async`}.
- `timeoutSeconds` (optional, 1–600, default 30) sets the per-handler default timeout.

### `scanners/`

- `axis: scanner`. Customer-hosted: `runtime: { location: customer-hosted, dispatch: lace-pull, oidcTrust: { issuer, audience } }` (Lace dispatches HTTPS to your endpoint with an OIDC-signed JWT), `runtime: { location: customer-hosted, dispatch: customer-push, scopeRequired: <scope> }` (your endpoint POSTs results to Lace), or `runtime: { location: customer-hosted, dispatch: customer-poll, ... }` (your worker long-polls a Lace queue).
- `outputs` declares ≥ 1 of `findings | snapshots | time_series | inventory` with sub-kind schemas.
- `defaultIntervalSeconds` (optional, positive integer) sets the default scan cadence in seconds. When an install omits it, the scheduler falls back to the 300s floor — set it so an admin who installs without overriding gets a fire-able install instead of a silent never-fires row. It is clamped to `[300, 86400]` (5 min – 24 h) at schedule time. An install may override per-install via `configJson.intervalSeconds` (same clamp applies).

### `agents/`

- `axis: agent`. `runtime.location: customer-hosted` (private manifests cannot use `lace-managed`).
- `trigger` declares the timeline producer the agent reads and the hooks it subscribes to:
  - `trigger.producer` ∈ {`run`, `handler`, `scanner`, `agent`}.
  - For `producer: run`, `trigger.hooks.available` ⊆ {`pre_plan`, `post_plan`, `pre_apply`, `post_apply`, `post_destroy`} and must be non-empty. `trigger.hooks.default` ⊆ `trigger.hooks.available`.
  - For any other producer, `trigger.hooks.available` is empty (the agent fires on the Event itself, filtered by payload).
- `emits.surface` ∈ {`whisperer`, `triage`} — the advisory surface the agent writes to.
- Customer-hosted agents MUST declare a `reads` descriptor for egress governance:
  ```yaml
  reads:
    scope: org | team | stack
    kinds: [run_history, plan_diff]
  ```
- `runtime.dispatch` must be `lace-pull` or `customer-poll`; `customer-push` is not valid for agents (agents emit advisories, they do not return verdicts via push webhook).
- For `customer-poll`, `runtime.pollScopeRequired` must be exactly `agents:poll`.

## Review checklist

For every PR:

- [ ] Envelope fields present (apiVersion, axis, author, name, version, displayName, configSchema, runtime).
- [ ] `version` is a fresh `v`-prefixed semver, bumped vs. base branch.
- [ ] `(axis, author, name)` is unique across the repo.
- [ ] `runtime.location: customer-hosted` for handlers, scanners, and agents (private manifests cannot use `lace-managed`).
- [ ] Per-axis fields conform to the schema referenced above.
- [ ] `README.md` exists alongside `manifest.yaml` and explains usage.
- [ ] Modules: `terraform fmt -check` + `terraform init -backend=false && terraform validate` pass.
- [ ] No secrets, credentials, or production data in any manifest or README.

## Manifest immutability

Once a `(axis, author, name, version)` row lands in `registry_manifest`, its content is frozen. Re-publishing the same version with different content is rejected with HTTP 409. To ship a fix, bump `version`.

## Visibility

Manifests published from this repo carry `org_id = <your org>`. They are visible only to your org's catalog browse. Public manifests authored by Lace at `lace-cloud/registry` (carrying `org_id = NULL`) are also visible to your catalog browse, but you cannot publish to that namespace from here.
