# PostgreSQL

## Persistence

Control plane:

```console
kubectl apply -f data-postgres-pv.yaml
```

Worker nodes:

```console
sudo mkdir -p /mnt/postgresql
sudo chown 1001:1001 /mnt/postgresql
```

## Installation

(optional) Inspect & modify the default values :

```console
helm show values oci://registry-1.docker.io/bitnamicharts/postgresql > custom-values.yaml
```

```console
kubectl apply -f psql-initdb-scripts-secret.yaml
```

```console
helm upgrade --install postgresql oci://registry-1.docker.io/bitnamicharts/postgresql -f custom-values.yaml
```

### Post-installation message

```
PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.3.0-debian-12-r23 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

## Restore database from backup

Example for nextcloud. Commands are run on the control plane:

```console
kubectl cp nextcloud_db_backup.sql postgresql-0:/bitnami/postgresql/
kubectl exec postgresql-0 -- psql -U nextcloud -f /bitnami/postgresql/nextcloud_db_backup.sql
```