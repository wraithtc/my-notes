PS D:\> powershell -ExecutionPolicy Bypass -File "\\wsl.localhost\Ubuntu-24.04\home\tongchao\dev\task\aipet\pet-id\scripts\setup-wsl-lan-portproxy.ps1"
WSL distro: Ubuntu-24.04
WSL IP: 172.20.166.1

Portproxy rules:

侦听 ipv4:                 连接到 ipv4:

地址            端口        地址            端口
--------------- ----------  --------------- ----------
0.0.0.0         8000        172.20.166.1    8000
0.0.0.0         4173        172.20.166.1    4173


LAN URLs:
API:      http://192.168.10.104:8000/v1/health
Frontend: http://192.168.10.104:4173