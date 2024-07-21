# Plex

## Installation

1. Add the Plex helm repo:
```
helm repo add plex https://raw.githubusercontent.com/plexinc/pms-docker/gh-pages
```

2. Inspect and modify the default values:

```
helm show values plex/plex-media-server > custom-values.yaml
```

You can check an [example here](./custom-values.yaml).

3. Create the pms-config [PersistentVolume](./plex-pv.yaml):

plex-pv.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pms-config
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 200Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/pms-config"
 ```
 
 ```
 kubectl apply -f pms-pv.yaml
 ```
 
 4. Install Plex

```
helm install plex-media-server plex/plex-media-server -f custom-values.yaml
```

If you want to import an existing library, don't setup the server yet. Otherwise, you can go ahead and start the setup.

The plex server should be reachable either via the Ingress you setup or via the `EXTERNAL-IP` the service obtained from the LoadBalancer:

Check the LoadBalancer IP:
```
kubectl get svc
```

Check the Ingress URL:
```
kubectl get ingress
```

Other scenarios are possible, but outside the scope of this guide.

### Import an existing library

(optional) Backup the existing library:

```
tar -czvf pms.tgz Library
```

The "Library" folder should look like that inside: `Library/Application Support/Plex Media Server`.

Don't forget to stop the server before doing the backup. The location of the library depends on the [type of server you are running](https://support.plex.tv/articles/202915258-where-is-the-plex-media-server-data-directory-located/). If you are running docker, the path is setup in the docker-compose file or in the Dockerfile.

In the custom values, you can enable the script that will wait for you to copy an archive named `pms.tgz`. First, copy it to the pod using a different name:

```
kubectl cp pms.tgz <namespace>/<podname>:/pms.tgz.up -c <release name>-pms-chart-pms-init
```

Then, move it so it has the proper name:

```
kubectl exec -n <namespace> --stdin --tty <pod>  -c <release name>-pms-chart-pms-init h  -- mv /pms.tgz.up /pms.tgz
```

As soon, as you move it, the script should start processing it.

## Plex configuration

### Custom URL
To be able to distinguish from local to distant access to the server, we must configure Plex to use a custom URL: Settings -> Network -> Custom URL. This is also why I don't use a simple NodePort for the plex service. Otherwise, all the traffic looks like it is coming from 10.201.0.1 (the gateway of my pod CIDR, see above).

I'm not totally sure you need both the custom URL AND the external access using port forwarding, but it started to work in my setup after both were setup.