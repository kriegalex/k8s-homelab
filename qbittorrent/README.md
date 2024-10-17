# QBittorrent Chart

A Helm chart for deploying a QBittorrent client that uses a VPN tunnel provided by PIA.

## Prerequisites
Before deploying this chart, ensure your Kubernetes cluster meets the following requirements:

1. Kubernetes 1.19+ – This chart uses features that are supported on Kubernetes 1.19 and above.
3. Helm 3.0+ – Helm 3 is required to manage and deploy this chart.
3. WireGuard Kernel Module – WireGuard must be available on all nodes in your cluster.

## Installation

### Allow unsafe sysctl

You need to allow two unsafe sysctl to be set by this chart. To do so, we must modify the existing kubelet configuration on the target worker nodes.

```console
cat <<EOF | sudo tee -a /var/lib/kubelet/config.yaml

# Add or update the following section:
allowedUnsafeSysctls:
  - "net.ipv4.conf.all.src_valid_mark"
  - "net.ipv6.conf.all.disable_ipv6"
EOF
```

Restart the kubelet service:

```console
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Persistence

On the control plane:
```console
kubectl apply -f qbittorrent-config-pv.yaml
```

On the applicable worker nodes:
```console
sudo mkdir -p /mnt/qbittorrent/config
sudo chown 1000:1000 /mnt/qbittorrent/config
```

### Define a secret for PIA VPN credentials

```
kubectl create secret generic vpn-pia-credentials \
  --from-literal=pia-user='your_username' \
  --from-literal=pia-password='your_password' \
  --namespace=default
```

### Setup the wireguard config for Proton

Follow [this guide](https://protonvpn.com/support/wireguard-configurations) to obtain your wireguard configuration.

Copy the wg0.conf file to where your persistence was setup, by default `/mnt/qbittorrent/config/wireguard`.

### Helm

1. Add the Helm chart repo

```bash
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
```

2. Inspect & modify the default values (optional)

```bash
helm show values k8s-charts/qbittorrent > custom-values.yaml
```

3. Install the chart

```bash
helm upgrade --install qbit k8s-charts/qbittorrent --values custom-values.yaml
```

## QBittorrent login

The login is `admin`. The password is visible in the logs of the qbittorent app the first time you start it:

```
kubectl logs qbit-qbittorrent-POD_NAME
```

Replace POD_NAME by the name of your pod (`kubectl get pods`).

## Port Forwarding

The port forwarding should be handled automatically by the docker image, if the correct environment variables have been set.

You can check it in the logs:

```console
kubectl logs qbit-qbittorrent-POD_NAME
```

Expected result:
```console
******** Information ********
To control qBittorrent, access the WebUI at: http://localhost:8080

[INF] [] [VPN] Forwarded port is [PORT].
[INF] [] [QBITTORRENT] Updated forwarded port to [PORT].
```