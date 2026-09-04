# akuity-mcp-demo

GitOps bootstrap for a Kargo promotion pipeline on the Akuity Platform.

## Layout

| Path | Managed by | Contents |
| --- | --- | --- |
| `bootstrap/argocd/` | direct-applied via the Akuity platform MCP endpoint | Argo CD `Application` manifests |
| `bootstrap/kargo/` | synced by the `guestbook-bootstrap` Application | Kargo `Project`, `Warehouse`, `Stage` chain |
| `dev/`, `staging/`, `prod/` | synced by the per-environment Applications | Deployment + Service per environment |
| `bootstrap/fanout/` | synced by the `fanout-bootstrap` Application | Fan-out demo Kargo `Project`, `Warehouse`, `Stage` graph |
| `fanout/<env>/` | synced by the per-environment Applications | Deployment per fan-out environment (`replicas: 0`) |

## Pipeline

`ghcr.io/akuity/guestbook` → Warehouse → `dev` → `staging` → `prod`

### Fan-out example pipeline (`fanout`)

A second Kargo project demonstrating **fan-out then fan-in with an AND gate** —
the `(cluster a + cluster b) -> (cluster c + cluster d) -> the rest` shape:

```
                    +-- staging1 --+
ghcr.io/.../guestbook -> dev1 --+              +--> prod1, prod2, prod3, prod4
                    +-- staging2 --+
```

- `staging1` and `staging2` both subscribe to `dev1` (they open in parallel as
  soon as `dev1` verifies).
- `prod1`-`prod4` subscribe to **both** stagings and set
  `sources.availabilityStrategy: All`, so freight is not available to any prod
  until it has verified in `staging1` **and** `staging2`.

```yaml
requestedFreight:
  - origin:
      kind: Warehouse
      name: fanout
    sources:
      availabilityStrategy: All   # AND. Default is OneOf, which is OR.
      stages:
        - staging1
        - staging2
```

`availabilityStrategy` is the whole ballgame: the default `OneOf` means *any one*
upstream stage verifying opens the gate, which is almost never what a
multi-cluster prod rollout wants.

Every environment runs `replicas: 0` — this project exists to show promotion
behavior, not to serve traffic. Promotions here are also **credential-free**
(`argocd-update` only, no git write); see the note in `bootstrap/fanout/stage-*.yaml`.


Promotions are executed by Kargo: it commits the new image tag to this repo and
syncs the environment's Argo CD Application. **Do not edit the `image:` field in
an environment directory by hand** — that is the pipeline's write path.

## Platform

- Argo CD instance `argocd-instance` — `bsdtcto6dsa10rph.cd.akuity.cloud`
- Kargo instance `kargo-instance` — `qassw5iwhizl8sq4.kargo.akuity.cloud`
- Workload cluster `mac1`, Kargo agent/shard `kargo-mac1`
