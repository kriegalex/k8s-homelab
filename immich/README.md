# Immich

## Persistence

We manage everything manually to avoid PVCs binding to the wrong PV. Also, a homelab K8S is not really a cloud native kubernetes cluster that dynamically asks storage from a cloud service using `storageClass`.

```
kubectl create -f immich-psql-pv.yaml -f immich-psql-pvc.yaml
kubectl create -f immich-redis-pv.yaml -f immich-redis-pvc.yaml
kubectl create -f immich-ml-cache-pv.yaml -f immich-ml-cache-pvc.yaml
```

### Setup permissions

```
sudo mkdir -p /mnt/immich/psql
sudo chown 1001:1001 /mnt/immich/psql
sudo mkdir -p /mnt/immich/redis
sudo chown 1001:1001 /mnt/immich/redis
sudo mkdir -p /mnt/immich/ml-cache
sudo chown 1001:1001 /mnt/immich/ml-cache
```

## Installation

```
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
```

```console
helm show values k8s-charts/immich > custom-values.yaml
```

> Adapt any needed values. Check [my repository](https://github.com/kriegalex/k8s-charts/blob/main/charts/immich/values.yaml) or [the original repository](https://github.com/immich-app/immich-charts) for more information.

```
helm upgrade --install immich k8s-charts/immich -f custom-values.yaml
```

## Uninstall

```
helm uninstall immich
kubectl delete -f immich-ml-data-pvc.yaml -f immich-ml-data-pv.yaml \
  -f immich-psql-data-pv.yaml
kubectl delete pvc data-immich-postgresql-0
```

## Backup

If using the immich postgresql subchart:
```console
kubectl exec immich-postgresql-0 -- pg_dumpall --clean --if-exists -U immich -d immich -f /tmp/immich_db_backup.sql
kubectl cp immich-postgresql-0:/tmp/immich_db_backup.sql immich_db_backup.sql
```

## Restore

If using the immich postgresql subchart:
```console
kubectl cp immich_db_backup.sql immich-postgresql-0:/bitnami/postgresql/
kubectl exec -it immich-postgresql-0 -- psql -U immich -d immich -f /bitnami/postgresql/immich_db_backup.sql
```

### "error: invalid command \N"

If you try to use a regular centralized PostgreSQL database with immich, it won't work. A popular example of that would be the bitnami image of PostgreSQL. 

Immich however uses the `tensorchord/pgvecto-rs` image for PSQL. It has some special extensions that make it incompatible with regular versions of PostgreSQL servers.