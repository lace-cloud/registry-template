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
agents/acme/cost-whisperer/{manifest.yaml, README.md}
```

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

The four axis subdirs ship empty (`.gitkeep`). Add `<author>/<name>/manifest.yaml` per the envelope.

## How it relates to the public registry

The public registry at `lace-cloud/registry` and your private registry are structurally identical. Both:

- Use the same `manifest.yaml` envelope.
- Run the same CI workflows.
- Call the same `lace registry register` CLI against the same `POST /api/v1/registry/index` endpoint.

The only differences:

| | **Public registry** (`lace-cloud/registry`) | **Your private registry** |
|---|---|---|
| API key scope | `REGISTRY_PUBLISH` | `REGISTRY_PUBLISH:org` |
| Resulting `org_id` | `NULL` (visible to all orgs) | your org (visible only to your org) |
| Reviewers | `@lace-cloud/platform-team` | whoever you put in CODEOWNERS |

The Lace cloud handles the `org_id` clamp server-side based on the bearer token's scope. Your private manifests cannot leak into the public catalog and vice versa.

## Branch strategy

`feature/* → develop → main`. Push to `main` triggers `publish.yml`. Customize protection rules to taste.

## ADR 0004: in-tree handlers and agents are Lace-eng only

Customer-authored manifests (public *or* private) must declare:

```yaml
runtime:
  location: customer-hosted
  dispatch: lace-pull | customer-push | customer-poll
  ...
```

`runtime: { location: lace-managed }` is reserved for `author: lace` manifests in the public registry — those manifests' handler/agent code ships in Lace's tree. Your private manifests will be rejected at publish time if they try to declare `lace-managed`.

## Troubleshooting

### `403: REGISTRY_PUBLISH_ORG requires an org-bound service token`

Your `LACE_REGISTRY_KEY` is missing the `REGISTRY_PUBLISH:org` scope or is not bound to an org. Re-issue from the portal.

### `409: manifest at this version already published with different content`

Manifests are immutable per `(axis, author, name, version)`. Bump the `version` field and reopen the PR.

### CI envelope check passes but publish fails with a zod error

The CI's envelope check is fail-fast; the API runs the full per-axis zod (in `apps/api/src/lib/registry/axes/` of the lace monorepo) at publish time. Read the API error message for the field path that failed.

## Get help

- File an issue in your private registry repo.
- Reach out to `support@lace.cloud` for envelope or scope questions.
