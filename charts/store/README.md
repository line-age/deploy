# store chart

Runs [Lineage store](https://github.com/line-age/store): the API, its database migrations,
and optionally the QGIS project and county lines, from the public images
`ghcr.io/line-age/store` and `ghcr.io/line-age/store-counties`.

It doesn't include a database. Store runs against plain PostGIS (controlplane L-0006),
which you provide, with PostGIS available to the user the chart connects as (the first
migration runs `CREATE EXTENSION postgis`).

## Install

Make a Secret with the connection URL, then install:

```sh
kubectl create secret generic store-db \
  --from-literal=DATABASE_URL='postgres://USER:PASSWORD@HOST:5432/lineage?sslmode=require'

helm install store oci://ghcr.io/line-age/charts/store \
  --set database.existingSecret=store-db \
  --set image.tag=sha-<store commit> \
  --set 'github.teams={line-age/team-store}'
```

Before every install and upgrade a Job runs the migrations (`dbmate --wait up`); the API
starts once they're done. Admin commands run in the API pod:

```sh
kubectl exec deploy/store -- lineage-store token create backport --pipeline backport@<sha> --assert-for-others
kubectl exec deploy/store -- lineage-store agent add --github <login> --name "<name>" --email <email>
```

## Values

| Value | Default | What it is |
| --- | --- | --- |
| `database.existingSecret` | (required) | Secret holding the database URL |
| `database.urlKey` | `DATABASE_URL` | its key: `postgres://user:pw@host:5432/db?sslmode=...` |
| `image.repository` | `ghcr.io/line-age/store` | |
| `image.tag` | the chart's `appVersion` (`main`) | pin `sha-<commit>` or a release |
| `github.teams` | `[]` | `org/team-slug` teams whose members may sign in; empty means pipeline tokens only |
| `github.api` | `https://api.github.com` | |
| `migrations.enabled` | `true` | the migrate Job |
| `qgisProject.enabled` | `false` | write the ready-made QGIS project into the database after install and upgrade |
| `qgisProject.host`, `.port`, `.dbname` | `""`, `5432`, `lineage` | the database as QGIS users reach it (required when enabled) |
| `counties.enabled` | `false` | load historical county lines (~50 MB download, once) |
| `counties.pgUrlKey` | `PG_URL` | key in the same Secret, GDAL form: `PG:host=... port=5432 dbname=... user=... password=...` |
| `service.type`, `.port` | `ClusterIP`, `80` | |
| `ingress.*` | off | standard Ingress: `className`, `annotations`, `hosts`, `tls` |
| `replicaCount`, `resources`, `nodeSelector`, `tolerations`, `affinity` | | as usual |

Pods run as uid 10001 with a read-only root filesystem and no capabilities
(`podSecurityContext`, `securityContext`).

## Testing it

`ci/postgis.yaml` is a throwaway PostGIS and Secret for a kind cluster; CI installs the
chart against it with `ci/values.yaml` and `ci/everything-on.yaml`. Not for real use.
