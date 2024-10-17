# Actual Budget

## Persistence

We manage everything manually to avoid PVCs binding to the wrong PV. Also, a homelab K8S is not really a cloud native kubernetes cluster that dynamically asks storage from a cloud service using `storageClass`.

```
kubectl apply -f actual_budget-data-pv.yaml -f actual_budget-data-pvc.yaml
```

### Setup permissions (worker nodes)

```
sudo mkdir -p /mnt/actual-budget/data
```

## Installation

```console
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
helm show values k8s-charts/actual-budget > custom-values.yaml
```

> Adapt any needed values. Check [the original repository](https://github.com/kriegalex/k8s-charts/tree/main/charts/actual-budget) for more information.

```console
helm upgrade --install actual-budget k8s-charts/actual-budget -f custom-values.yaml
```

Create the first superuser:
```console
kubectl exec -it actual-budget-PODID -- su -s /bin/bash paperless -c "./manage.py createsuperuser"
```

## Uninstall

```
helm uninstall actual-budget
kubectl delete -f actual_budget-data-pv.yaml -f actual_budget-data-pvc.yaml
```

## Backup / restore

See the [documentation](https://actualbudget.org/docs/backup-restore/backup). Actual Budget has a built-in mechanism for backuping and restoring the budgets.