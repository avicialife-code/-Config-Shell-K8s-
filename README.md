#!/bin/bash
# ============================================================
# 文件名: init.sh
# 执行位置: 全部9台主机（每台修改 HOSTNAME）
# 功能: 基础环境配置（DNS、swap、工具、时间、内核）
# ============================================================

# ========== 唯一需要手动修改的一行 ==========
HOSTNAME="KubernetesBase"        # 按主机规划表修改
# =============================================

echo "=============================================="
echo "  基础初始化: $HOSTNAME"
echo "=============================================="

# 1. 设置主机名
hostnamectl set-hostname $HOSTNAME
echo "主机名已设置为 $HOSTNAME"

# 2. 配置 DNS 解析（锁定防止被覆盖）
cat > /etc/resolv.conf << 'EOF'
nameserver 223.5.5.5
nameserver 8.8.8.8
nameserver 114.114.114.114
EOF
chattr +i /etc/resolv.conf 2>/dev/null || true
echo "DNS 已配置并锁定"

# 3. 关闭 swap
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
echo "swap 已禁用"

# 4. 安装基础工具
yum install -y vim net-tools wget curl lsof telnet bash-completion \
    openssh-clients rsync chrony tcpdump bind-utils jq tree httpd-tools git
echo "基础工具安装完成"

# 5. 时间同步
systemctl enable --now chronyd
chronyc sources -v
echo "时间同步已启用"

# 6. 生成 SSH 密钥
if [ ! -f /root/.ssh/id_rsa ]; then
    ssh-keygen -t rsa -b 2048 -N "" -f /root/.ssh/id_rsa
    echo "SSH 密钥已生成"
fi

# 7. 加载内核模块
modprobe br_netfilter
modprobe overlay
cat > /etc/modules-load.d/k8s.conf << 'EOF'
br_netfilter
overlay
EOF

# 8. 配置 sysctl
cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sysctl --system

mkdir -p /root/.ssh /data /opt /var/log/kubernetes

echo "=============================================="
echo "  $HOSTNAME 基础初始化完成"
echo "  请执行: exec bash 刷新主机名"
echo "=============================================="
