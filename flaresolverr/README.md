# Sonarr

## Add repository

```
helm repo add k8s-home-lab-repo https://k8s-home-lab.github.io/helm-charts/
helm repo update
```

## Install chart

```
helm upgrade --install flaresolverr k8s-home-lab-repo/flaresolverr -f custom-values.yaml
```

## Default values

```
#
# IMPORTANT NOTE
#
# This chart inherits from our common library chart. You can check the default values/options here:
# https://github.com/k8s-at-home/library-charts/tree/main/charts/stable/common/values.yaml
#

image:
  # -- image repository
  repository: ghcr.io/flaresolverr/flaresolverr
  # -- image pull policy
  pullPolicy: IfNotPresent
  # -- image tag
  tag:

# -- environment variables. See more environment variables in the [flaresolverr documentation](https://github.com/FlareSolverr/FlareSolverr#environment-variables).
# @default -- See below
env:
  # -- Set the container timezone
  TZ: UTC

# -- Configures service settings for the chart.
# @default -- See values.yaml
service:
  main:
    ports:
      http:
        port: 8191

ingress:
  # -- Enable and configure ingress settings for the chart under this key.
  # @default -- See values.yaml
  main:
    enabled: false
```