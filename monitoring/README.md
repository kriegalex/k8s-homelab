# Kubernetes Grafana Dashboards Installation Guide

This guide will help you install the Kubernetes Grafana Dashboards from the [dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes) repository on top of a standard `kube-prometheus-stack` deployment using Helm.

## Prerequisites

- A Kubernetes cluster up and running.
- `kubectl` configured to access your cluster.
- Helm 3 installed on your local machine.
- The `kube-prometheus-stack` Helm chart deployed on your cluster.

## Step 1: Deploy the kube-prometheus-stack

If you haven't already deployed the `kube-prometheus-stack`, you can install it using the following Helm commands:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install --create-namespace --namespace monitoring kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

This command installs the kube-prometheus-stack in the monitoring namespace. More information [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack).

You can see and modify default values using:

```bash
helm show values prometheus-community/kube-prometheus-stack > custom-values.yaml
```

```bash
helm upgrade --install --create-namespace --namespace monitoring kube-prometheus-stack prometheus-community/kube-prometheus-stack -f custom-values.yaml
```

### Grafana login

The default user is `admin`.

The password must be retrieved from kubernetes secrets:

```bash
kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

## Step 2: Install the Kubernetes Grafana Dashboards
To install the dashboards, follow these steps:

### 1. Configure Helm Values
You need to configure Helm values to integrate these dashboards with the existing Grafana instance deployed by the kube-prometheus-stack. You can use the provided [grafana-dashboards.yaml](./grafana-dashboards.yaml) file as a reference.

### 2. Deploy the Dashboards
If you did not already integrate them during `kube-prometheus-stack` install, deploy the dashboards using Helm:

```bash
helm upgrade --namespace monitoring kube-prometheus-stack prometheus-community/kube-prometheus-stack -f grafana-dashboards.yaml
```

### 3. Verify the Installation

Once the installation is complete, you should see the new Kubernetes dashboards in the Grafana instance under the Kubernetes folder.

Access your Grafana instance (usually via http://<grafana-ip>:3000), log in, and navigate to the dashboards section to verify the Kubernetes dashboards have been successfully installed.

## Conclusion
You have now installed the Kubernetes Grafana Dashboards on top of the kube-prometheus-stack. You can explore these dashboards to gain insights into the performance and health of your Kubernetes cluster.

For more details or advanced configurations, refer to the official [dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes) repository.