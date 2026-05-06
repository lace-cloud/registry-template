# Contributing

This repo holds your org's private Lace registry manifests. PRs are reviewed by the team(s) named in `.github/CODEOWNERS`.

## Folder shape

```
<axis>/<author>/<name>/manifest.yaml
<axis>/<author>/<name>/README.md
```

`<axis>` ∈ {`modules`, `scanners`, `run-tasks`, `chaos-providers`}. `<author>` is your org's slug or a team-internal namespace (e.g. `acme`, `acme-platform`, `acme-security`). The `(axis, author, name)` tuple must be unique within this repo.

Modules carry `*.tf` files alongside `manifest.yaml` and may nest under a cloud-system tier (e.g. `modules/aws/<name>/main.tf`). Non-module axes are manifest + README only.

## Manifest envelope

Every artifact ships a `manifest.yaml` with this base envelope:

```yaml
apiVersion: '1'
axis: module | scanner | run_task | chaos_provider
author: acme                       # kebab-case (1-64 chars)
name: internal-vpc                 # kebab-case (1-64 chars), unique within axis+author
version: v1.0.0                    # semver, must be 'v'-prefixed
displayName: "Acme Internal VPC"   # human-readable (1-96 chars)
description: "..."
categories: [networking, internal]
configSchema: { ... }              # JSON-Schema for per-install config
runtime:
  location: customer-hosted        # private manifests CANNOT use 'lace-managed' (ADR 0004)
  dispatch: lace-pull | customer-push
  ...
authors: ["Acme Platform <platform@acme.example>"]
# Axis-specific fields layered on top — see per-axis acceptance criteria below.
```

The deep validation runs server-side. The CI envelope check is fail-fast.

## Per-axis acceptance criteria

### `modules/`

- `axis: module`. `runtime.location: customer-hosted` is the only valid runtime for private modules (ADR 0004).
- `main.tf` exists. `terraform fmt -check` passes. `terraform validate` passes after `terraform init -backend=false`.
- `version` bumped on any `*.tf` change.
- `bundle.modulePath` is the path under the repo root.

### `scanners/`

- `axis: scanner`. Customer-hosted: `runtime: { location: customer-hosted, dispatch: lace-pull, oidcTrust: { issuer, audience } }` (Lace dispatches HTTPS to your endpoint with an OIDC-signed JWT) **or** `runtime: { location: customer-hosted, dispatch: customer-push, scopeRequired: <scope> }` (your endpoint POSTs results to Lace).
- `outputs` declares ≥ 1 of `findings | snapshots | time_series | inventory` with sub-kind schemas.

### `run-tasks/`

- `axis: run_task`. `runtime.location: customer-hosted`.
- `hooks.available` ⊆ {`pre_plan`, `post_plan`, `pre_apply`, `post_apply`, `post_destroy`}, ≥ 1 entry. `hooks.default` ⊆ `hooks.available`.
- `signing: { kind: hmac_sha256 }` for any task carrying secrets.
- `verdictMode` ∈ {`sync`, `async`}.
- `endpointSource: 'install'` (per-install URL) or `'manifest'` (single URL pinned in the manifest; `endpointUrl` required).

### `chaos-providers/`

- `axis: chaos_provider`. `runtime.location: customer-hosted` with `dispatch: customer-push` (your provider POSTs `ProviderRunResult` callbacks to Lace).
- `targetCatalog.targets` lists `(resourceType, uriPattern, actions, rollbackKinds)` tuples covering every action your provider supports.
- `callbackSigning: { kind: hmac_sha256 }` is required for customer-hosted providers.

## Review checklist

For every PR:

- [ ] Envelope fields present (apiVersion, axis, author, name, version, displayName, configSchema, runtime).
- [ ] `version` is a fresh `v`-prefixed semver, bumped vs. base branch.
- [ ] `(axis, author, name)` is unique across the repo.
- [ ] `runtime.location: customer-hosted` (private manifests cannot use `lace-managed`).
- [ ] Per-axis fields conform to the schema referenced above.
- [ ] `README.md` exists alongside `manifest.yaml` and explains usage.
- [ ] Modules: `terraform fmt -check` + `terraform init -backend=false && terraform validate` pass.
- [ ] No secrets, credentials, or production data in any manifest or README.

## Manifest immutability

Once a `(axis, author, name, version)` row lands in `registry_manifest`, its content is frozen. Re-publishing the same version with different content is rejected with HTTP 409. To ship a fix, bump `version`.

## Visibility

Manifests published from this repo carry `org_id = <your org>`. They are visible only to your org's catalog browse. Public manifests authored by Lace at `lace-cloud/registry` (carrying `org_id = NULL`) are also visible to your catalog browse, but you cannot publish to that namespace from here.
