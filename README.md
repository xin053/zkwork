# zkwork_aleo_gpu_worker 使用教程

**备注: 依照本教程如若遇到问题, 可在 `zk.work` 官方 `discord` 群组中 `@xin053`; 或在本仓库提交 `github issues`**

## 环境

`ubuntu-server 22.04`, 理论上其他版本 `ubuntu` 也适用(未测试)

## 使用流程

1. 操作系统初始化(可选)
2. 安装 `cuda` 及驱动
3. 下载并启动锄头

## 操作系统初始化(可选)

该初始化操作仅优化 `ulimit` 与内核参数, 以下参数为常用优化参数, 可根据实际需求进行修改

```bash
# 设置 ulimit
cat >> /etc/security/limits.conf <<EOF
* - nofile 65535
* - nproc 65535
root - nofile 65535
root - nproc 65535
EOF

ulimit -HSn 65535

# 修改内核参数
cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward = 1
net.ipv4.ip_local_port_range = 1024 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_mem = 786432 1048576 1572864
net.ipv4.tcp_rmem = 32768 436600 873200
net.ipv4.tcp_wmem = 8192 436600 873200
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_window_scaling = 0
net.ipv4.tcp_sack = 0
net.core.netdev_max_backlog = 30000
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_syncookies = 0
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
vm.overcommit_memory = 1
vm.max_map_count = 262144
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_time = 1800
net.core.somaxconn = 65535
fs.file-max = 2000000
fs.inotify.max_user_watches = 2000000
fs.inotify.max_user_instances = 1024
net.core.somaxconn = 282144
EOF

sysctl -p
```

## 安装 `cuda` 及驱动(全程不需要重启服务器)

### 卸载老版本 `nvidia` 驱动(仅更新驱动版本时需要卸载)

如果已经安装了驱动, 但是需要更新驱动, 可以先执行下面的命令卸载驱动后再安装新版驱动(不需要重启)

```bash
# 卸载 nvidia 驱动, 有些方式安装的驱动没有 nvidia-uninstall 可执行文件, 这种情况下可以通过 apt 命令卸载
nvidia-uninstall -sq
apt --purge -y remove '*nvidia*' 'libxnvctrl*' || true
```

### 安装驱动

这里提供两种驱动安装的方法:

1. 通过 `apt` 在线安装
2. 通过 `cuda` `.run` 文件安装(推荐, 兼容性更好一些)

#### 通过 `apt` 在线安装

```bash
# apt 换源(可选), 下面是 aliyun 换源命令, 供参考; 其他源可以参考: https://help.mirrors.cernet.edu.cn/ubuntu/   (注意需要选择对应的 ubuntu 版本)
sed -i "s@http.*\.com@https://mirrors.aliyun.com@g" /etc/apt/sources.list /etc/apt/sources.list.d/ubuntu.sources > /dev/null 2>&1 || true

# 安装必要的依赖文件
apt update && apt install -y build-essential linux-headers-"$(uname -r)"

# 禁用休眠, 防止带桌面环境的系统因系统休眠导致无法远程
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# 禁用内核自动更新, ubuntu 默认开启内核自动更新, 但自动更新会导致 nvidia 驱动失效, 需要重新编译, 所以建议关闭内核自动更新
systemctl disable --now unattended-upgrades
sed -i '/^APT::Periodic::Unattended-Upgrade/ s/"1"/"0"/' /etc/apt/apt.conf.d/20auto-upgrades
apt-mark hold linux-image-generic linux-headers-generic

# 卸载 nouveau 内核模块, 因开源的 nouveau 驱动与 nvidia 驱动不兼容, 所以需要卸载并加入内核黑名单
rmmod nouveau > /dev/null 2>&1 || true

cat > /etc/modprobe.d/blacklist-nouveau.conf <<EOF
blacklist nouveau
blacklist lbm-nouveau
options nouveau modeset=0
EOF

update-initramfs -u

# 安装 550 版本 server 驱动, 安装其他版本, 修改包名中的 550 即可
apt install -y nvidia-dkms-550-server nvidia-driver-550-server nvidia-utils-550-server

# 测试
nvidia-smi

# 开启 nvidia 持久化模式(可选)
sed -i s#no-persistence-mode#persistence-mode#g /lib/systemd/system/nvidia-persistenced.service
systemctl daemon-reload
systemctl restart nvidia-persistenced
systemctl enable nvidia-persistenced
```

