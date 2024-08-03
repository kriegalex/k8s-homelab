# Sonarr

## Add repository

```
helm repo add k8s-charts https://kriegalex.github.io/k8s-charts/
helm repo update
```

## Install chart

```
kubectl apply -f sonarr-tv-config-pv.yaml
helm upgrade --install sonarr k8s-charts/sonarr -f custom-values.yaml
```

## Default values

```
# Default values for sonarr.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: hotio/sonarr
  pullPolicy: IfNotPresent
  # -- Overrides the image tag whose default is the chart appVersion.
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

deployment:
  # -- Deployment strategy
  strategy:
    type: Recreate

serviceAccount:
  # -- Specifies whether a service account should be created
  create: true
  # -- Annotations to add to the service account
  annotations: {}
  # -- The name of the service account to use.
  # -- If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations:
    {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources:  {}
  # -- We usually recommend not to specify default resources and to leave this as a conscious
  # -- choice for the user. This also increases chances charts run on environments with little
  # -- resources, such as Minikube. If you do want to specify resources, uncomment the following
  # -- lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}

env:
  PUID: "1000"
  PGID: "1000"
  TZ: "Europe/London"
  UMASK: "002"

persistence:
  storageClass: ""
  existingClaim: ""
  enabled: true
  accessMode: ReadWriteOnce
  size: 800Mi
  # -- if you need any additional volumes, you can define them here
  additionalVolumes: []
  # -- if you need any additional volume mounts, you can define them here
  additionalMounts: []
```