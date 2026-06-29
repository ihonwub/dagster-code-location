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
OFFICIAL · dagster-user-deployments @ 1.11.7
   │   renders:
   ▼
Kubernetes (namespace dagster-<team>):
     Deployment dagster-loc-<team>   (the gRPC code server)
     Service    dagster-loc-<team>   (DNS the control plane connects to)
   + from this chart: NetworkPolicy allow-instance-grpc
```

## What a DS team provides (indirectly)

DS teams do **not** edit this chart. They edit one file in Repo A:
`charts/dagster-workspace/locations/<team>.yaml` (name, module, port, image).
Repo A's tenant ApplicationSet turns that file into the values below.

## Values

| Key | Set by | Meaning |
|---|---|---|
| `dagster-user-deployments.deployments` | tenant appset | the team's single deployment (`name: dagster-loc-<name>`, image, module, port) |
| `dagster-user-deployments.serviceAccount` | tenant appset | per-team ServiceAccount + IRSA role |
| `networkPolicy.locationName` | tenant appset | team name, for the pod selector |
| `networkPolicy.port` | tenant appset | the gRPC port to allow |
| `networkPolicy.instanceNamespace` | default `dagster` | control-plane namespace allowed in |

> **Design note:** a Helm wrapper cannot template its subchart's values, so the
> friendly "team file → `deployments[]`" mapping is done in Repo A's tenant
> ApplicationSet, not here. This chart's job is to pin the official chart, ship
> safe defaults, and add the NetworkPolicy.

## Publish (so Repo A can pull it)

```sh
helm dependency update .
helm lint . && helm template . --set networkPolicy.locationName=demo   # sanity check
helm package .
helm push dagster-code-location-0.1.0.tgz oci://<your-registry>
```

Then pin that version in Repo A's `manifests/code-locations-appset.yaml`
(`targetRevision`).

## Verify before you ship

1. **Subchart globals** — a standalone code server still needs Postgres creds and
   `DAGSTER_HOME` to launch runs. Wire `dagster-user-deployments.global.*` to
   match the control plane.
2. **Pod label key** — the NetworkPolicy selects `deployment: dagster-loc-<name>`;
   confirm the official chart's label key for your version.
