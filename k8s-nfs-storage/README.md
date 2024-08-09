# NFS shares

## NFS dependencies

Make sure to install the nfs client tools on all worker nodes:

```console
sudo apt-get install -y nfs-common
```

## Media

```console
kubectl apply -f nfs-anime-pv.yaml
kubectl apply -f nfs-anime-pvc.yaml

kubectl apply -f nfs-tv-pv.yaml
kubectl apply -f nfs-tv-pvc.yaml

kubectl apply -f nfs-movies-pv.yaml
kubectl apply -f nfs-movies-pvc.yaml
```

## Torrent

```console
kubectl apply -f nfs-torrent-pv.yaml
kubectl apply -f nfs-torrent-pvc.yaml
```

## Nextcloud

```console
kubectl apply -f nfs-nextcloud-pv.yaml
kubectl apply -f nfs-nextcloud-pvc.yaml
```

## Immich

```console
kubectl apply -f nfs-immich-pv.yaml
kubectl -n immich apply -f nfs-immich-pvc.yaml
```

## Gitea

```console
kubectl apply -f nfs-gitea-pv.yaml
kubectl apply -f nfs-gitea-pvc.yaml
```

## Paperless

```console
kubectl apply -f nfs-paperless-pv.yaml
kubectl apply -f nfs-paperless-pvc.yaml
```

## Backup

```console
kubectl apply -f nfs-backup-pv.yaml
kubectl apply -f nfs-backup-pvc.yaml
```