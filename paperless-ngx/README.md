# Paperless-ngx

## Persistence

We manage everything manually to avoid PVCs binding to the wrong PV. Also, a homelab K8S is not really a cloud native kubernetes cluster that dynamically asks storage from a cloud service using `storageClass`.

```
kubectl apply -f paperless-media-pv.yaml -f paperless-media-pvc.yaml
kubectl apply -f paperless-psql-pv.yaml -f paperless-psql-pvc.yaml
```

In my setup, I reuse an existing nfs share with subpaths for the `CONSUME` and `EXPORT` folders. `DATA` is in the [same path](https://docs.paperless-ngx.com/configuration/#PAPERLESS_MEDIA_ROOT) as `MEDIA`.

> Note: make sure the files to consume in the NFS share have the proper permissions. In unRAID, by default, this means that the files are owned by `nobody:users` instead of `paperless:paperless` (1000:1000).
> 
> Make sure any older file to consume is copied with the archive option (i.e. `cp -a` or `rsync -a`).

### Setup permissions (worker nodes)

```
sudo mkdir -p /mnt/paperless/media
sudo mkdir -p /mnt/paperless/psql
sudo chown 1001:1001 /mnt/paperless/psql
```

## Installation

```console
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
helm show values k8s-charts/paperless-ngx > custom-values.yaml
```

> Adapt any needed values. Check [the original repository](https://github.com/kriegalex/k8s-charts/tree/main/charts/paperless-ngx) for more information.

```console
helm upgrade --install paperless-ngx k8s-charts/paperless-ngx -f custom-values.yaml
```

Create the first superuser:
```console
kubectl exec -it paperless-ngx-PODID -- su -s /bin/bash paperless -c "./manage.py createsuperuser"
```

## Uninstall

```
helm uninstall paperless-ngx
kubectl delete -f paperless-media-pvc.yaml -f paperless-media-pv.yaml \
  -f paperless-psql-pvc.yaml -f paperless-psql-pv.yaml
```

## Backup / restore

See the [documentation](https://docs.paperless-ngx.com/administration/). Paperless-ngx has a built-in mechanism for backuping and restoring.