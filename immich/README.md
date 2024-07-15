# Immich

## values.yaml

Setup your own values.yaml before installing via helm

## NFS

Make sure NFS tools are installed on all nodes

```
sudo apt-get -y install nfs-common
```

## Namespace

```
 kubectl create namespace immich 
```

## Redis persistence

```
kubectl create -f immich-redis-data-pv.yaml -n immich
```

### Setup permissions

```
sudo mkdir -p /mnt/immich/redis-data
sudo chown 1001:root /mnt/immich/redis-data
```

## PSQL persistence

```
kubectl create -f immich-psql-data-pv.yaml -n immich
```

### Setup permissions

```
sudo mkdir -p /mnt/immich/psql-data
sudo chown 1001:root /mnt/immich/psql-data
```

## Immich persistence

```
kubectl create -f immich-library-pvc.yaml -n immich
```

```
kubectl create -f immich-ml-data-pv.yaml -f immich-ml-data-pvc.yaml -n immich
```

## Installation

```
helm repo add immich https://immich-app.github.io/immich-charts
helm repo update
```

```
helm install --namespace immich immich immich/immich -f custom-values.yaml
```

## Uninstall

```
helm -n immich uninstall immich
kubectl -n immich delete -f immich-ml-data-pvc.yaml -f immich-ml-data-pv.yaml \
  -f immich-psql-data-pv.yaml -f immich-redis-data-pv.yaml -f immich-library-pvc.yaml
kubectl -n immich delete pvc data-immich-postgresql-0 redis-data-immich-redis-master-0
```