# registry-template

Starter template for an org-private Lace registry monorepo. Clone this template, point it at your org's Lace cloud, and your platform team can author internal Terraform modules, handlers, scanners, and agents under PR review — same shape as the public `lace-cloud/registry` monorepo.

## Quickstart

### 1. Create your registry repo

```bash
gh repo create acme-corp/lace-registry --template lace-cloud/registry-template --private
git clone git@github.com:acme-corp/lace-registry.git
cd lace-registry
```

### 2. Issue an org-scoped service token

In the Lace portal: **Org Settings → Service Tokens → Create**. Choose:

- **Scope:** `registry:publish:org` (publishes manifests visible only to your org).
- **Name:** `registry-bot` (or whatever you like).

Copy the token. It is shown once.

### 3. Set the secret in GitHub

```bash
gh secret set LACE_REGISTRY_KEY --body '<paste-the-token>'
```

### 4. Set CODEOWNERS

Edit `.github/CODEOWNERS` so PRs are reviewed by the right team:

```
*                            @acme-corp/platform-team
modules/                     @acme-corp/platform-team
handlers/acme/               @acme-corp/platform-team
scanners/acme/               @acme-corp/platform-team
agents/acme/                 @acme-corp/platform-team
```

### 5. Author manifests

Drop manifests under the appropriate axis subdir. Each manifest follows the standard envelope (see [CONTRIBUTING.md](./CONTRIBUTING.md)):

```
modules/acme/internal-vpc/{manifest.yaml, main.tf, ...}
handlers/acme/policy-gate/{manifest.yaml, README.md}
scanners/acme/sox-compliance/{manifest.yaml, README.md}
agents/acme/cost-advisor/{manifest.yaml, README.md}
```

The `<author>` segment (`acme` above) must be your **org slug** exactly — an org-scoped token cannot publish a foreign namespace. See [CONTRIBUTING.md](./CONTRIBUTING.md#folder-shape).

Open a PR; CI validates the envelope. On merge to `main`, `publish.yml` calls `lace registry register --axis <axis> --manifest <path>` against `https://api.lace.cloud/api/v1/registry/index`. The endpoint clamps `org_id` to your org based on the service token's scope; manifests are visible only to your org's catalog browse.

## What's in this template

```
.
├── README.md                       ← this file
├── CONTRIBUTING.md                 ← envelope + per-axis acceptance criteria
├── .github/
│   ├── CODEOWNERS                  ← edit to point at your review team
│   └── workflows/
│       ├── ci.yml                  ← PR gate
│       └── publish.yml             ← push to main → register
├── modules/                        ← Terraform modules (with .tf files)
├── handlers/                       ← Handler integrations (Run-lifecycle gates, routers, notifiers)
├── scanners/                       ← Observatory scanners
└── agents/                         ← Advisory-by-emission agents
```

Each axis subdir ships with a `README.md` placeholder that documents what goes there and keeps the directory in git. Add `<author>/<name>/manifest.yaml` per the envelope.

These four axes are the concept model's four artifacts, and the set is closed — the API validates `axis` against a fixed enum:

| Axis | Role on the timeline |
|---|---|
| `module` | Terraform the runner applies — the **desired state** an artifact contributes. |
| `handler` | Subscribes to Run-lifecycle hooks and returns a **Verdict**. Only `pre_apply` gates; every other hook is observing-only. |
| `scanner` | Produces **Findings / snapshots / time-series / inventory** on a cadence, into the Observatory. |
| `agent` | Reads a producer and emits an **Advisory**. Never returns a verdict, never gates. |

## How it relates to the public registry

The public registry at `lace-cloud/registry` and your private registry are structurally identical. Both:

- Use the same `manifest.yaml` envelope.
- Run the same CI workflows.
- Call the same `lace registry register` CLI against the same `POST /api/v1/registry/index` endpoint.

The only differences:

| | **Public registry** (`lace-cloud/registry`) | **Your private registry** |
|---|---|---|
| API key scope | `registry:publish` | `registry:publish:org` |
| Resulting `org_id` | `NULL` (visible to all orgs) | your org (visible only to your org) |
| Reviewers | `@lace-cloud/platform-team` | whoever you put in CODEOWNERS |

The Lace cloud handles the `org_id` clamp server-side based on the bearer token's scope. Your private manifests cannot leak into the public catalog and vice versa.

## Branch strategy

`feature/* → develop → main`. Push to `main` triggers `publish.yml`. Customize protection rules to taste.

## In-tree dispatch is Lace-authored only

`runtime.location` has exactly two values — `in-tree` and `external`. Handler, scanner, and agent manifests you publish from this repo must declare:

```yaml
runtime:
  location: external
  dispatch: lace-pull | customer-push | customer-poll
  ...
```

`runtime: { location: in-tree }` means the artifact's code ships inside Lace's own tree, resolved through a compile-time `IN_TREE_*` map. It requires **both** `author: lace` **and** public visibility (`org_id IS NULL`), so a manifest published from this repo is rejected at publish time (HTTP 422) if it declares `in-tree`.

**Modules are exempt.** The gate guards in-tree *handler dispatch*, a concept the module axis does not carry — for a module, `in-tree` just means "Terraform applied by the Lace runner", which is the normal case for everyone's modules, including yours.

> `location: customer-hosted` and `location: lace-managed` are the retired spelling and no longer validate. Use `external` and `in-tree`.

## Troubleshooting

### `403: REGISTRY_PUBLISH_ORG requires an org-bound service token`

Your `LACE_REGISTRY_KEY` is missing the `registry:publish:org` scope or is not bound to an org. Re-issue from the portal.

### `403: Cannot publish under author "X" from org "Y" — author must match the org slug`

An org-scoped token may only publish under its own namespace. Set the manifest's `author` to your org slug exactly — team-internal namespaces (`acme-platform`) are rejected. Carve out team scope in `name` instead.

### `409: manifest at this version already published with different content`

Manifests are immutable per `(axis, author, name, version)`. Bump the `version` field and reopen the PR.

### CI envelope check passes but publish fails with a validation error

The CI check in this repo is deliberately shallow and fail-fast — it verifies the envelope fields, the axis/directory agreement, and a few values known to be retired. The authoritative per-axis validation runs server-side at publish time and is stricter in two ways worth knowing:

- **Schemas are strict.** An unrecognized key is an error, not a warning. A plausible-looking field that isn't in the schema fails the publish.
- **The error names the field path.** Read the `path` in the API response first — it points at the exact key, which is usually faster than re-reading the manifest.

For the current accepted shape of any axis, the published manifests in the public registry at [`lace-cloud/registry`](https://github.com/lace-cloud/registry) are working examples of every axis. `lace registry register` also prints the full validation error, so running it locally against your manifest is the fastest way to iterate.

### `422: binding an external agent whose read scope exceeds its bind scope requires explicit egress acknowledgement`

Not a publish error — an *install* error your users will hit. The agent's manifest declares a `reads.scope` broader than the scope the admin is binding at, so the binding needs an explicit egress acknowledgement. Either the admin re-binds at a broader scope or acknowledges, or you narrow `reads.scope` in the manifest. See [CONTRIBUTING.md](./CONTRIBUTING.md#egress-consent-readsscope-decides-how-hard-your-artifact-is-to-install).

## Get help

- File an issue in your private registry repo.
- Reach out to `support@lace.cloud` for envelope or scope questions.
