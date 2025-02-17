# Essential Steps

## Environment

RHEL 8 or 9
Kubernetes 1.31+

## Prerequisites (Preparing the hosts)

* Set **SELinux** to permissive mode

```bash
# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

* Firewall Rules

```bash
sudo firewall-cmd --permanent --zone=public --add-port=6443/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10248/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10250/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-all

# or just turn off firewalled
sudo systemctl stop firewalld
```

* Enable IPv4 packet forwarding and letting iptables see bridged traffic

```bash
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that net.ipv4.ip_forward is set to 1
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# Verify that the br_netfilter, overlay modules are loaded 
lsmod | grep br_netfilter
lsmod | grep overlay
```

## Container runtimes (Choosing CRI-O as example)

* add CRI-O yum/dnf repository

```bash
sudo cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/rpm/repodata/repomd.xml.key
EOF
```

* install and start CRI-O

```bash
sudo dnf install -y cri-o
sudo systemctl enable --now crio
```

## Installing kubeadm (kubectl, kubelet)

* Add the Kubernetes yum/dnf repository

```bash
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
sudo cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

* Install kubelet, kubeadm and kubectl:

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

## Creating a cluster with kubeadm

* Create a new cluster with a configuration file.

```bash
sudo kubeadm init --config kubeadm-config.yaml

# Output should be similar to following:
[init] Using Kubernetes version: v1.32.2
......

Your Kubernetes control-plane has initialized successfully!

......

kubeadm join 192.168.6.91:6443 --token 1ak9ak.kcwm9mcet1zkzke9 \
        --discovery-token-ca-cert-hash sha256:206fcfb491fa3c7033a60b4b2aafda3133366c7a9e07a53f9d53753a54f6a3c1
```

* To start using your cluster, you need to run the following as a regular user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get node -o wide

# Output should be similar to following, **STATUS** should be "NotReady" due to no Network Policy Provider has been installed yet:
[onme@rhel9-1 ~]$ kubectl get node -o wide
NAME          STATUS     ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel9-1.zzz   NotReady   control-plane   24s   v1.32.2   192.168.6.91   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0
```

* Add more node

```bash
sudo kubeadm join 192.168.6.91:6443 --token 1ak9ak.kcwm9mcet1zkzke9 \
        --discovery-token-ca-cert-hash sha256:206fcfb491fa3c7033a60b4b2aafda3133366c7a9e07a53f9d53753a54f6a3c1

# get all nodes
[onme@rhel9-1 ~]$ kubectl get node -o wide
NAME          STATUS     ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel9-1.zzz   NotReady   control-plane   5m55s   v1.32.2   192.168.6.91   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0
rhel9-3.zzz   NotReady   <none>          80s     v1.32.2   192.168.6.93   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0
```

## Install a Network Policy Provider (Calico)

There are several Network Policy Providers, **Antrea**,  **Calico**,  **Cilium**, **Flannel**, **Romana**, **Weave Net**, **Kube-router** etc. Choose **Calico** in this tutorial.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
kubectl create -f calico-custom-resources.yaml
```

```bash

watch kubectl get pods -n calico-system

# Output should be refresh every 2 seconds:
Every 2.0s: kubectl get pods -n calico-system

NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-59ffc58b56-dkkl4   0/1     Pending             0          11s
calico-node-lrvvh                          0/1     Running             0          11s
calico-typha-585bc48f5c-8dpwn              1/1     Running             0          11s
csi-node-driver-wnn9c                      0/2     ContainerCreating   0          11s

# After a couple of minutes, the **STATUS** should be "Ready"
[onme@rhel9-1 kubernetes]$ kubectl get node -o wide
NAME          STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel9-1.zzz   Ready    control-plane   7m44s   v1.32.2   192.168.6.91   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0
rhel9-2.zzz   Ready    <none>          103s    v1.32.2   192.168.6.92   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0
rhel9-3.zzz   Ready    <none>          78s     v1.32.2   192.168.6.93   <none>        Red Hat Enterprise Linux 9.5 (Plow)   5.14.0-503.23.2.el9_5.x86_64   cri-o://1.33.0

```

By now, the K8S cluster is been succeffully created. You can add as many node as you want later on.

## Miscellaneous

* Any step screwed up, you can start all over by reset the cluster.

```bash
# Need to be run at every nodes
sudo kubeadm reset -f
```

* Forget to save the **join** commmand

```bash
kubeadm token create --print-join-command

# Output
kubeadm join 192.168.6.91:6443 --token z60yk2.qlkisyryj9shi08a \
        --discovery-token-ca-cert-hash sha256:e6134e5c16f8d1d40e754d0e73472f346a6adcde53ac107e04615e587cc24335
```
