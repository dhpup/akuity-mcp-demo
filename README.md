# akuity-mcp-demo

GitOps bootstrap for a Kargo promotion pipeline on the Akuity Platform.

## Layout

| Path | Managed by | Contents |
| --- | --- | --- |
| `bootstrap/argocd/` | direct-applied via the Akuity platform MCP endpoint | Argo CD `Application` manifests |
| `bootstrap/kargo/` | synced by the `guestbook-bootstrap` Application | Kargo `Project`, `Warehouse`, `Stage` chain |
| `dev/`, `staging/`, `prod/` | synced by the per-environment Applications | Deployment + Service per environment |
| `bootstrap/teams/<team>/` | synced by each `<team>-bootstrap` Application | Per-team Kargo `Project`, `Warehouse`, `Stage` chain |
| `teams/<team>/<env>/` | synced by the per-environment Applications | Deployment + Service per team environment |

## Pipeline

`ghcr.io/akuity/guestbook` → Warehouse → `dev` → `staging` → `prod`

### Team pipelines

`team-red`, `team-blue`, and `team-yellow` each get an independent Kargo project
with its own Warehouse on the same image and a `dev` → `staging` → `prod` chain:

`ghcr.io/akuity/guestbook` → `<team>` Warehouse → `dev` → `staging` → `prod`

Each stage manages the Argo CD Application `<team>-<env>`, deploying into the
namespace of the same name on cluster `mac1`. The environments are scaffolded at
`v0.0.1`; an environment stays undeployed until its first promotion.

Promotions are executed by Kargo: it commits the new image tag to this repo and
syncs the environment's Argo CD Application. **Do not edit the `image:` field in
an environment directory by hand** — that is the pipeline's write path.

## Platform

- Argo CD instance `argocd-instance` — `bsdtcto6dsa10rph.cd.akuity.cloud`
- Kargo instance `kargo-instance` — `qassw5iwhizl8sq4.kargo.akuity.cloud`
- Workload cluster `mac1`, Kargo agent/shard `kargo-mac1`
