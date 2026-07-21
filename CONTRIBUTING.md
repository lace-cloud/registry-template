# Contributing

This repo holds your org's private Lace registry manifests. PRs are reviewed by the team(s) named in `.github/CODEOWNERS`.

## Folder shape

```
<axis>/<author>/<name>/manifest.yaml
<axis>/<author>/<name>/README.md
```

`<axis>` ∈ {`modules`, `handlers`, `scanners`, `agents`}. The `(axis, author, name)` tuple must be unique within this repo.

`<author>` **must equal your org's slug exactly** — publishing with an org-scoped service token enforces `author === org.slug`, so a team-internal namespace like `acme-platform` is rejected with HTTP 403 even though it is valid kebab-case. If your org slug is `acme`, every manifest here is authored `acme`. Both `author` and `name` are lowercase kebab-case, 1–64 chars, matching `^[a-z0-9][a-z0-9-]{0,63}$`. Use `<name>` to carve out team scope (e.g. `name: platform-internal-vpc`).

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
  location: external               # 'in-tree' is Lace-authored + public only
  dispatch: lace-pull | customer-push | customer-poll
  ...
authors: ["Acme Platform <platform@acme.example>"]
# Axis-specific fields layered on top — see per-axis acceptance criteria below.
```

The deep validation runs server-side. The CI envelope check is fail-fast.

### `runtime` is a closed discriminated union

`runtime.location` has exactly two values, and `dispatch` exists only on `external`:

| `location` | `dispatch` | Additional fields | Who may publish |
|---|---|---|---|
| `in-tree` | *(none)* | `handlerId` | `author: lace` **and** public only — **modules exempt** |
| `external` | `lace-pull` | `webhookUrl?`, `audienceFormat?`, `oidcTrust?`, `configRequires` | anyone |
| `external` | `customer-push` | `scopeRequired` | anyone |
| `external` | `customer-poll` | `queueScope`, `pollScopeRequired`, `pollWaitSeconds` | anyone |

Everything you publish from this repo is org-private, so **every non-module manifest here declares `location: external`**. `in-tree` is rejected at publish time for any author other than `lace`, and rejected outright for org-private manifests regardless of author.

> The older `lace-managed` / `customer-hosted` spelling is gone. If you copied a manifest from an older template, rewrite `customer-hosted` → `external`.

## Per-axis acceptance criteria

### `modules/`

- `axis: module`. Modules may declare `runtime.location: in-tree` (Terraform applied by the Lace runner — the universal case) or `external`. **Modules are the one axis exempt from the `author: lace` restriction on `in-tree`:** that restriction guards in-tree *handler dispatch*, which the module axis does not carry, so your org may publish `in-tree` modules.
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

A Handler is the universal verdict producer: it subscribes to Run-lifecycle hooks and returns a Verdict.

- `axis: handler`. `runtime.location: external` (org-private manifests cannot be `in-tree`).
- `hooks.available` ⊆ {`pre_plan`, `post_plan`, `pre_apply`, `post_apply`, `post_destroy`}, ≥ 1 entry. `hooks.default` ⊆ `hooks.available`.
  - Only `pre_apply` **gates** a Run. The other four are observed-only and never back-pressure — the portal badges them "observing-only". Do not ship a handler that expects `post_plan` to block.
- `endpointSource` ∈ {`manifest`, `install`}. `in_tree` is reserved for `author: lace` in the public registry and additionally requires `runtime.location: in-tree`, a `runtime.handlerId`, and `signing.kind: none`.
  - `manifest`: declare `endpointUrl` (one URL every install dispatches to). Required for this mode.
  - `install`: omit `endpointUrl` — each org supplies its own at install time. Declaring both is rejected.
- `signing`: `{ kind: hmac_sha256, headerName?: <header> }` for any handler carrying secrets, or `{ kind: none }`. `headerName` defaults to `X-Lace-Handler-Signature`.
- `verdictMode` ∈ {`sync`, `async`}. `sync` returns the verdict in the dispatch response body; `async` returns 202 and later POSTs to `/api/v1/handlers/:invocationId/callback`.
- `timeoutSeconds` (optional, 1–600, default 30) sets the per-handler default timeout. Bindings may override it.

### `scanners/`

- `axis: scanner`. `runtime.location: external`, with one of:
  - `dispatch: lace-pull` + `oidcTrust: { issuer, audience }` — Lace dispatches HTTPS to your endpoint with an OIDC-signed JWT. A declared `webhookUrl` must be `https://`.
  - `dispatch: customer-push` + `scopeRequired: <scope>` — your endpoint POSTs results to Lace.
  - `dispatch: customer-poll` + `queueScope` / `pollScopeRequired` — your worker long-polls a Lace queue.
  - A public/vendor lace-pull scanner must pin `webhookUrl` in the manifest. Org-private scanners like yours run on your own infrastructure, so the endpoint is supplied per install instead.
