# akuity-mcp-demo

GitOps bootstrap for a Kargo promotion pipeline on the Akuity Platform.

## Layout

| Path | Managed by | Contents |
| --- | --- | --- |
| `bootstrap/argocd/` | direct-applied via the Akuity platform MCP endpoint | Argo CD `Application` manifests |
| `bootstrap/kargo/` | synced by the `guestbook-bootstrap` Application | Kargo `Project`, `Warehouse`, `Stage` chain |
| `dev/`, `staging/`, `prod/` | synced by the per-environment Applications | Deployment + Service per environment |
| `bootstrap/fanout/` | synced by the `fanout-bootstrap` Application | Fan-out demo Kargo `Project`, `Warehouse`, `Stage` graph (incl. control flow stages) |
| `fanout/<env>/` | synced by the per-environment Applications | Deployment per fan-out environment (`replicas: 0`) |

## Pipeline

`ghcr.io/akuity/guestbook` → Warehouse → `dev` → `staging` → `prod`

### Fan-out example pipeline (`fanout`)

A second Kargo project demonstrating **fan-out then fan-in with an AND gate** —
the `(cluster a + cluster b) -> (cluster c + cluster d) -> the rest` shape:

```
                      +-- staging1 --+     +-- control1 --> prod1 prod2 prod3 prod4   (yellow)
dev1 --+                             +--> -+
                      +-- staging2 --+     +-- control2 --> prod5 prod6 prod7 prod8   (orange)
```

- `staging1` and `staging2` both subscribe to `dev1` (they open in parallel as
  soon as `dev1` verifies).
- `control1` and `control2` are **control flow stages** — Stages with no
  `promotionTemplate` steps. They deploy nothing and have no Argo CD
  Application; each is just the fan-in gate for one prod wave. Both subscribe
  to **both** stagings with `sources.availabilityStrategy: All`, so freight
  reaches neither until it has verified in `staging1` **and** `staging2`.
- Each prod then subscribes to a single control flow stage: `prod1`-`prod4` to
  `control1`, `prod5`-`prod8` to `control2`. The AND gate has moved up to the
  control flow stage, so the prods no longer need `availabilityStrategy`.
- The prods are colored in the Kargo UI via the `kargo.akuity.io/color`
  annotation on the `Stage`: `prod1`-`prod4` are `yellow`, `prod5`-`prod8` are
  `orange`, so the two waves read apart at a glance.

The control flow stages change *grouping*, not *concurrency*: the four prods
under one control flow stage still open and promote in parallel, and the two
control flow stages themselves open in parallel off the same stagings. What
they buy is one promotion handle per wave — promote `control1` and the yellows
go; hold `control2` and the oranges wait.

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

That block now lives on `control1`/`control2`. `availabilityStrategy` is the
whole ballgame: the default `OneOf` means *any one* upstream stage verifying
opens the gate, which is almost never what a multi-cluster prod rollout wants.

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
