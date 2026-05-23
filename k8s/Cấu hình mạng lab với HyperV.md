# Cấu hình mạng lab HyperV
Dưới đây là cách setup hoàn chỉnh:

* Hyper-V Internal Switch
* NAT internet
* static IP cho Debian VM
* phù hợp để học Kubernetes/CKA

Mô hình sẽ như này:

```text id="o2c6y9"
Internet
    ↓
Windows Host
    ↓ NAT
vEthernet (k8s-int)
192.168.100.1
    ↓
-------------------------
|       Hyper-V         |
-------------------------
    ↓
Debian VM
192.168.100.10
192.168.100.11
192.168.100.12
```

---

# PHẦN 1 — Tạo Internal Switch

## Mở Hyper-V Manager

```text id="a7q40m"
Hyper-V Manager
→ Virtual Switch Manager
```

---

## Tạo switch mới

Chọn:

```text id="w8j95g"
New virtual network switch
→ Internal
```

Đặt tên:

```text id="0b9b9k"
k8s-int
```
Ấn:
```text id="xqj39x"
Apply
```
---

# PHẦN 2 — Kiểm tra adapter trên Windows

Mở PowerShell Admin:

```powershell id="x7l0g9"
Get-NetAdapter
```

Bạn sẽ thấy:

```powershell
(base) PS C:\WINDOWS\system32> Get-NetAdapter

Name                      InterfaceDescription                    ifIndex Status       MacAddress             LinkSpeed
----                      --------------------                    ------- ------       ----------             ---------
vEthernet (Default Swi... Hyper-V Virtual Ethernet Adapter             24 Up           00-15-5D-5D-FF-F1        10 Gbps
Bluetooth Network Conn... Bluetooth Device (Personal Area Netw...      22 Disconnected E4-C7-67-6C-A0-6F         3 Mbps
vEthernet (k8s-int)       Hyper-V Virtual Ethernet Adapter #4          59 Up           00-15-5D-B5-83-05        10 Gbps
vEthernet (ubuntu)        Hyper-V Virtual Ethernet Adapter #3          11 Up           00-15-5D-B5-83-03        10 Gbps
vEthernet (WSL (Hyper-... Hyper-V Virtual Ethernet Adapter #2          53 Up           00-15-5D-7C-6B-97        10 Gbps
Wi-Fi                     Intel(R) Wi-Fi 6E AX211 160MHz                5 Up           E4-C7-67-6C-A0-6B       432 Mbps


(base) PS C:\WINDOWS\system32>
```

---

# PHẦN 3 — Gán IP cho switch

Đây sẽ là gateway cho VM.

## Set IP:

```powershell id="1q06eh"
New-NetIPAddress `
  -IPAddress 192.168.100.1 `
  -PrefixLength 24 `
  -InterfaceAlias "vEthernet (k8s-int)"
```

---

# Verify

```powershell id="9v08l2"
Get-NetIPAddress -InterfaceAlias "vEthernet (k8s-int)"
```

Bạn sẽ thấy:

```text id="3v7j43"
192.168.100.1
```

Hoăc
```powershell
ipconfig
```
```powershell
Ethernet adapter vEthernet (k8s-int):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::8e0b:1a6c:66fd:a131%59
   IPv4 Address. . . . . . . . . . . : 192.168.100.1
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :
```

---

# PHẦN 4 — Tạo NAT

Đây là bước cho VM ra internet.

## Tạo NAT

```powershell id="0t6e2t"
New-NetNat `
  -Name "k8s-nat" `
  -InternalIPInterfaceAddressPrefix 192.168.100.0/24
```

---

# Verify NAT

```powershell id="6vl0r7"
Get-NetNat
```

Bạn sẽ thấy:

```text id="jfrx56"
Name: k8s-nat
InternalIPInterfaceAddressPrefix: 192.168.100.0/24
```

---

# PHẦN 5 — Attach switch vào VM

Trong Hyper-V:

```text id="2c4zbp"
VM Settings
→ Network Adapter
→ Virtual Switch
→ chọn k8s-int
```

Làm cho:

* master
* worker1
* worker2

---

# PHẦN 6 — Config static IP cho Debian

# MASTER NODE

## Xem tên NIC

Trong Debian:

```bash id="hktn5n"
ip a
```

Ví dụ thấy:

```text id="jvwz0n"
eth0
```

---

## Edit network config

```bash id="u90g3u"
sudo nano /etc/network/interfaces
```

Nội dung:

```text id="yoj46y"
auto eth0
iface eth0 inet static
    address 192.168.100.10
    netmask 255.255.255.0
    gateway 192.168.100.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

---

# WORKER 1

```text id="qjlwmr"
auto eth0
iface eth0 inet static
    address 192.168.100.11
    netmask 255.255.255.0
    gateway 192.168.100.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

---

# WORKER 2

```text id="63n1iq"
auto eth0
iface eth0 inet static
    address 192.168.100.12
    netmask 255.255.255.0
    gateway 192.168.100.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

---

# Restart networking

```bash id="6db4a9"
sudo systemctl restart networking
```

hoặc reboot VM.

---


# PHẦN 7 — Troubleshooting

# Nếu VM không ra internet

## Check NAT

Windows:

```powershell id="xjlwm1"
Get-NetNat
```

---

# Check IP forwarding

Windows thường tự bật.

---

# Check firewall

Có thể Windows Firewall block.

Tạm test:

```powershell id="1bx9yy"
Set-NetFirewallProfile `
  -Profile Domain,Public,Private `
  -Enabled False
```

Sau khi test xong nên bật lại.

---

# Nếu ping host được nhưng không ping internet

Thường do:

* chưa tạo NAT
* gateway sai
* DNS sai

---

# Kết quả cuối cùng

| Device       | IP             |
| ------------ | -------------- |
| Windows Host | 192.168.100.1  |
| master       | 192.168.100.10 |
| worker1      | 192.168.100.11 |
| worker2      | 192.168.100.12 |

Đây là setup rất chuẩn cho:

* kubeadm
* CKA lab
* Cilium
* Flannel
* Calico
* MetalLB lab sau này.

---
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1MjU3MDc3NTgsNDUzMjU5NjE0XX0=
-->