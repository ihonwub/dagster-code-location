# dagster-code-location (Repo B)

A small, versioned Helm **wrapper chart** that deploys **one** Dagster code
location: a team's gRPC code-server pod plus a NetworkPolicy that locks it down
to the control plane.

It wraps Dagster's official
[`dagster-user-deployments`](https://artifacthub.io/packages/helm/dagster/dagster-user-deployments)
chart (a Helm dependency) and adds the per-location NetworkPolicy.

## Where it fits

```
Repo A · tenant ApplicationSet  (one Argo App per locations/<team>.yaml)
   │   installs THIS chart, once per team, with the team's values
   ▼
THIS CHART · dagster-code-location
   │   Chart.yaml depends on:
   ▼
OFFICIAL · dagster-user-deployments @ 1.13.10
   │   renders:
   ▼
Kubernetes (namespace dagster-<team>):
     Deployment  (the gRPC code server; Helm release dagster-loc-<team>)
     Service <team>   (DNS the control plane connects to:
                       <team>.dagster-<team>.svc.cluster.local)
   + from this chart: NetworkPolicy allow-instance-grpc
```

The official chart names the Service after the deployment entry's `name`, so a
file named `signals` yields Service `signals` and host
`signals.dagster-signals.svc` — which is exactly what Repo A's workspace registry
derives. The two halves agree by convention, not configuration.

## What a DS team provides (indirectly)

DS teams do **not** edit this chart. They edit one file in Repo A:
`apps/dagster-workspace/locations/<team>.yaml` — and that file is a **standard
upstream `dagster-user-deployments` deployment entry** (`name`, `port`,
`dagsterApiGrpcArgs`, `image`, `resources`, …). Repo A's tenant ApplicationSet
splats it whole into the values below.

## Values

| Key | Set by | Meaning |
|---|---|---|
| `dagster-user-deployments.deployments` | tenant appset | the team's single deployment, splatted from their location file (`name` = the file's `name`, image, gRPC args, port, …) |
| `dagster-user-deployments.global.postgresqlAuthWifEnabled` | tenant appset | `true` — skips Postgres-secret injection into the code-server pod (it doesn't need DB creds; run pods do, in the `dagster` ns) |
| `dagster-user-deployments.serviceAccount` | tenant appset | per-team ServiceAccount (the `eks.amazonaws.com/role-arn` annotation is vestigial on Pod Identity clusters; per-location identity moves to a Pod Identity association as a follow-up) |
| `networkPolicy.locationName` | tenant appset | team name, for the pod selector |
| `networkPolicy.port` | tenant appset | the gRPC port to allow |
| `networkPolicy.instanceNamespace` | default `dagster` | control-plane namespace allowed in |

> **Design note:** a Helm wrapper cannot template its subchart's values, so the
> friendly "team file → `deployments[]`" mapping is done in Repo A's tenant
> ApplicationSet, not here. This chart's job is to pin the official chart, ship
> safe defaults, and add the NetworkPolicy.

## How Repo A consumes it

Argo CD pulls this chart **straight from this git repo** — the tenant
ApplicationSet (`apps/dagster-code-locations/applicationset.yaml`) source points at
the repo + path `.` and `targetRevision: main`. There is **no OCI publish step**;
the `dagster-user-deployments` subchart is fetched at render time from the
repository pinned in `Chart.yaml`, so `charts/` and `Chart.lock` are not committed
(see `.gitignore`). Local sanity check:

```sh
helm dependency build .
helm template . --set networkPolicy.locationName=demo --set networkPolicy.port=4001
```

## Verify before you ship

1. **Subchart version** — `Chart.yaml` pins `dagster-user-deployments` to the
   version the control plane runs (currently `1.13.10`). Keep them aligned.
2. **Pod label key** — the NetworkPolicy selects `deployment: <name>` (the
   deployment entry's name); confirm the official chart's label key for your
   version.