- `outputs` declares ≥ 1 of `findings | snapshots | time_series | inventory` with sub-kind schemas.
- `defaultIntervalSeconds` (optional, positive integer) sets the default scan cadence in seconds. When an install omits it, the scheduler falls back to the 300s floor — set it so an admin who installs without overriding gets a fire-able install instead of a silent never-fires row. It is clamped to `[300, 86400]` (5 min – 24 h) at schedule time. An install may override per-install via `configJson.intervalSeconds` (same clamp applies).

### `agents/`

An Agent is the model's fourth artifact: **advisory-by-emission**. It reads a timeline producer and writes an Advisory. It never returns a Verdict and never gates a Run — so agent manifests carry no `endpointSource`, `signing`, `verdictMode`, or `timeoutSeconds`, and agent bindings have no `mode`.

- `axis: agent`. `runtime.location: external` (org-private manifests cannot be `in-tree`).
#### `trigger` and `emits` are one decision, not two

There are exactly four Manifestations, they are not customer-extensible, and **each one's triggering Interaction is platform-fixed**. So choosing where your Advisory renders also determines what your agent may trigger on. Pick the row, then fill in both columns from it:

| `emits.manifestation` | The Interaction it renders at | Matching `trigger` |
|---|---|---|
| `preview` | An Actor composing a plan (pre-Run-submission) | `producer: run`, hooks `[post_plan]` |
| `draft` | An Actor opening a fresh Handler form | `producer: handler` |
| `triage` | A new Finding lands | `producer: scanner` |
| `trace` | An incident is declared | `producer: agent` |

⚠️ **The schema does not currently enforce this pairing** — it validates `trigger` and `emits` independently, so a manifest declaring `producer: run` + `manifestation: trace` will publish successfully and then render an Advisory about a plan onto the incident-investigation surface, where it is noise. Treat the table as the contract regardless. The four moments are fixed because each answers a question specific to that moment; an Advisory that arrives at the wrong one has no audience.

Mechanics, once you've picked the row:

- `trigger.producer` ∈ {`run`, `handler`, `scanner`, `agent`}.
- For `producer: run`, `trigger.hooks.available` ⊆ {`pre_plan`, `post_plan`, `pre_apply`, `post_apply`, `post_destroy`} and must be non-empty. `trigger.hooks.default` ⊆ `trigger.hooks.available`.
- For any other producer, `trigger.hooks.available` is empty — lifecycle hooks exist only on the `run` producer; the agent fires on the Event itself, filtered by payload.
- `emits.manifestation` takes exactly one of the four values.

  > **Breaking change.** The 2-persona `emits.surface` (`whisperer` | `triage`) is **rejected at publish** — the schema is strict. It survives only on the read/callback path so live agents mid-migration aren't 400'd, mapped `whisperer → preview`, `triage → triage`. New manifests must use `emits.manifestation`.

**A Manifestation holds many Advisories.** Binding your agent to a surface does not claim it — several agents may render at the same Manifestation and scope. The surface presents them as a ranked queue, each attributed to its emitting install, deduplicated where two Advisories declare the same normalized subject, and confidence-gated. That curation is display-only: every Advisory stays on the timeline in full, even when it doesn't make the view. Write your Advisory to stand on its own rather than assuming it is the only one shown.

