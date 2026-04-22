# Cấu hình các node
## Cấu hình node
* Cài containerd 
```
apt install containerd
```
* Disable all  [swap spaces](https://phoenixnap.com/kb/swap-space)  with the **swapoff**  command:

```
sudo swapoff -a
```

Then use the  [sed command](https://phoenixnap.com/kb/linux-sed) below to make the necessary adjustments to the  _/etc/fstab_  file:

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

* Load the required  **containerd**  modules. Start by opening the containerd configuration file in a  [text editor](https://phoenixnap.com/kb/best-linux-text-editors-for-coding), such as  [nano](https://phoenixnap.com/kb/use-nano-text-editor-commands-linux):

```
sudo nano /etc/modules-load.d/containerd.conf
```

* Add the following two lines to the file:

```
overlay
br_netfilter
```
Save the file and exit.

* Next, use the [modprobe command](https://phoenixnap.com/kb/modprobe-command) to add the modules:

```
sudo modprobe overlay
```

```
sudo modprobe br_netfilter
```

* Open the **k8s.conf** file to configure Kubernetes networking:

```
sudo nano /etc/sysctl.d/k8s.conf
```

* Add the following lines to the file:

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```
* Reload the configuration by typing:

```
sudo sysctl --system
```
* Stop and disable  **AppArmor**:

```
sudo systemctl stop apparmor && sudo systemctl disable apparmor
```
* Restart  **containerd**:

```
sudo systemctl restart containerd.service
```

## Cài đặt K8S
### Trên tất cả các node
1.  Update the  `apt`  package index and install packages needed to use the Kubernetes  `apt`  repository:
```bash
unlink  /etc/resolv.conf
ln  -sf  /run/systemd/resolve/resolv.conf  /etc/resolv.conf
```
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg socat
```
2.  Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
3.  Add the appropriate Kubernetes  `apt`  repository. Please note that this repository have packages only for Kubernetes 1.31; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
    
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

```
4.  Update the  `apt`  package index, install kubelet, kubeadm and kubectl, and pin their version:
    
```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
    
5.  (Optional) Enable the kubelet service before running kubeadm:
```shell
sudo systemctl enable --now kubelet
```

### Trên masternode

1. Cấu hình ***cgroup-driver***  
***Nếu sử dụng cgroup-driver là cgroupfs***
* Open the  **kubelet**  file in a text editor.

```bash
sudo nano /etc/default/kubelet
```

* Add the following line to the file:

```yaml
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

Save and exit.

* Reload the configuration and restart the kubelet:

```bash
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```
---
***Nếu sử dụng cgroup-driver là systemd (default của k8s)***
* Chỉnh sửa nội dung trong file ***/etc/containerd/config.toml***
* Nếu file ko tồn tại thì chạy lệnh sau để gen config
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
#### Configuring the  `systemd`  cgroup driver[](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)
* To use the  `systemd`  cgroup driver in  `/etc/containerd/config.toml`  with  `runc`, set
```yaml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

```
```bash
systemctl restart containerd
```
---

2. Initialize the cluster by typing:

```bash
sudo kubeadm init --control-plane-endpoint=<master_node_ip> --upload-certs --pod-network-cidr=12.10.0.0/16 --service-cidr=12.80.0.0/12 --service-dns-domain "oke.local"
```
3. Create a  directory  for the Kubernetes cluster:

```bash
mkdir -p $HOME/.kube
```

4. Copy the configuration file to the directory:

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

5. Change the ownership of the directory to the current user and group using the  [chown command](https://phoenixnap.com/kb/linux-chown-command-with-examples):

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

6. Get token to join node in control plane
```
kubeadm token create --print-join-command
```
Sau khi chạy lệnh này sẽ in ra một command sử dụng để join các workernode. Copy lệnh này và chạy trên các workernode

## Cài đặt Calico CNI plugin
1. Install tigera-operator
```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```
2. Use the  [wget command](https://phoenixnap.com/kb/wget-command-with-examples)  to download  _custom-resources.yaml_:

```
wget https://docs.projectcalico.org/manifests/custom-resources.yaml
```
3. Lấy địa chỉ ip range của cluster
* Hiển thị Ip range của pod
```bash
kubectl cluster-info dump | grep -m 1 cluster-cidr
```
```bash
kubectl describe cm kubeadm-config -n kube-system |grep Subnet
kubeadm config view | grep Subnet
```
* Hiển thị ip range của service
```bash
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
```

4. Open the downloaded file in a text editor.

```
nano custom-resources.yaml
```
Thay field ```spec.calicoNetwork.ipPools.cidr``` bằng ip range cidr của pod lấy được ở trên
5. Apply the configuration
```
kubectl create -f custom-resources.yaml
```
6. Show ip block ip calico
```sh
calicoctl ipam show --show-blocks
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAwOTQxNzMzOF19
-->