#### 通过 `cuda` `.run` 文件安装

```bash
# apt 换源(可选), 下面是 aliyun 换源命令, 供参考; 其他源可以参考: https://help.mirrors.cernet.edu.cn/ubuntu/   (注意需要选择对应的 ubuntu 版本)
sed -i "s@http.*\.com@https://mirrors.aliyun.com@g" /etc/apt/sources.list /etc/apt/sources.list.d/ubuntu.sources > /dev/null 2>&1 || true

# 安装必要的依赖文件
apt update && apt install -y build-essential linux-headers-"$(uname -r)"

# 禁用休眠, 防止带桌面环境的系统因系统休眠导致无法远程
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# 禁用内核自动更新, ubuntu 默认开启内核自动更新, 但自动更新会导致 nvidia 驱动失效, 需要重新编译, 所以建议关闭内核自动更新
systemctl disable --now unattended-upgrades
sed -i '/^APT::Periodic::Unattended-Upgrade/ s/"1"/"0"/' /etc/apt/apt.conf.d/20auto-upgrades
apt-mark hold linux-image-generic linux-headers-generic

# 卸载 nouveau 内核模块, 因开源的 nouveau 驱动与 nvidia 驱动不兼容, 所以需要卸载(通过 cuda .run 文件安装方式会自动设置 nouveau 内核黑名单, 不需要手动设置)
rmmod nouveau > /dev/null 2>&1 || true

# 下载 cuda .run文件, 这里使用的 cuda_12.4.1_550.54.15(为了更好的兼容锄头, 建议选择550及以上版本), 其他版本参考: https://developer.nvidia.com/cuda-toolkit-archive
# 这里使用的 cuda 安装包没有没有 nvidia 驱动安装包是因为有些锄头可能会依赖 cuda, 为了避免更换锄头可能需要另外再安装 cuda, 所以建议直接通过 cuda 安装包安装 nvidia 驱动及 cuda toolkit
wget https://developer.download.nvidia.com/compute/cuda/12.4.1/local_installers/cuda_12.4.1_550.54.15_linux.run

# 安装, 如果安装报错, 日志文件为 /var/log/nvidia-installer.log
bash cuda_12.4.1_550.54.15_linux.run --silent --driver --toolkit

# 配置 cuda 环境变量
cat > /etc/profile.d/nvidia_cuda.sh <<'EOF'
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
EOF

# 加载环境变量
source /etc/profile.d/nvidia_cuda.sh

# 测试
nvidia-smi

# 开启 nvidia 持久化模式(可选)
nvidia-smi -pm 1
(crontab -l 2>/dev/null | grep -v "^#" || true; echo "@reboot nvidia-smi -pm 1") | sort - | uniq - | crontab -
```

## 下载并启动锄头

