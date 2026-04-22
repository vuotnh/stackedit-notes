<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Import image vào cluster](#import-image-vào-cluster)
- [Kiến trúc các thư mục config trong k8s](#kin-trúc-các-th-mc-config-trong-k8s)
- [Một số lỗi khi cài đặt cluster](#mt-s-li-khi-cài-t-cluster)
- [Các lỗi liên quan tới network CNI plugin](#các-li-liên-quan-ti-network-cni-plugin)
   * [Cilium plugin](#cilium-plugin)
- [Khi core dns pod bị CrashLoopBackOff](#khi-core-dns-pod-b-crashloopbackoff)
   * [Cài đặt crictl để exec vào container trên node](#cài-t-crictl-exec-vào-container-trên-node)
- [Kubernetes IP address ranges](#kubernetes-ip-address-ranges)
   * [Cluster networking types](#cluster-networking-types)
- [Link tham khảo](#link-tham-kho)

<!-- TOC end -->

<!-- TOC --><a name="import-image-vào-cluster"></a>

## Import image vào cluster
* export image từ docker
```bash
docker save <image-name> -o <filename.tar>
```
* import image vào cluster
```bash
ctr -n=k8s.io images import <filename-from-previous-step>
crictl images
```
## Kiến trúc các thư mục config trong k8s
* Các config của kubelet về CNI, container runtime được lưu ở thư mục
```
/var/lib/kubelet
/etc/cni/net.d
```
* Log của các container pod được lưu trong thư mục ***/var/log/pods***
* Tham khảo ở [đây](https://yuminlee2.medium.com/kubernetes-folder-structure-and-functionality-overview-5b4ec10c32bf) hoặc [ở đây](https://www.notion.so/Kubernetes-CNI-container-network-interface-1199814fd71d80d8a003d271ff52af8e?pvs=4)
## Một số lỗi khi cài đặt cluster

* Show key to join node
```bash
kubeadm token create --print-join-command
```
* Để ko bị lỗi khi cài flannel, lúc init cần có cidr 
```
sudo kubeadm init --control-plane-endpoint=masternode --upload-certs --pod-network-cidr=10.244.0.0/16
```
* Lỗi container runtime, thường liên quan đến containerd, tắt dịch vụ ***apparmor***
```bash
sudo systemctl stop apparmor && sudo systemctl disable apparmor
```
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
* [Tham khảo](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#configure-kubelets-using-kubeadm)
---
## Các lỗi liên quan tới network CNI plugin
* CNI workflow model: [link](https://yuminlee2.medium.com/kubernetes-container-network-interface-cni-ee5b21514664)

### Cilium plugin
* Fix lỗi cilium trên workernode
```
cilium install --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.44.0.0/16" --set bgpControlPlane.enabled=true
```

---
## Khi core dns pod bị CrashLoopBackOff
* [tham khảo](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
* show toàn bộ configmap
```
kubectl get configmap --all-namespaces
```
* Sửa configmap có tên ***coredns***
```bash
kubectl edit cm coredns -n kube-system
```
* Sửa ```proxy . /etc/resolv.conf``` thành ```proxy . 8.8.8.8```
* Xóa 2 pod để tạo lại pod mới

### Cài đặt crictl để exec vào container trên node
[tham khảo]()
```bash
VERSION="v1.30.0" # check latest version in /releases page
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

##  Kubernetes IP address ranges
-   The network plugin is configured to assign IP addresses to Pods.
-   The kube-apiserver is configured to assign IP addresses to Services.
-   The kubelet or the cloud-controller-manager is configured to assign IP addresses to Nodes.
---

### Cluster networking types
Kubernetes clusters, attending to the IP families configured, can be categorized into:
-   IPv4 only: The network plugin, kube-apiserver and kubelet/cloud-controller-manager are configured to assign only IPv4 addresses.
-   IPv6 only: The network plugin, kube-apiserver and kubelet/cloud-controller-manager are configured to assign only IPv6 addresses.
-   IPv4/IPv6 or IPv6/IPv4  [dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/):
    -   The network plugin is configured to assign IPv4 and IPv6 addresses.
    -   The kube-apiserver is configured to assign IPv4 and IPv6 addresses.
    -   The kubelet or cloud-controller-manager is configured to assign IPv4 and IPv6 address.
    -   All components must agree on the configured primary IP family.

Kubernetes clusters only consider the IP families present on the Pods, Services and Nodes objects, independently of the existing IPs of the represented objects. Per example, a server or a pod can have multiple IP addresses on its interfaces, but only the IP addresses in  `node.status.addresses`  or  `pod.status.ips`  are considered for implementing the Kubernetes network model and defining the type of the cluster.
## Link tham khảo
https://github.com/shawnsong/kubernetes-handbook/tree/master
https://viblo.asia/s/GJ59jLJaKX2
https://viblo.asia/u/hmquan08011996/series
[Tool triển khai k8s](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
https://blog.sighup.io/how-to-run-kubernetes-without-docker/
https://phoenixnap.com/kb/install-kubernetes-on-ubuntu
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc4NDgzODg3OF19
-->