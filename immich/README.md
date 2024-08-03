# Immich

## PSQL persistence

```
kubectl create -f immich-psql-data-pv.yaml
```

### Setup permissions

```
sudo mkdir -p /mnt/immich/psql-data
sudo chown 1001:root /mnt/immich/psql-data
```

## Immich persistence

```
kubectl create -f immich-ml-data-pv.yaml -f immich-ml-data-pvc.yaml
```

## Redis

### External

If using an external redis server, retrieve the password to put it in the custom-values.yaml. The template may be improved in the future to directly use the actual Secret.

```
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)
```

### Internal

If using the internal subchart for redis, enable it in the custom-values.yaml and setup an appropriate PersistentVolume for it.

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

## Manual backup

(optional) if using the postgresql subchart:
```console
kubectl exec immich-postgresql-0 -- pg_dumpall --clean --if-exists -U immich -d immich -f /tmp/immich_db_backup.sql
kubectl cp immich-postgresql-0:/tmp/immich_db_backup.sql immich_db_backup.sql
```

## Restore

### External database

```console
kubectl cp immich_db_backup.sql postgresql-0:/bitnami/postgresql/
kubectl exec -it postgresql-0 -- psql -U immich -d immich -f /bitnami/postgresql/immich_db_backup.sql
```

### "error: invalid command \N"

If you try to use a regular centralized PostgreSQL database with immich, it won't work. A popular example of that would be the bitnami image of PostgreSQL. 

Immich however uses the `tensorchord/pgvecto-rs` image for PSQL. It has some special extensions that make it incompatible with regular versions of PostgreSQL servers.