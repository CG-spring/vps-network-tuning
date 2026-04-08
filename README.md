# VPS 网络调优指南

> TCP/BBR 优化、路由选择和延迟改善

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/CG-spring/vps-network-tuning)](https://github.com/CG-spring/vps-network-tuning)

**[English](README_EN.md)**

---

## 📋 目录

- [快速开始](#快速开始)
- [BBR 安装](#bbr-安装)
- [TCP 优化](#tcp-优化)
- [路由优化](#路由优化)
- [测速工具](#测速工具)
- [常见问题](#常见问题)

---

## 🚀 快速开始

一键安装 BBR：

```bash
wget -qO- https://raw.githubusercontent.com/CG-spring/vps-network-tuning/main/install-bbr.sh | bash
```

或手动安装：

```bash
# 查看当前 TCP 拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

# 启用 BBR（需要内核 4.9+）
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

---

## 🔧 BBR 安装

### 方法 1：官方 BBR（内核 4.9+）

```bash
# 添加到 /etc/sysctl.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# 应用配置
sysctl -p

# 验证
sysctl net.ipv4.tcp_congestion_control
# 输出：net.ipv4.tcp_congestion_control = bbr
```

### 方法 2：BBR Plus（高延迟网络更好）

```bash
# 下载并安装 BBR Plus 内核
wget https://github.com/cx9208/bbrplus/raw/master/ok_bbrplus_centos.sh
chmod +x ok_bbrplus_centos.sh
./ok_bbrplus_centos.sh

# 重启后启用
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbrplus" >> /etc/sysctl.conf
sysctl -p
```

### 方法 3：LotServer（商业版，性能最佳）

```bash
# 一键安装
wget --no-check-certificate -qO /tmp/appex.sh "https://raw.githubusercontent.com/0oVicero0/serverSpeeder_Install/master/appex.sh" && bash /tmp/appex.sh
```

---

## ⚙️ TCP 优化配置

### 推荐 sysctl.conf

```bash
# 文件：/etc/sysctl.conf

# TCP 内存设置
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# 连接跟踪
net.netfilter.nf_conntrack_max = 655350
net.netfilter.nf_conntrack_tcp_timeout_established = 1200

# TCP 优化
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_mtu_probing = 1

# 启用 BBR
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

应用配置：`sysctl -p`

---

## 🌐 路由优化

### 测试当前路由

```bash
# 安装 besttrace
wget https://github.com/zq/besttrace/releases/download/v1.3/besttrace-linux-amd64 -O besttrace
chmod +x besttrace

# 测试到电信
./besttrace -q 1 -g cn 202.96.209.133

# 测试到联通
./besttrace -q 1 -g cn 221.12.1.227

# 测试到移动
./besttrace -q 1 -g cn 211.136.112.50
```

### 常见优化方案

| 方案 | 适用场景 | 难度 |
|------|----------|------|
| BGP 优化路由 | 回国最佳 | 简单（购买） |
| CN2 GIA | 高端回国线路 | 简单（购买） |
| 9929/10099 | 性价比回国 | 简单（购买） |
| BGP 劫持 | 高级用户 | 困难 |

---

## 📊 测速工具

### 带宽测试

```bash
# 安装 speedtest-cli
apt install -y speedtest-cli

# 运行测试
speedtest-cli
```

### 综合性能测试

```bash
# 使用 bench.sh
wget -qO- bench.sh | bash

# 或使用更全面的脚本
wget -qO- https://raw.githubusercontent.com/teddysun/across/master/bench.sh | bash
```

### 回国延迟测试

```bash
# 安装 pingtest
pip install ping3

# 测试脚本
python3 << 'EOF'
from ping3 import ping
import statistics

hosts = {
    '北京电信': '202.96.209.133',
    '上海电信': '202.96.209.5',
    '广州电信': '202.96.128.86',
    '北京联通': '221.12.1.227',
    '上海联通': '221.12.33.227',
}

for name, ip in hosts.items():
    times = [ping(ip, timeout=2) for _ in range(5) if ping(ip, timeout=2)]
    if times:
        avg = statistics.mean(times) * 1000
        print(f"{name}: {avg:.1f}ms")
    else:
        print(f"{name}: 超时")
EOF
```

---

## ❓ 常见问题

**Q: 应该使用哪个版本的 BBR？**
A: 先尝试官方 BBR。如果延迟高（>150ms）或有丢包，再试 BBR Plus。

**Q: BBR 会提升我的网速吗？**
A: BBR 在高延迟或丢包网络中提升明显。在干净、低延迟网络中改善有限。

**Q: BBR 安全吗？**
A: 安全。BBR 由 Google 开发，已集成到 Linux 内核 4.9+。

**Q: 为什么我的 VPS 回国慢？**
A: 常见原因：1) 路由差，2) 未做 TCP 优化，3) 超售严重，4) 网络拥塞。

---

## 🔗 相关资源

- [VPSVIP](https://vpsvip.net) - VPS 推荐
- [ClashVIP](https://clashvip.net) - Clash 教程
- [ClashHub](https://clashhub.net) - 规则与配置

---

## 📄 许可证

MIT 许可证 - 详见 [LICENSE](LICENSE)

---

**最后更新**: 2026-04-08
