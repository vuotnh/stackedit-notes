# Cấu hình server Debian
## 1. Cập nhật phần mềm
Chỉnh sửa file `/etc/apt/sources.list` với nội dung sau
```bash
deb http://deb.debian.org/debian trixie main contrib non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free-firmware
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE3NDkwOTQ0NF19
-->