- External agents MUST declare a `reads` descriptor for egress governance — an agent with no `reads` would silently bypass consent at bind time:
  ```yaml
  reads:
    scope: org | team | stack     # broadest context scope read
    kinds: [phase, verdict]       # ⊆ {phase, verdict, finding, advisory}, ≥ 1 entry
  ```
  `reads.kinds` is a closed vocabulary and must be non-empty — an empty ceiling reads nothing, so it is rejected rather than shipped as an unreachable manifest. Values like `run_history` or `plan_diff` are **not** in the vocabulary and will fail publish.
- `runtime.dispatch` must be `lace-pull` or `customer-poll`. `customer-push` is invalid for agents — that is the handler async verdict-return path, and agents do not return verdicts.
- For `customer-poll`, `runtime.pollScopeRequired` must be exactly `agents:poll` (the first-class mintable scope). Any other string is unmintable, so the poller could never obtain a usable token.

## Egress consent: `reads.scope` decides how hard your artifact is to install

**Read this before picking `reads.scope`.** An artifact with `runtime.location: external` runs outside Lace, so dispatching to it sends org-internal timeline context off-platform. The platform gates that with a per-binding consent check — and the scope you declare in the manifest determines whether the admins installing your artifact sail through or hit a wall.

Everything published from this repo is org-private, which the platform derives as `ownership: self`. For `self`, the rule is:

> **Explicit acknowledgement is required only when `reads.scope` is *broader* than the scope the admin binds at.**

Scopes rank `stack` (narrowest) < `team` < `org` (broadest). Reading *narrower* than the bind scope is always fine.

| Declared `reads.scope` | Bound at `stack` | Bound at `team` | Bound at `org` |
|---|---|---|---|
| `stack` | implicit | implicit | implicit |
| `team` | **explicit ack** | implicit | implicit |
| `org` | **explicit ack** | **explicit ack** | implicit |

When an ack is required and the caller doesn't pass one, binding fails with HTTP 422 `egress_consent_required`. The consent is recorded on the binding either way, and at dispatch the agent's read token is minted bounded to exactly the consented scope — so declaring a broad scope does not quietly grant a broad read; it just moves the friction to install time.

**Practical guidance:** declare the *narrowest* `reads.scope` your agent actually needs. Declaring `org` "to be safe" is the opposite of safe — it forces an explicit acknowledgement on every binding that isn't already org-wide, and it is the single most common reason an otherwise-correct agent is painful to adopt.

> Two related notes. **Vendor-published (public) external artifacts always require explicit ack**, regardless of scope — that path doesn't apply to this repo, but it's why you'll see the prompt on public catalog artifacts. And **modules are exempt entirely**: a module binds in-code with no endpoint, so there is nothing to egress to.
>
> Today this gate is enforced for external **agents**. Handlers and scanners are covered by the same governance model and inherit it as their external-observe dispatch path lands, so design your read surface as if it already applies.

## Review checklist

For every PR:

- [ ] Envelope fields present (apiVersion, axis, author, name, version, displayName, configSchema, runtime).
- [ ] `version` is a fresh `v`-prefixed semver, bumped vs. base branch.
- [ ] `(axis, author, name)` is unique across the repo.
- [ ] `runtime.location: external` for handlers, scanners, and agents (org-private manifests cannot be `in-tree`). Modules are exempt and may be `in-tree`.
- [ ] No `customer-hosted` / `lace-managed` / `emits.surface` left over from an older template.
- [ ] Per-axis fields conform to the schema referenced above.
- [ ] `README.md` exists alongside `manifest.yaml` and explains usage.
- [ ] Modules: `terraform fmt -check` + `terraform init -backend=false && terraform validate` pass.
- [ ] External agents: `reads.scope` is the narrowest scope that actually works — a broader scope forces an explicit egress acknowledgement on every narrower binding.
- [ ] Agents: `trigger` matches the Interaction that `emits.manifestation` renders at (see the pairing table). The schema does not check this.
- [ ] No secrets, credentials, or production data in any manifest or README.

## Manifest immutability

Once a `(axis, author, name, version)` row lands in `registry_manifest`, its content is frozen. Re-publishing the same version with different content is rejected with HTTP 409. To ship a fix, bump `version`.

## Visibility

Manifests published from this repo carry `org_id = <your org>`. They are visible only to your org's catalog browse. Public manifests authored by Lace at `lace-cloud/registry` (carrying `org_id = NULL`) are also visible to your catalog browse, but you cannot publish to that namespace from here.
