
# Enable Hyper v cho window home
```powershell
pushd "%~dp0"
dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum >hyper-v.txt
for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
del hyper-v.txt
Dism /online /enable-feature /featurename:Microsoft-Hyper-V -All /LimitAccess /ALL
pause
```
- tạo file hyperv.bat rồi chạy với quyền admin
# Cấu hình ip tĩnh cho ubuntu
```
root@vm:/home/ubuntu# cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses: [172.16.13.100/24]
      gateway4: 172.16.13.1
      nameservers:
          addresses: [8.8.8.8, 8.8.4.4]
      mtu: 1450
  version: 2
```
# Tạo cấu hình virtual switch ubuntu
- `Virtual Switch Manager` => Tạo Interal Virtual Switch
- Vào `Control Panel` => `Network and Internet` => `Network and Sharing Center` => `Change adapter settings`, chuột phải vào NIC tương ứng, chọn `Properties`, tích vào dòng `IPv4`, chọn `Properties`.
- Tại đây chọn Usse the following IP address
	- `IP address`: địa chỉ ip `.1` của dải muốn chọn (là địa chỉ gateway của dải mạng)
	- Điền subnet mask, sau đó chọn oke
	- Tạo máy ảo bằng network này
# Cấu hình NAT
- Cấu hình ip 
```
ip addr add 172.16.13.100/24 dev eth0
ip route add default via 172.16.13.1
```
- power shell với quyền admin (khi máy ảo ping được tới default gateway nhưng chưa ra được internet => window chưa tạo nat cho dải mạng)
```
Get-VMSwitch
Get-VMSwitch -Name "Default Switch"
Restart-Service vmms
# tạo nat cho dải mạng
New-NetNat -Name "HyperVNat" -InternalIPInterfaceAddressPrefix "172.16.13.0/24"
```

# Kiểm tra Windows Firewall
Windows mặc định **block ICMP request**.  
Mở cho phép Ping
```PowerShell
netsh  advfirewall  firewall  add  rule  name="Allow ICMPv4-In"  protocol=icmpv4:8,any  dir=in  action=allow
```
Hoặc bật rule có sẵn
```PowerShell
Get-NetFirewallRule -DisplayName "*File and Printer Sharing (Echo Request - ICMPv4-In)*"
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2NzkzOTUxMzgsMTIwNTk1NDI4OF19
-->