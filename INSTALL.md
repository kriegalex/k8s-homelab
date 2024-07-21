# Kubernetes installation

## Goal

Deploy a control plane and 1/2/3/... workers.

One of the worker will run a Plex server with hardware transcoding on an Intel ARC GPU.

## Environment

- Ubuntu 22.04.3 LTS with 6.5 kernel (HWE)
- Intel ARC A380 GPU for hardware transcoding
- Fast local storage for Plex metadata
- Simple home networking

## Control plane installation

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

#### Understanding the Parameters
##### net.bridge.bridge-nf-call-ip6tables
This parameter enables the bridge network driver to pass IPv6 traffic to ip6tables for processing. In simpler terms, it allows ip6tables to filter traffic that is bridged (i.e., passed from one network interface to another without being processed by the host's IP stack).

##### net.bridge.bridge-nf-call-iptables
Similarly, this parameter enables the bridge network driver to pass IPv4 traffic to iptables for processing. This allows iptables to filter bridged IPv4 traffic.

##### net.ipv4.ip_forward
The net.ipv4.ip_forward parameter controls whether the Linux kernel forwards IPv4 packets. By default, a Linux machine acts as a host, not a router, and does not forward IP packets from one network interface to another. This parameter must be enabled if you want your Linux machine to act as a router, forwarding packets between different network interfaces.

#### Scenario Explanation
In a Kubernetes cluster, suppose you have a service that exposes port 6379 (commonly used by Redis) and allows pods to communicate with this service.

1. Service Definition:

You define a Kubernetes service that maps port 6379 to one or more backend pods.
Kubernetes creates iptables rules to handle this mapping, ensuring that traffic destined for the service IP on port 6379 is correctly forwarded to the appropriate pods.

2. Bridged Traffic:

Pods communicate over the network using virtual network interfaces (veth pairs) and bridges. For example, a pod on Node A might want to communicate with the Redis service, which is mapped to port 6379.

##### Importance of net.bridge.bridge-nf-call-iptables

What Happens When This Parameter Is Set (net.bridge.bridge-nf-call-iptables = 1):

- Packet Handling:
  - When a packet destined for port 6379 arrives at the network bridge interface on the node, the Linux kernel passes this packet to iptables for processing.
  - Iptables applies the rules defined by Kubernetes, such as NAT and port forwarding rules, to ensure the packet is routed correctly to the backend pod running Redis.

What Happens When This Parameter Is Not Set (net.bridge.bridge-nf-call-iptables = 0):

- Packet Handling:
  - When a packet destined for port 6379 arrives at the network bridge interface, it bypasses iptables because the kernel is not configured to pass bridged traffic to iptables.
  - The packet does not get processed by the iptables rules, meaning the necessary NAT and port forwarding rules are not applied.
  - As a result, the packet may not reach the intended pod running Redis, leading to communication failures and service disruptions.

### Install the container runtime

#### Install runc

```
wget -q --show-progress https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc 
```

#### Install CNI plugins

```
wget -q --show-progress https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz

sudo mkdir -p /opt/cni/bin

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.1.tgz
```

#### Install containerd

```
wget -q --show-progress \
  https://github.com/containerd/containerd/releases/download/v1.7.16/containerd-1.7.16-linux-amd64.tar.gz \
  https://raw.githubusercontent.com/containerd/containerd/v1.7.16/containerd.service

sudo tar Cxzvf /usr/local containerd-1.7.16-linux-amd64.tar.gz

sudo mkdir /etc/containerd
/usr/local/bin/containerd config default | sudo tee /etc/containerd/config.toml

sudo mkdir -p /usr/local/lib/systemd/system

sudo mv containerd.service /usr/local/lib/systemd/system/
```

As per the [kubernetes doc](https://kubernetes.io/docs/setup/production-environment/container-runtimes/), if the cgroup is provided by `systemd`, one setting must be changed in config.toml:

```
sudo nano /etc/containerd/config.toml
```

You must look for the `runc.options` portion: 
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

And now, enable the service:

```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Install kubernetes

#### Install the deployment tools

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
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. (Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```

#### Bootstrap the cluster
You can change the pods CIDR and service CIDR to match your needs. More information [here](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init).
```
sudo kubeadm init --pod-network-cidr 10.244.0.0/16
```

If the command is successful, you get instructions at the end, to help you setup the admin config. Set it up on the user that will access the cluster (probably your current admin user):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

(optional) By default, your cluster will not schedule Pods on the control plane nodes for security reasons. If you want to be able to schedule Pods on the control plane nodes, for example for a single machine Kubernetes cluster, run:
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Helm
The instructions can be [viewed here]().
1. Download
```
wget https://get.helm.sh/helm-v3.15.0-linux-amd64.tar.gz
```

2. Install
```
tar -zxvf helm-v3.15.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

3. Check
```
helm help
```

### Install the network addon

We will later install MetalLB to provide the load balancing to our cluster. [Check this page](https://metallb.io/installation/network-addons/) for the latest compatibility between MetalLB and network addons.

I have chosen Flannel. Instructions can be [found here](https://github.com/flannel-io/flannel/blob/master/README.md).

Install Flannel:

```
# Needs manual creation of namespace to avoid helm error
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm install flannel --set podCidr="10.244.0.0/16" --namespace kube-flannel flannel/flannel
```

From now on, the coreDNS pods should start working, since the network addon is active:
```
kubectl get pods -A -o wide
```
You should see the coredns pods in "running" state:
```
kube-system  coredns-XXX-YY  1/1  Running   1 
kube-system  coredns-XXX-YY  1/1  Running   1 
```

### MetalLB load balancer

The instructions can be [viewed here](https://metallb.io/installation/).

1. First, we must change the strictARP setting:
```
kubectl edit configmap -n kube-system kube-proxy
```
Change strictARP:
```
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

2. Deploy MetalLB:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
kubectl get pods -n metallb-system
```

3. Set the IP pool:

metallb-ippool.yaml
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lan-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.1.2-10.0.1.254
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lan
  namespace: metallb-system
spec:
  ipAddressPools:
  - lan-pool
```

```
kubectl apply -f metallb-ippool.yaml
```

### NGINX Ingress

In my case, I have an external service called Nginx Proxy Manager already setup, so I don't want to break it. I've setup my domain to point to the IP that MetalLB will give to my plex service. It should be the first in our LAN pool, but please check it to be sure (in case you have other apps running beside Plex).

https://sub.domain.com -> http://10.0.1.2:32400

If you want a nginx as ingress for your kubernetes cluster, you'll need to set it up inside kubernetes. The instructions can be [viewed here](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start).

1. Install
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

2. Cert-manager

You will probably also want to manage some certificates using Let's Enrypt. Please have a look at [cert-manager & let's encrypt](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/). 

## NFS Dynamic Provisioning

[NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) is an automatic provisioner that use your existing and already configured NFS server to support dynamic provisioning of Kubernetes Persistent Volumes via Persistent Volume Claims. Persistent volumes are provisioned as \${namespace}-\${pvcName}-\${pvName}.

```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=10.0.0.2 --set nfs.path=/mnt/user/k8s-data
```

Please check the [k8s-nfs-storage](k8s-nfs-storage/README.md) folder for more information on the possible configurable parameters.

### Manual NFS setup

If you don't want to use NFS directly via helm charts, you must setup first your NFS shares manually.

If the media files are on a remote server, you must create NFS shares on your remote server (probably your media NAS). [More information for Ubuntu 22.04 here](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-22-04).

On your client, where kubernetes and your pod will be running, you must edit the `/etc/fstab` so that they mount at each reboot:

Example:
```
SERVER_IP:SERVER_PATH CLIENT_PATH nfs auto,nofail,noatime,nolock,intr,tcp,rsize=1048576,wsize=1048576,actimeo=1800 0 0
```

## Plex worker installation

### Basic installation

The process to setup the worker node is the same as the control plane node, except that you skip the load balancer, network addon and ingress steps. You simply install kubernetes (kubelet, kubeadm, kubectl) and its dependencies (containerd, runc, CNI, ...).

Then, instead of using `kubeadm init`, you use `kubeadm join`. You may not have written down the join command that `kubeadm` gave you on the control plane node. To create a new `kubeadm join` command, simply use:

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
sudo apt install intel-opencl-icd
clinfo -l
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

#### Misc

Check your BIOS/UEFI settings to see what can be enabled. In my setup, I have set up the following on a B550m from Asrock:

- Main graphics adapter: External (if the CPU has a GPU, avoids having two GPUs in the server, could cause issues)
- SR-IOV: enabled (more details [here](https://docs.nvidia.com/networking/display/mlnxofedv590560125/single+root+io+virtualization+(sr-iov))
- Resizable BAR : enabled (free performance for the GPU)

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

## Home networking

### Static routing
MetalLB will assign an IP from the pool 10.0.1.2 - 10.0.1.254 to your Plex service. The network needs a way to know where to find 10.0.1.X.

There are various ways to deal with that, but in a simple homelab context, the easiest way it to configure your router with a static route that says:

"Any traffic to 10.0.1.0/24 network needs to go through the plex host machine"

In my case, my kubernetes node has the IP 10.0.0.8, so I must create a static route from 10.0.1.0/24 to 10.0.0.8. Then, our network stack on the kubernetes host knows how to deal with traffic for 10.0.1.X.

### Port forwarding
Since homelab networks use NAT (one single public IP), don't forget to forward 32400 to the appropriate IP (10.0.1.2 in my case, the first in the load balancer pool).
