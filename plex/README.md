# Plex

## Plex worker installation

### Basic installation

Follow the main [INSTALL.md](../INSTALL.md) to install a worker node.

Then, instead of using `kubeadm init`, you use the `kubeadm join` command that was created during the install of the control plane. 

### Label the worker node

```
kubectl label node WORKER-NODE-NAME node-role.kubernetes.io/worker=worker
```

### Recreate a token

(optional) If you forgot the join command, or if the token expired, you can recreate a token. An expired token will result in the `kubeadm join` command to appear as stuck. You can check what is going on by adding `--v=2` at the end of the command:

```
sudo kubeadm join 10.0.0.10:6443 --token TOKEN --discovery-token-ca-cert-hash SHA --v=2
```

An expired token may give the following error:

> Retrying due to error: could not find a JWS signature in the cluster-info ConfigMap for token ID "TOKEN"

To recreate a new `kubeadm join` command, use:

```
# On the control plane node
kubeadm token create --print-join-command
```

### Enable hardware transcoding for an Intel ARC GPU

#### Drivers
If you are on Ubuntu 22.04 LTS, or on a modern Linux distribution with access to kernel >=6.2, the drivers for an Intel ARC GPU should already be working. More details [here](https://dgpu-docs.intel.com/driver/installation.html#ubuntu-install-steps). On Ubuntu 22.04.3 LTS, the kernel 6.5 is easily available with the [HWE kernel](https://askubuntu.com/questions/1442208/how-to-enable-hwe-on-ubuntu-22-04).

You can check the state of the drivers and of the GPU using HWInfo:
```
sudo apt install hwinfo
hwinfo --display
```

Note the i915 driver and its "active" status:
```
 Driver: "i915"
  Driver Modules: "i915"
  Memory Range: 0xfb000000-0xfbffffff (rw,non-prefetchable)
  Memory Range: 0xb0000000-0xbfffffff (ro,non-prefetchable)
  Memory Range: 0xfc000000-0xfc1fffff (ro,non-prefetchable,disabled)
  IRQ: 94 (96 events)
  Module Alias: "pci:v00008086d000056A6sv00001849sd00006007bc03sc00i00"
  Driver Info #0:
    Driver Status: i915 is active
```

#### Intel OpenCL ICD

To test that OpenCL is working on the node, please install the intel runtime and run clinfo:

```
sudo apt install intel-opencl-icd clinfo
sudo clinfo -l
```

The output for my Intel ARC A310 (please note the ID [0x56a6](https://dgpu-docs.intel.com/devices/hardware-table.html)):
```
Platform #0: Intel(R) OpenCL HD Graphics
 `-- Device #0: Intel(R) Graphics [0x56a6]
```

#### Enable GuC

Use this command to activate GuC in the kernel config:

```
echo "options i915 enable_guc=2" | sudo tee /etc/modprobe.d/i915.conf
```

#### GRUB changes for i915

First, identify which GPU ID is right for you using the Intel Hardware Table. In my case, it is 56a5 for an ARC A380.

Then, we will add some default parameters to GRUB bootloader:
```
sudo nano /etc/default/grub
```

On Ubuntu 22.04.3 LTS, my GRUB_CMDLINE_LINUX_DEFAULT was empty. I modified it with:

```
GRUB_CMDLINE_LINUX_DEFAULT="i915.enable_hangcheck=0 i915.force_probe=56a5"
```

Please note the use of my GPU ID in `i915.force_probe=`.

**Reboot:**

```
sudo reboot
```

#### Install the plugin on the cluster

[Instructions here](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html#installation).

Deploy GPU plugin with the help of NFD (Node Feature Discovery). It detects the presence of Intel GPUs and labels them accordingly. GPU plugin’s node selector is used to deploy plugin to nodes which have such a GPU label.
```
kubectl apply -k 'https://github.com/intel/intel-device-plugins-for-kubernetes/deployments/nfd?ref=v0.30.0'
kubectl apply -k 'https://github.com/intel/intel-device-plugins-for-kubernetes/deployments/nfd/overlays/node-feature-rules?ref=v0.30.0'
kubectl apply -k 'https://github.com/intel/intel-device-plugins-for-kubernetes/deployments/gpu_plugin/overlays/nfd_labeled_nodes?ref=v0.30.0'
```

You can verify that the plugin has been installed on the expected nodes by searching for the relevant resource allocation status on the nodes:
```
kubectl get nodes -o=jsonpath="{range .items[*]}{.metadata.name}{'\n'}{' i915: '}{.status.allocatable.gpu\.intel\.com/i915}{'\n'}"
```

**(IMPORTANT) Check the label of this ARC gpu for pod affinity:**

```
kubectl describe node plex
```

Output example:
```
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    gpu.intel.com/device-id.0300-56a5.count=1
                    gpu.intel.com/device-id.0300-56a5.present=true
                    gpu.intel.com/family=A_Series
                    intel.feature.node.kubernetes.io/gpu=true
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=plex
                    kubernetes.io/os=linux
```

We want to know the ID of our GPU, in this example `0300-56a5`. This will be useful for the [custom values](./plex/custom-values.yaml) during the [Plex installation](./plex/README.md).

#### (optional) Verify the plugin installation

On Ubuntu, install the common build package that will have `make` & Cie tools:

```
sudo apt-get install -y build-essential
```

Clone the intel repository:

```
git clone https://github.com/intel/intel-device-plugins-for-kubernetes.git
cd intel-device-plugins-for-kubernetes/
```



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
  storageClassName: fast-nvme # by default is "standard"
  hostPath:
    path: "/mnt/pms-config"
 ```
 
 ```
 kubectl apply -f plex-pv.yaml
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

If you used the script in the helm custom values, plex pod should be in an Init status:

```
NAME                                               READY   STATUS     RESTARTS   AGE
plex-media-server-0                                0/1     Init:0/1   0          55s
```

(optional) Backup the existing library:

```
tar -czvf pms.tgz Library
```

The "Library" folder should look like that inside: `Library/Application Support/Plex Media Server`.

Don't forget to stop the server before doing the backup. The location of the library depends on the [type of server you are running](https://support.plex.tv/articles/202915258-where-is-the-plex-media-server-data-directory-located/). If you are running docker, the path is setup in the docker-compose file or in the Dockerfile.

In the custom values, you can enable the script that will wait for you to copy an archive named `pms.tgz`. First, copy it to the pod using a different name:

```
kubectl cp pms.tgz plex-media-server-0:/pms.tgz.up -c plex-media-server-pms-init
```

Then, move it so it has the proper name:

```
kubectl exec --stdin --tty plex-media-server-0 -c plex-media-server-pms-init h -- mv /pms.tgz.up /pms.tgz
```

As soon, as you move it, the script should start processing it.

#### I/O timeout error

If you have an "i/o timeout" error while using `kubectl cp`, consider copying the archive directly onto the plex worker node.

From the worker node:

```
# adapt it if you modified the PersistentVolume for plex
cd /mnt/pms-config
sudo scp user@IP:/mnt/user/backup/plex/pms.tgz .
```

From the control plane:

```
kubectl exec --stdin --tty plex-media-server-0 -c plex-media-server-pms-init h -- mv /config/pms.tgz /pms.tgz
```

## Plex configuration

### Custom URL
To be able to distinguish from local to distant access to the server, we must configure Plex to use a custom URL: Settings -> Network -> Custom URL. This is also why I don't use a simple NodePort for the plex service. Otherwise, all the traffic looks like it is coming from 10.201.0.1 (the gateway of my pod CIDR, see above).

I'm not totally sure you need both the custom URL AND the external access using port forwarding, but it started to work in my setup after both were setup.