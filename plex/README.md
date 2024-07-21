# Plex

## Plex worker installation

### Disable SWAP

```
sudo swapoff -a
```

To make it permanent, edit the `/etc/fstab` if needed (comment line with swap img)

```
sudo nano /etc/fstab
```

### Setup kernel parameters

Since we are using Ubuntu 22, we need to load br_netfilter and make sure it persists across reboots.

```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
br_netfilter
EOF
sudo modprobe br_netfilter
```

```
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### Install the container runtime

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```
sudo apt-get install -y containerd.io
```

```
containerd config default | sudo tee /etc/containerd/config.toml
sudo nano /etc/containerd/config.toml
```

You must look for the `runc.options` portion: 
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

```
sudo systemctl restart containerd
```

### Kubernetes runtimes

These instructions are for Kubernetes v1.30.

1. Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg nfs-common
```

2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
> Note: In releases older than Debian 12 and Ubuntu 22.04, directory /etc/apt/keyrings does not exist by default, and it should be created before the curl command.

3. Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni
```

5. (Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```

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

#### Verify the cluster plugin

Follow [these instructions](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html#testing-and-demos) to run a docker image inside your cluster that will test that the Intel GPU is working as intended.

You need to [install Docker](https://docs.docker.com/engine/install/) and the `make` tools for your OS (`apt install build-essential` on Ubuntu/Debian).

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

If you have a kubernetes cluster with nginx as ingress (or another ingress), as well as cert-manager (see [here](../ingress/README.md)), you can setup a custom URL for Plex instead of relying on the traditional forwarding of port 32400.

In that case, the plex service doesn't need to be of type LoadBalancer, and you can enable the ingress portion of the plex helm configuration.

Don't forget to setup the plex server with the custom URL under `Settings > Server > Network > Custom server access URLs`:

```
https://plex.mycustomdomain.com:443
```