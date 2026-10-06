# akuity-mcp-demo

GitOps bootstrap for a Kargo promotion pipeline on the Akuity Platform.

## Layout

| Path | Managed by | Contents |
| --- | --- | --- |
| `bootstrap/argocd/` | direct-applied via the Akuity platform MCP endpoint | Argo CD `Application` manifests |
| `bootstrap/kargo/` | synced by the `guestbook-bootstrap` Application | Kargo `Project`, `Warehouse`, `Stage` chain |
| `dev/`, `staging/`, `prod/` | synced by the per-environment Applications | Deployment + Service per environment |

## Pipeline

`ghcr.io/akuity/guestbook` → Warehouse → `dev` → `staging` → `prod`

Promotions are executed by Kargo: it commits the new image tag to this repo and
syncs the environment's Argo CD Application. **Do not edit the `image:` field in
an environment directory by hand** — that is the pipeline's write path.

## Platform

- Argo CD instance `argocd-instance` — `bsdtcto6dsa10rph.cd.akuity.cloud`
- Kargo instance `kargo-instance` — `qassw5iwhizl8sq4.kargo.akuity.cloud`
- Workload cluster `mac1`, Kargo agent/shard `kargo-mac1`

## Team Red (CHANGE-57)

A second, independent pipeline for the same image, under `team-red/`:

| Path | Managed by | Contents |
| --- | --- | --- |
| `team-red/bootstrap/argocd/` | direct-applied via the Akuity platform MCP endpoint | `team-red-bootstrap` + `team-red-{dev,staging,prod}` Applications |
| `team-red/bootstrap/kargo/` | synced by `team-red-bootstrap` | Kargo Project `team-red`, Warehouse, Stage chain |
| `team-red/{dev,staging,prod}/` | synced by `team-red-{env}` | Deployment + Service, namespaces `team-red-{env}` on `mac1` |

`ghcr.io/akuity/guestbook` → Warehouse → `dev` → `staging` → `prod` (Kargo project `team-red`).
The same rule applies: the `image:` fields are written only by Kargo promotions.
