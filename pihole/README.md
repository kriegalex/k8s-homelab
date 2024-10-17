# Pi-hole

## Persistence

We manage everything manually to avoid PVCs binding to the wrong PV. Also, a homelab K8S is not really a cloud native kubernetes cluster that dynamically asks storage from a cloud service using `storageClass`.

```
kubectl apply -f pihole-config-pv.yaml -f pihole-config-pvc.yaml
```

### Setup permissions (worker nodes)

```
sudo mkdir -p /mnt/pihole/config
```

## Add repository

```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

## Custom values

```console
helm show values mojo2600/pihole > custom-values.yaml
```

## Install chart

```
helm upgrade --install pihole mojo2600/pihole -f custom-values.yaml
```

