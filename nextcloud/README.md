# Nextcloud Helm Chart

Target environment:

- Homelab small server
- <= 1TB stored data

## Setup secrets

```console
kubectl create secret generic nextcloud-credentials \
  --from-literal=nextcloud-username='your_username' \
  --from-literal=nextcloud-password='your_password' \
  --namespace=default
```

```console
kubectl create secret generic nextcloud-psql-credentials \
  --from-literal=db-username='nextcloud' \
  --from-literal=db-password='your_password' \
  --from-literal=admin-password='your_admin_pwd' \
  --namespace=default
```

## Setup persistence

On the control plane:
```console
kubectl apply -f nextcloud-config-pv.yaml -f nextcloud-config-pvc.yaml
kubectl apply -f nextcloud-psql-pv.yaml -f nextcloud-psql-pvc.yaml
kubectl apply -f nextcloud-redis-pv.yaml -f nextcloud-redis-pvc.yaml
```

### Setup permissions (worker nodes)

```
sudo mkdir -p /mnt/nextcloud/psql
sudo chown 1001:1001 /mnt/nextcloud/psql
sudo mkdir -p /mnt/nextcloud/redis
sudo chown 1001:1001 /mnt/nextcloud/redis
sudo mkdir -p /mnt/nextcloud/config
sudo chown www-data:www-data /mnt/nextcloud/config
```

### NFS

If you are creating a PV that uses NFS, it can be helpful to setup the share with the option `no_root_squash`, as Nextcloud will try to chown the data during initialization. If not done, it doesn't seem to break the installation.

## Install Nextcloud

```console
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm install nextcloud k8s-charts/nextcloud -f custom-values.yaml
```

## (optional) Backup and restore existing docker 

### Backup

#### Postgres docker

PostgreSQL:
```console
docker exec -it nextcloud occ maintenance:mode --on
docker exec postgresql15 mkdir /backup
# backup inside the docker
docker exec postgresql15 pg_dump -U nextcloud -d nextcloud -f /backup/nextcloud_db_backup.sql
# extract the backup to the host
docker cp postgresql15:/backup/nextcloud_db_backup.sql /BACKUP_PATH/nextcloud-psql/
```

#### linuxserver/nextcloud docker

Nextcloud config:
```console
docker cp nextcloud:/config /BACKUP_PATH/nextcloud-config
```

Nextcloud data:
```console
docker cp nextcloud:/data /BACKUP_PATH/nextcloud-data
```

#### K8S

```console
# occ must be launched as www-data
kubectl exec -it nextcloud-POD -c nextcloud -- su -s /bin/bash www-data -c "/var/www/html/occ maintenance:mode --on"
kubectl exec -it nextcloud-postgresql-0 -- pg_dump --clean --if-exists -U nextcloud -d nextcloud -f /tmp/nextcloud_db_backup.sql
kubectl cp nextcloud-postgresql-0:/tmp/nextcloud_db_backup.sql nextcloud_db_backup.sql
```

### Restore

To avoid issues, it is better to be restoring to the same nextcloud version, as well as the same docker with the same general configuration. For example, going from a docker "linuxserver/nextcloud" v25 to a helm chart using nextcloud/nextcloud v29 may give you a hard time. The inside of the config folder may not be arranged the same.

```console
kubectl cp nextcloud_db_backup.sql postgresql-0:/tmp/nextcloud_db_backup.sql
kubectl exec -it postgresql-0 -c postgresql -- psql -U postgres -d template1 -c "DROP DATABASE \"nextcloud\";"
kubectl exec -it postgresql-0 -c postgresql -- psql -U postgres -d template1 -c "CREATE DATABASE \"nextcloud\";"
kubectl exec -it postgresql-0 -c postgresql -- psql -U postgres -d template1 -c "ALTER DATABASE \"nextcloud\" OWNER TO nextcloud;"
kubectl exec -it postgresql-0 -c postgresql -- psql -U postgres -d template1 -c "GRANT ALL PRIVILEGES ON DATABASE \"nextcloud\" TO nextcloud;"
kubectl exec -it postgresql-0 -c postgresql -- psql -U nextcloud -d nextcloud -f /tmp/nextcloud_db_backup.sql
```

```console
kubectl cp /BACKUP_PATH/nextcloud-data <nextcloud_pod>:/var/www/html/data
kubectl cp /BACKUP_PATH/nextcloud-config <nextcloud_pod>:/var/www/html/config
```

#### DB hostname change

If the information about your DB changes, like the hostname or the DB password, changing them in the secret or in the helm chart is not enough.

You **MUST** also go to the config folder and modify accordingly in the `config.php` file.

#### Reindex files

If you start from scratch and manually copy the users data (/var/www/html/data/USER/files/MY_FILES), you must call `occ` to rescan the data.

```console
kubectl exec --tty --stdin nextcloud-POD -c nextcloud -- su -s /bin/bash www-data
./occ maintenance:mode --on
```

Perform now the copy operations.

When you are finished, you can launch the scan:

```console
kubectl exec --tty --stdin nextcloud-POD -c nextcloud -- su -s /bin/bash www-data
./occ maintenance:mode --off
./occ files:scan --all
```