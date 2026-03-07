# Mở CMD admin từ PowerShell
```powershell
Start-Process  cmd  -Verb  runAs
Start-Process  powershell  -Verb  runAs
```
# Quản lý process bằng Powershell admin
1.  Xem tất cả process đang chạy
```powershell
Get-Process
```
2. Tìm process theo tên, theo PID
```powershell
Get-Process  chrome
Get-Process -Id 4567
```
3. Kill process theo PID
```powershell
Stop-Process  -Id  4567  -Force
```
4. Kill nhiều process cùng lúc
```powershell
Get-Process  python  |  Stop-Process  -Force
```
5. Xem process tree
```powershell
Get-CimInstance Win32_Process | Select ProcessId,ParentProcessId,Name
```
6. kill toàn bộ process chiếm port 8000
```powershell
Stop-Process  -Id (Get-NetTCPConnection  -LocalPort  8000).OwningProcess  -Force
```
7. Restart File Explorer
```powershell
Stop-Process -Name explorer -Force; Start-Process explorer
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1ODYxNDE0ODUsMTcxNjE4NTk0NF19
-->