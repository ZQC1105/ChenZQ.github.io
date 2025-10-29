## 查找占用 8080 端口的进程 PID

netstat -ano | grep :8080

## 关闭

$ powershell -command "Stop-Process -Id 12092 -Force"
