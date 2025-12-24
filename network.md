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
