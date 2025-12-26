# 静态ip的配置
在命令行查看wsl的ip地址比如
```
以太网适配器 vEthernet (WSL):

   连接特定的 DNS 后缀 . . . . . . . :
   本地链接 IPv6 地址. . . . . . . . : fe80::4385:6527:e76f:bb1b%32
   IPv4 地址 . . . . . . . . . . . . : 172.23.192.1
   子网掩码  . . . . . . . . . . . . : 255.255.240.0
   默认网关. . . . . . . . . . . . . :
```
NAT 模式下 WSL 重启后 IP 会变，想固定 IP 可以通过脚本实现，步骤如下：  
在 WSL 中创建静态 IP 脚本  
```bash
sudo nano /etc/wsl-static-ip.sh
```
粘贴以下内容（IP 选 172.18.0.100，避免和默认分配的 IP 冲突）：
```bash
#!/bin/bash
sudo ip addr flush dev eth0
sudo ip addr add 172.23.192.100/20 broadcast 172.23.207.255 dev eth0
sudo ip route add default via 172.23.192.1 dev eth0 # 与wsl适配器的ip地址相同
sudo chattr -i /etc/resolv.conf
sudo rm -rf /etc/resolv.conf
echo "nameserver 223.6.6.6" | sudo tee /etc/resolv.conf > /dev/null
echo "nameserver 119.29.29.29" | sudo tee -a /etc/resolv.conf > /dev/null
sudo chattr +i /etc/resolv.conf
```
给脚本添加执行权限
```bash
sudo chmod +x /etc/wsl-static-ip.sh
```
设置 WSL 启动时自动执行脚本
```bash
sudo nano /etc/profile.d/wsl-init.sh
```
```bash
# 启动时执行静态 IP 脚本
/etc/wsl-static-ip.sh
```
保存退出，给这个文件也加执行权限：
sudo chmod +x /etc/profile.d/wsl-init.sh
验证静态 IP
重启 WSL 后，执行 ip addr show eth0，如果显示 172.18.0.100 就说明配置成功。
# 一键配置静态ip，但需要仔细修改内容
```
sudo bash -c '
cat > /etc/wsl-static-ip.sh << EOF
#!/bin/bash
sudo ip addr flush dev eth0
sudo ip addr add 172.23.192.100/20 broadcast 172.23.207.255 dev eth0
sudo ip route add default via 172.23.192.1 dev eth0 # 与wsl适配器的ip地址相同
sudo chattr -i /etc/resolv.conf
sudo rm -rf /etc/resolv.conf
echo "nameserver 223.6.6.6" | sudo tee /etc/resolv.conf > /dev/null
echo "nameserver 119.29.29.29" | sudo tee -a /etc/resolv.conf > /dev/null
sudo chattr +i /etc/resolv.conf
EOF
chmod +x /etc/wsl-static-ip.sh
cat >> /etc/wsl.conf << EOF
[boot]
command = "/usr/local/bin/set-wsl-static-ip.sh"  # WSL启动时以root执行该脚本

[network]
generateResolvConf = false  # 禁止自动生成resolv.conf，保留自定义DNS
EOF
'
```
# windows的端口转发
使用Portproxy模式下的Netsh命令即能实现Windows系统中的端口转发，转发命令如下:
```
netsh interface portproxy add v4tov4 listenaddress=[localaddress] listenport=[localport] connectaddress=[destaddress]
```
解释一下这其中的参数意义
```
listenaddress – 等待连接的本地ip地址
listenport – 本地监听的TCP端口（待转发）
connectaddress – 被转发端口的本地或者远程主机的ip地址
connectport – 被转发的端口
```
举个例子，服务器内网IP是172.16.0.4，需要将8080端口转发到国外服务器104.104.104.104的9999端口，那么命令如下：
```
netsh interface portproxy add v4tov4  listenaddress=172.16.0.4 listenport=8080 connectaddress=104.104.104.104 connectport=9999
```
下面的命令是用来展示系统中的所有转发规则：
```
netsh interface  portproxy show  v4tov4
```
删除刚才创建的那个转发的命令：
```
netsh interface  portproxy delete v4tov4 listenaddress=172.16.0.4 listenport=8080
```
注意：连接时请确保防火墙（Windows防火墙或者其他的第三方防护软件）允许外部连接到一个全新的端口，如果不允许，那么只能自行添加一个新的Windows防火墙规则。

该命令的常用参数如下：
```
netstat -ano | find listenport 查看是否启动成功
netsh interface portproxy show all 显示系统中的转发规则列表
netsh interface portproxy dump 查看portproxy设置
netsh interface portproxy delete v4tov4 listenport=localport listenaddress=localaddress
netsh interface portproxy reset 清除所有端口转发规则
```