`zkwork` 锄头不同版本对比参考: [https://docs.google.com/spreadsheets/d/1L3kDeoVodZ6_kTzp5WDOvmobObxN8aeAAwT1qPXoTo0/edit?gid=0#gid=0](https://docs.google.com/spreadsheets/d/1L3kDeoVodZ6_kTzp5WDOvmobObxN8aeAAwT1qPXoTo0/edit?gid=0#gid=0), 请自行选择版本, 这里以 `v0.2.1` 版本为例

```bash
cd /opt
# 注意版本, 此处使用的 0.2.1 版本
wget https://github.com/6block/zkwork_aleo_gpu_worker/releases/download/v0.2.1/aleo_prover-v0.2.1.tar.gz
# 如果直接从 github 下载失败, 可以使用如下公益代理或自行通过代理软件下载
wget https://ghp.ci/https://github.com/6block/zkwork_aleo_gpu_worker/releases/download/v0.2.1/aleo_prover-v0.2.1.tar.gz

# 解压
tar zvxf aleo_prover-v0.2.1.tar.gz
```

### 创建 systemd 文件

通过 `systemd` 可以为锄头实现开机自启, 软件异常退出自动重启, 以及可以通过 `systemctl` 命令统一管理等功能

```bash
vim /lib/systemd/system/aleo.service
```

写入以下内容, **注意修改其中的 `钱包地址` 与 ` 矿工名字`; 如果需要指定 `GPU`, 需要在 `ExecStart` 参数末尾添加 `-g 0 -g 1 -g 2` 参数指定 `GPU`**

```ini
[Unit]
Description=aleo prover service
After=network.target
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/aleo_prover
ExecStart=/opt/aleo_prover --pool aleo.hk.zk.work:10003 --address 钱包地址 --custom_name 矿工名字
ExecStop=/usr/bin/pkill -9 -f "aleo"
Restart=always
RestartSec=1
SuccessExitStatus=3 4
RestartForceExitStatus=3 4
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

执行 `reload` 命令

```bash
systemctl daemon-reload
```

### 使用

```bash
# 启动
systemctl start aleo

# 停止
systemctl stop aleo

# 重启
systemctl restart aleo

# 设置开启自启动
systemctl enable aleo

# 查看日志
journalctl -f -u aleo
```

## 其他

### 代理

使用代理的好处:

1. 可以直接下载 `github` 上的文件
2. 让锄头运行过程中走代理, 可以让网络请求包括 `dns` 解析都走代理加密, 这样理论上上游运营商侧通过网络抓包无法检测挖矿, 从而避免被服务提供商或者 `isp` 封堵或清退
3. 当前 `zk.work` 的 `pool` 地址可以直接, 但有可能将来会被墙

缺点:

1. 增加了运维复杂度, 需要自行管理和维护代理
2. 有被请喝茶风险

**当然也可以自行选择购买机场服务, 这里不多介绍了, 请自行了解**

代理协议有很多种, 这里仅以 `trojan` 为例, 请自行斟酌

`trojan` 服务端部署文档参考: [https://github.com/Alvin9999/new-pac/blob/master/%E8%87%AA%E5%BB%BAtrojan%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%95%99%E7%A8%8B.md](https://github.com/Alvin9999/new-pac/blob/master/%E8%87%AA%E5%BB%BAtrojan%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%95%99%E7%A8%8B.md)

`trojan` 客户端部署参考: [https://github.com/p4gefau1t/trojan-go](https://github.com/p4gefau1t/trojan-go)

```bash
# docker-compose 启动 trojan 客户端示例
# 安装 docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun > /dev/null 2>&1

# 创建配置文件
mkdir -p /opt/proxy
cat > /opt/proxy/client.json <<EOF
{
  "run_type": "client",
  "local_addr": "0.0.0.0",
  "local_port": 1080,
  "remote_addr": "修改为trojan服务端ip",
  "remote_port": 443,
  "log_level": 0,
  "password": ["修改为trojan 密码"],
  "ssl": {
    "verify": false,
    "verify_hostname": false
  }
}
EOF

# 这里将 1080 映射到主机 127.0.0.1 上主要是为了安全, 仅允许本机访问; 如果节点很多, 可以监听到内网ip 或者监听到公网并做好 ip 白名单限制
cat > /opt/proxy/docker-compose.yaml <<EOF
services:
  trojan:
    image: p4gefau1t/trojan-go
    container_name: trojan
    hostname: trojan
    restart: unless-stopped
    ports:
      - "127.0.0.1:1080:1080"
    volumes:
      - ./client.json:/opt/client.json
    command:
      - /opt/client.json
EOF

# 启动代理服务
cd /opt/proxy

# 新版本 docker 集成了 compose 命令, 如果是老版本的 docker, 需要自动下载 docker-compose 命令后, 通过 docker-compose up -d 启动
docker compose up -d
```

#### ubuntu 使用代理

这里列出两种方式, 请自行选择：

1. 使用 nsproxy 进行代理

```bash
# 下载
wget -O /usr/local/bin/nsproxy https://github.com/nlzy/nsproxy/releases/download/v0.2.0/nsproxy_x86_64-linux-musl && chmod +x /usr/local/bin/nsproxy

# 使用参考: https://github.com/nlzy/nsproxy
nsproxy -s 127.0.0.1 -p 1080 /opt/aleo_prover --pool aleo.hk.zk.work:10003 --address 钱包地址 --custom_name 矿工名字
```

2. 使用 proxychains 进行代理

```bash
# 安装
apt install -y proxychains
# 修改配置, 这里以 trojan 代理为例, /etc/proxychains.conf 配置文件请根据实际情况修改
sed -i -e "s/^socks.*/socks5  127.0.0.1 1080/g" /etc/proxychains.conf
# 使用示例
proxychains /opt/aleo_prover --pool aleo.hk.zk.work:10003 --address 钱包地址 --custom_name 矿工名字
```

### 监控

`zkwork` 之后的版本会集成监控告警, 这里仅提供思路

```bash
# 安装 jq
apt install -y jq

# 或者 active 状态 work 的数目
curl -k -s "https://zk.work/api/aleo/miner/钱包地址/workerList?page=1&size=10&isActive=true" | jq '.data.total'

# 获取到数目后, 如果数目有问题, 通过 shell 调用邮件, 企业微信或钉钉告警脚本产生告警
```

也可以通过 `python` `requests` 包获取 `active work` 总数目后, 配合 `yagmail` 包发送邮件


### nvidia GPU 常见故障处理

参考: [https://www.volcengine.com/docs/6419/839699](https://www.volcengine.com/docs/6419/839699)

这里主要列出几种常见情况:

1. `rev ff` 掉卡

```bash
# 执行以下命令发现有 gpu 状态为 rev ff 状态; 该状态大部分情况还是卡有问题, 建议联系供应商换卡
lspci | grep -i nvidia
```

2. 执行 `nvidia-smi` 命令卡住或报错

```bash
# 查看 xid, 然后根据文档查看 xid 对应的描述
dmesg | grep -i xid
```

`xid` 官方文档: [https://docs.nvidia.com/deploy/xid-errors/index.html](https://docs.nvidia.com/deploy/xid-errors/index.html)

常见 `xid` 报错: [https://www.chenshaowen.com/blog/common-gpu-operation-and-fault-handling.html](https://www.chenshaowen.com/blog/common-gpu-operation-and-fault-handling.html)

3. `GPU` 温度过高

```bash
nvidia-smi --query-gpu=index,temperature.gpu --format=csv,noheader
```

4. 其他排查问题思路

a. 查看 `cpu`, 内存, 磁盘`io`, 网络`io` 资源占用情况
b. 查看 `BMC` 健康日志

### 自动化

可以通过 `ansible` 剧本实现上述操作的批量自动化配置, 控制锄头安装, 启停, 一键更新锄头版本以及更换锄头等; 这里仅提供思路, 请自行按照需求进行开发

1. `prepare.yml` 剧本在部署节点上下载锄头
2. `install.yml` 剧本在 `work` 节点上安装驱动及代理
3. `run.yml` 剧本在 `work` 节点上从部署节点 `scp` 拷贝锄头并创建 `systemd` 文件启动锄头
4. `stop.yml` 剧本调用 `systemctl stop` 命令停止锄头
5. `check.yml` 剧本检测 `gpu` 利用率, 平均利用率高于 `90%` 以上表示锄头正常工作
