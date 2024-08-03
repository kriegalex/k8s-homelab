# QBittorrent + Gluetun Chart

A Helm chart for deploying a QBittorrent client that uses a VPN tunnel provided by Gluetun.

## Installation

### Persistence

On the control plane:
```console
kubectl apply -f gluetun-config-pv.yaml
kubectl apply -f qbittorrent-config-pv.yaml
```

On the applicable worker nodes:
```console
sudo mkdir -p /mnt/qbittorrent/config
sudo chown 1000:1000 /mnt/qbittorrent/config
sudo mkdir -p /mnt/gluetun/config
```

### Define a secret for VPN credentials

```
kubectl create secret generic vpn-credentials \
  --from-literal=OPENVPN_USER='your_username' \
  --from-literal=OPENVPN_PASSWORD='your_password' \
  --namespace=default
```

### Helm

1. Add the Helm chart repo

```bash
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
```

2. Inspect & modify the default values (optional)

```bash
helm show values k8s-charts/qbittorrent-gluetun > custom-values.yaml
```

3. Install the chart

```bash
helm upgrade --install qbit k8s-charts/qbittorrent-gluetun --values custom-values.yaml
```

## QBittorrent login

The login is `admin`. The password is visible in the logs of the qbittorent app the first time you start it:

```
kubectl logs POD_NAME qbittorrent
```

Replace POD_NAME by the name of your pod (`kubectl get pods`). The two arguments are required because this pod runs two apps.

## Gluetun Port Forwarding

You can get the port obtained from PIA by:

1. Looking at the gluetun app logs

```console
kubectl logs POD_NAME gluetun
```

2. Check the JSON file in the config persistent volume of your app "gluetun"

In my case, the PV for the config persistent volume of gluetun comes from a NFS share on a remote server (storageClass: "nfs-client").

```console
ssh USER@NFS_SERVER
cat /NFS_PATH/config/FILE.json
```

3. Connect to the container directly

```console
kubectl exec --stdin --tty POD_NAME -c gluetun -- /bin/sh
ls /gluetun/
```

The `-c` is required because this pod runs two apps.