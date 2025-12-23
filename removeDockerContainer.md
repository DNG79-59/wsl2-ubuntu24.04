# 总体来说最建议的是，通过构建镜像，打包数据来迁移
## 步骤 1：查看待迁移容器的核心信息（先确认关键参数）
先查容器的镜像、挂载目录、端口等，避免迁移后配置丢失：
```bash
# 1. 查看容器基本信息（确认容器名/ID、状态）
docker ps -a | grep sub-store  # 假设容器名是sub-store
```
```# 2. 查看容器详细配置（重点找「挂载目录」和「环境变量」）
docker inspect sub-store | grep -E "Mounts|Env|Ports" -A 10
```
关键输出示例（你的配置）：
```
"Mounts": [{"Source": "/data/sub-store", "Destination": "/opt/app/data"}],  # 数据挂载目录
"Env": ["SUB_STORE_CRON=55 23 * * *", "SUB_STORE_FRONTEND_BACKEND_PATH=/xxx"],  # 环境变量
"Ports": {"3001/tcp": [{"HostPort": "3001"}]}  # 端口映射
```
## 步骤 2：导出容器为可迁移的镜像文件
容器是「运行时实例」，无法直接迁移，需先打包成镜像（保留容器内所有修改）：
```bash
# 1. （可选）停止容器（打包镜像更稳定，非必须）
docker stop sub-store
```
```
# 2. 将容器打包为新镜像（格式：docker commit 容器名/ID 新镜像名:标签）
docker commit sub-store sub-store-migrate:v1  # 自定义镜像名+标签，方便识别
```
```
# 3. 验证镜像是否创建成功
docker images | grep sub-store-migrate  # 能看到镜像列表即成功
```
# 4. 导出镜像为压缩文件（方便跨主机传输）
```
docker save sub-store-migrate:v1 | gzip > /tmp/sub-store-image.tar.gz
```
说明：/tmp 是临时目录，也可改为你指定的路径（如 ~/sub-store-image.tar.gz）  
## 步骤 3：定位并备份容器的持久化数据  
容器的核心数据在「挂载目录」（步骤 1 查到的 Source: /etc/sub-store），只需备份这个目录：
```bash
# 1. 确认数据目录的文件（验证数据存在）
ls -l /data/sub-store  # 能看到容器的配置/日志/订阅数据即正常
```
```
# 2. 打包数据目录为压缩文件（保留完整路径）
tar -zcvf /tmp/sub-store-data.tar.gz /etc/sub-store
```
参数说明：  
-z：用gzip压缩；-c：创建压缩包；-v：显示打包过程；-f：指定文件名  
/etc/sub-store：要备份的目录（步骤1查到的挂载源目录）  
## 步骤 4：确认原主机的迁移文件
此时原主机 /tmp 目录下会有两个关键文件：  
```bash
ls /tmp | grep sub-store
```
输出：
```
sub-store-image.tar.gz  # 镜像压缩包
sub-store-data.tar.gz   # 数据压缩包
```
# 二、传输迁移文件到目标主机（B）
将原主机的镜像包 + 数据包传到目标主机，推荐 2 种方式：
方式 1：scp 直传（推荐，跨主机网络互通时用）
```bash
#在原主机执行（格式：scp 本地文件 目标主机用户名@目标主机IP:目标路径）
scp /tmp/sub-store-image.tar.gz 目标主机用户名@目标主机IP:/home/目标主机用户名/
scp /tmp/sub-store-data.tar.gz 目标主机用户名@目标主机IP:/home/目标主机用户名/
# 示例：scp /tmp/sub-store-image.tar.gz yandf@192.168.1.100:/home/yandf/
# 输入目标主机密码即可传输
```
方式 2：U 盘 / 网盘传输（无网络时用）  
把原主机 /tmp 下的两个压缩包拷贝到 U 盘；  
将 U 盘插到目标主机，把压缩包拷贝到目标主机的 /tmp 目录（或任意目录）。  
# 三、在目标主机（B）操作：导入镜像 + 恢复数据 + 重建容器
## 步骤 1：准备目标主机环境
确保目标主机已安装 Docker（未安装的话，参考之前的 Docker 安装脚本）：
```bash
# 验证Docker是否安装并运行
docker --version && systemctl status docker
```
若未启动，执行：
```
sudo systemctl start docker && sudo systemctl enable docker
```
## 步骤 2：导入镜像文件到目标主机
```bash
# 1. 进入迁移文件所在目录（假设文件传到了 /home/目标主机用户名/）
cd /home/目标主机用户名/

# 2. 导入镜像（格式：gzip -dc 镜像压缩包 | docker load）
gzip -dc sub-store-image.tar.gz | docker load

# 3. 验证镜像是否导入成功
docker images | grep sub-store-migrate  # 能看到镜像列表即成功
```
## 步骤 3：恢复容器的持久化数据
```bash
# 1. 创建数据挂载目录（和原主机保持一致，避免权限问题）
sudo mkdir -p /data/sub-store  # -p：自动创建多级目录

# 2. 解压数据压缩包（保留原路径，关键！）
sudo tar -zxvf sub-store-data.tar.gz -C /
# 参数说明：-C / 表示解压到根目录，保证 /etc/sub-store 路径和原主机一致
# 若提示「权限不足」，加 sudo 执行

# 3. 验证数据是否恢复成功
ls -l /etc/sub-store  # 能看到和原主机一致的文件即成功
```
## 步骤 4：重建容器（和原主机配置完全一致） 
用导入的镜像 + 恢复的数据，重建和原主机一样的容器（复制原主机的启动命令，仅镜像名改为导入的镜像）：  
```bash
docker run -it -d \
--restart=always \
-e "SUB_STORE_CRON=55 23 * * *" \
-e SUB_STORE_FRONTEND_BACKEND_PATH=/bYJyUABXrGClOt6qUFfM \
-p 3001:3001 \
-v /etc/sub-store:/opt/app/data \
--name sub-store \
sub-store-migrate:v1  # 改为导入的镜像名:标签
```
# 最后就是验证是否迁移成功，和删除原主机上的临时文件
```
# 1. 查看容器状态（STATUS为Up即运行中）
docker ps | grep sub-store

# 2. 查看容器日志（无报错即正常）
docker logs sub-store

# 3. 验证服务功能（访问容器服务）
curl http://localhost:3001/bYJyUABXrGClOt6qUFfM
# 能返回Sub-Store的页面/数据，说明迁移成功

# 4. 验证数据一致性（对比原主机的配置文件）
cat /etc/sub-store/config.json  # 和原主机的同文件内容一致即数据恢复成功
```
