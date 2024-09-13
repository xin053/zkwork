# zkwork_aleo_gpu_worker 使用教程

## 环境

`ubuntu-server 22.04`

## 使用流程

1. 操作系统初始化(可选)
2. 安装 `cuda` 及驱动
3. 启动锄头

## 初始化

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

## 安装驱动

```bash
apt update && apt install -y build-essential linux-headers-"$(uname -r)"

# 禁用休眠
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# 禁用内核自动更新
systemctl disable --now unattended-upgrades
sed -i '/^APT::Periodic::Unattended-Upgrade/ s/"1"/"0"/' /etc/apt/apt.conf.d/20auto-upgrades
apt-mark hold linux-image-generic linux-headers-generic

# 卸载 nouveau 内核模块
rmmod nouveau > /dev/null 2>&1 || true

# 下载 cuda .run文件, 这里使用的 cuda_12.4.1_550.54.15, 其他版本参考: https://developer.nvidia.com/cuda-toolkit-archive
wget https://developer.download.nvidia.com/compute/cuda/12.4.1/local_installers/cuda_12.4.1_550.54.15_linux.run

# 安装
bash cuda_12.4.1_550.54.15_linux.run --silent --driver --toolkit

# 配置环境变量
cat > /etc/profile.d/nvidia_cuda.sh <<'EOF'
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
EOF

# 加载环境变量
source /etc/profile.d/nvidia_cuda.sh

# 测试
nvidia-smi
```

## 下载并启动锄头

```bash
cd /opt
# 注意版本, 此处使用的 0.1.1 版本
wget https://github.com/6block/zkwork_aleo_gpu_worker/releases/download/v0.1.1/aleo_prover-v0.1.1.tar.gz

# 解压
tar zvxf aleo_prover-v0.1.1.tar.gz
```

### 创建 systemd 文件

```bash
vim /lib/systemd/system/aleo.service
```

写入以下内容, **注意修改其中的 `钱包地址` 与 ` 矿工名字`**

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
ExecStart=/opt/aleo_prover --network Mainnet --pool aleo.hk.zk.work:10003 --address 钱包地址 --custom_name 矿工名字
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

可以通过 `ansible` 剧本实现批量控制锄头安装, 启停
