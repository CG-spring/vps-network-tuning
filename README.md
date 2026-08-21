# VPS 网络调优完全指南

> 从内核参数到路由策略，系统性提升 VPS 跨境网络性能

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/CG-spring/vps-network-tuning)](https://github.com/CG-spring/vps-network-tuning)
[![Kernel](https://img.shields.io/badge/kernel-4.9%2B-blue.svg)](https://www.kernel.org)
[![BBR](https://img.shields.io/badge/BBR-Ready-brightgreen.svg)](https://github.com/google/bbr)

**[English](README_EN.md)** | **[中文](README.md)**

---

## 📋 目录

- [一、概述与原理](#一概述与原理)
  - [1.1 为什么 VPS 网络需要调优](#11-为什么-vps-网络需要调优)
  - [1.2 TCP 拥塞控制算法对比](#12-tcp-拥塞控制算法对比)
  - [1.3 BBR 的工作原理](#13-bbr-的工作原理)
- [二、快速开始](#二快速开始)
- [三、BBR 安装详解](#三bbr-安装详解)
  - [3.1 官方 BBR（推荐）](#31-官方-bbr推荐)
  - [3.2 BBR Plus](#32-bbr-plus)
  - [3.3 LotServer 锐速](#33-lotserver-锐速)
  - [3.4 内核升级（若需要）](#34-内核升级若需要)
- [四、TCP 参数深度调优](#四tcp-参数深度调优)
  - [4.1 缓冲区与内存配置](#41-缓冲区与内存配置)
  - [4.2 连接跟踪优化](#42-连接跟踪优化)
  - [4.3 TCP 时间戳与快速打开](#43-tcp-时间戳与快速打开)
  - [4.4 TIME_WAIT 状态优化](#44-time_wait-状态优化)
  - [4.5 完整 sysctl.conf 参考](#45-完整-sysctlconf-参考)
- [五、路由优化实战](#五路由优化实战)
  - [5.1 识别当前路由问题](#51-识别当前路由问题)
  - [5.2 线路类型详解](#52-线路类型详解)
  - [5.3 路由测试方法](#53-路由测试方法)
  - [5.4 路由策略建议](#54-路由策略建议)
- [六、网络监控与测速](#六网络监控与测速)
  - [6.1 综合性能测试](#61-综合性能测试)
  - [6.2 回国延迟测试](#62-回国延迟测试)
  - [6.3 带宽压测](#63-带宽压测)
  - [6.4 实时监控工具](#64-实时监控工具)
- [七、故障排查](#七故障排查)
  - [7.1 BBR 未生效](#71-bbr-未生效)
  - [7.2 网速无明显提升](#72-网速无明显提升)
  - [7.3 连接频繁断开](#73-连接频繁断开)
- [八、效果对比与验证](#八效果对比与验证)
- [九、常见问题](#九常见问题)
- [十、相关资源](#十相关资源)

---

## 一、概述与原理

### 1.1 为什么 VPS 网络需要调优

租用海外 VPS 后，很多用户会发现：明明带宽标称 1Gbps，访问海外网站却只有几 Mbps；延迟不高但下载速度异常慢。这些问题的根源通常不在带宽本身，而在 **TCP 拥塞控制机制与网络路径的匹配程度**。

默认情况下，Linux 使用 CUBIC 或 Reno 拥塞控制算法。这些算法设计于有线网络时代，对丢包极为敏感——一旦检测到丢包，立即将发送窗口减半。在当今复杂的跨境网络环境中，海光缆故障、跨国路由拥塞、运营商 QoS 等都会造成"隐性丢包"（数据包在某个节点被悄悄丢弃，TCP 以为网络畅通而持续发包），导致 CUBIC 性能严重下降。

**调优的核心目标**：让 TCP 拥塞控制算法更准确地感知真实网络容量，在充分利用带宽的同时避免因错误判断而自我限速。

### 1.2 TCP 拥塞控制算法对比

| 算法 | 开发方 | 适用场景 | 内核要求 | 稳定性 |
|------|--------|----------|----------|--------|
| **CUBIC**（默认） | Linux 内核 | 低丢包有线网络 | 2.6.19+ | ⭐⭐⭐⭐⭐ |
| **BBR** | Google | 高延迟/丢包跨境网络 | 4.9+ | ⭐⭐⭐⭐ |
| **BBR Plus** | 社区改进 | 回国高延迟线路 | 4.14+ | ⭐⭐⭐ |
| **LotServer** | @oxxt | 极致优化 | 任意 | ⭐⭐⭐ |
| **Westwood** | Linux 内核 | 无线/高抖动网络 | 任意 | ⭐⭐⭐⭐ |

**结论**：海外 VPS 跨境使用，优先尝试 BBR；回国线路若延迟 >150ms 或丢包明显，尝试 BBR Plus；极端情况考虑 LotServer。

### 1.3 BBR 的工作原理

BBR（Bottleneck Bandwidth and Round-trip propagation time）是 Google 于 2016 年发布的拥塞控制算法，与传统算法有本质区别：

**传统算法（丢包驱动）**：
```
发送速率 ↔ 丢包事件
丢包 → 减半窗口 → 慢慢恢复
```

**BBR（模型驱动）**：
```
发送速率 ↔ 带宽估算 + RTT 估算
不依赖丢包，而是主动测量瓶颈带宽和往返延迟
```

BBR 的核心逻辑是：维持一个 pacing rate，始终等于 `min(瓶颈带宽, 发送窗口) / RTT`，这样即使网络轻微丢包，只要 RTT 没有大幅增加，BBR 就会继续按最大带宽发送，不会自我限速。

**BBR 的最佳使用条件**：
- 带宽 > 10Mbps
- 单向延迟 > 20ms（高延迟网络效果更明显）
- 存在轻微丢包或网络波动

---

## 二、快速开始

不想了解细节？一行命令启用 BBR：

```bash
wget -qO- https://raw.githubusercontent.com/CG-spring/vps-network-tuning/main/install-bbr.sh | bash
```

手动操作（3 分钟完成）：

```bash
# 1. 查看当前使用的拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

# 2. 查看当前内核版本（BBR 需要 4.9+）
uname -r

# 3. 启用 BBR（将以下两行追加到 /etc/sysctl.conf）
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

# 4. 应用配置
sysctl -p

# 5. 验证 BBR 已加载
sysctl net.ipv4.tcp_congestion_control
# 期望输出：net.ipv4.tcp_congestion_control = bbr

# 6. 确认 qdisc 也生效了
sysctl net.core.default_qdisc
# 期望输出：net.core.default_qdisc = fq
```

> ⚠️ **注意**：BBR 需内核 4.9+。如果 `uname -r` 显示内核版本低于 4.9，需先升级内核（见 3.4 节）。

---

## 三、BBR 安装详解

### 3.1 官方 BBR（推荐）

官方 BBR 已随 Linux 内核 4.9+ 内置，无需额外安装，只需通过 sysctl 启用：

```bash
# 完整启用流程
cat >> /etc/sysctl.conf << 'EOF'
# BBR 拥塞控制
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF

sysctl -p

# 四重验证
echo "=== 验证 BBR 模块已加载 ==="
lsmod | grep bbr

echo "=== 验证拥塞控制算法 ==="
sysctl net.ipv4.tcp_congestion_control

echo "=== 验证 qdisc ==="
sysctl net.core.default_qdisc

echo "=== 验证可用算法列表（应有 bbr）==="
sysctl net.ipv4.tcp_available_congestion_control
```

**预期输出示例**：
```
tcp_congestion_control = bbr
default_qdisc = fq
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```

### 3.2 BBR Plus

BBR Plus 是社区对 BBR 的改进版，在某些高延迟回国线路上表现更好。**注意**：BBR Plus 需要专用内核。

#### Debian/Ubuntu 安装：

```bash
# 下载 BBR Plus 内核（以 x86_64 为例）
wget https://github.com/UJX6N/bbr-plus/releases/latest/download/linux-headers-6.1.57-bbrplus.x86_64.rpm \
     -O /tmp/bbrplus-headers.rpm 2>/dev/null || \
wget https://github.com/UJX6N/bbr-plus/releases/latest/download/bbrplus-6.1.57-x86_64.rpm \
     -O /tmp/bbrplus-kernel.rpm

# 转换并安装（需要 alien）
apt install -y alien
alien -i /tmp/bbrplus-kernel.rpm
alien -i /tmp/bbrplus-headers.rpm

# 更新 GRUB 引导
update-grub

# 重启
reboot
```

#### CentOS 7 安装（一键脚本）：

```bash
wget -N --no-check-certificate https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh && chmod +x tcp.sh && ./tcp.sh
```

> 💡 BBR Plus 相比官方 BBR 在回国线路上（特别是 163 优化 BGP 线路）可将带宽利用率从 20-30% 提升至 60-80%。

### 3.3 LotServer 锐速

LotServer（原名 ServerSpeeder，锐速）是商业闭源 TCP 加速软件，在特定场景下性能最优，但需要购买授权或使用社区版本。

**安装（Debian/Ubuntu）**：

```bash
# 自动化安装脚本
wget --no-check-certificate -qO /tmp/appex.sh "https://raw.githubusercontent.com/0oVicero0/serverSpeeder_Install/master/appex.sh" && \
chmod +x /tmp/appex.sh && \
bash /tmp/appex.sh install

# 启动
/appex/bin/lotServer.sh start

# 开机自启
systemctl enable lotserver || update-rc.d lotserver defaults
```

**常用命令**：

```bash
/appex/bin/lotServer.sh status    # 查看状态
/appex/bin/lotServer.sh restart   # 重启
/appex/bin/lotServer.sh uninstall # 卸载
```

> ⚠️ **警告**：LotServer 与 BBR 冲突，启用前需先关闭 BBR：`sysctl -w net.ipv4.tcp_congestion_control=cubic`

### 3.4 内核升级（若需要）

如果当前内核 < 4.9，需要升级。以下是常见 VPS 系统的升级方法：

#### Debian 10/11 升级内核：

```bash
# 安装 5.10 LTS 内核
apt update && apt install -y gnupg2
wget -qO- https://www.elrepo.org/elrepo-release-11.el7.elrepo.noarch.rpm | bash
yum install -y elrepo-release
yum install -y kernel-ml-5.10.0

# 查看已安装内核
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg

# 设置默认引导内核（替换数字）
grub2-set-default 0

# 重启
reboot
```

#### Ubuntu 20.04/22.04 升级内核：

```bash
# 安装 HWE 内核（硬件支持栈）
apt install -y --install-recommends linux-generic-hwe-20.04
# Ubuntu 22.04:
# apt install -y --install-recommends linux-generic-hwe-22.04

# 重启
reboot

# 验证
uname -r
```

---

## 四、TCP 参数深度调优

### 4.1 缓冲区与内存配置

TCP 发送/接收缓冲区的大小直接影响吞吐量。默认值通常保守（小于 256KB），在高速 VPS 上严重限制了性能。

```bash
# 查看当前 TCP 内存参数
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem
sysctl net.core.rmem_max
sysctl net.core.wmem_max
```

**参数含义（`tcp_rmem` / `tcp_wmem` 为三个值）**：
- 第一个值：最小缓冲区（4KB）
- 第二个值：默认值（约 16KB，过小！）
- 第三个值：最大缓冲区（自动调优上限）

**高带宽 VPS 推荐配置**：

```bash
# 接收缓冲区
net.core.rmem_max = 16777216      # 16MB
net.core.rmem_default = 8388608    # 8MB
net.ipv4.tcp_rmem = 4096 87380 16777216

# 发送缓冲区
net.core.wmem_max = 16777216      # 16MB
net.core.wmem_default = 8388608   # 8MB
net.ipv4.tcp_wmem = 4096 65536 16777216

# 双向内存上限（防止进程占用过多）
net.core.optmem_max = 65536
```

> 📌 **经验法则**：缓冲区设为带宽 × 延迟的 2-3 倍。例如：100Mbps 带宽 × 150ms 延迟 ≈ 2.4MB 最小缓冲区。

### 4.2 连接跟踪优化

VPS 同时处理大量连接时（如运行代理服务），nf_conntrack 表会迅速增长。若超过上限，新连接会被丢弃：

```bash
# 查看当前连接跟踪数
cat /proc/sys/net/netfilter/nf_conntrack_count

# 查看上限
sysctl net.netfilter.nf_conntrack_max
```

**典型问题**：连接数达到上限时，`dmesg` 会看到 `nf_conntrack: table full, dropping packet`。

**优化配置**：

```bash
# 提高连接跟踪上限
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30
```

> ⚠️ **注意**：nf_conntrack 是有状态的，关闭它会失去连接跟踪功能。某些 NAT 场景下不建议关闭。

### 4.3 TCP 时间戳与快速打开

```bash
# TCP 时间戳（更精确的 RTT 测量，BBR 依赖此功能）
net.ipv4.tcp_timestamps = 1

# TCP SACK（选择性确认，提升重传效率）
net.ipv4.tcp_sack = 1
net.core.somaxconn = 8192

# TCP Fast Open（减少握手延迟，Linux 3.7+）
# 0=关闭, 1=客户端可用, 3=客户端+服务器均可用
net.ipv4.tcp_fastopen = 3

# 启用 TCP MD5（高级安全选项）
# net.ipv4.tcp_md5sig_keys = 1
```

### 4.4 TIME_WAIT 状态优化

大量短连接场景下（如 HTTP 代理），TIME_WAIT 状态的连接会堆积：

```bash
# 允许复用 TIME_WAIT 状态的端口（关键！）
net.ipv4.tcp_tw_reuse = 1

# 缩短 FIN_WAIT_2 超时
net.ipv4.tcp_fin_timeout = 30

# 缩短 KEEPALIVE 探测时间（快速发现断开的连接）
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
```

### 4.5 完整 sysctl.conf 参考

将以下内容完整替换 `/etc/sysctl.conf`（**操作前先备份**：`cp /etc/sysctl.conf /etc/sysctl.conf.bak`）：

```bash
# ====== /etc/sysctl.conf ======
# 内核参数优化配置 - VPS 网络调优

# ========== 网络基础 ==========
# 允许转发（代理/VPN 必须开启）
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# 关闭 ICMP 重定向（安全+减少干扰）
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0

# ========== TCP BBR 拥塞控制 ==========
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# ========== TCP 缓冲区配置 ==========
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_mem = 786432 1048576 1572864

# ========== 连接跟踪 ==========
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30

# ========== TCP 时间参数 ==========
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_sack = 1
net.core.somaxconn = 8192
net.ipv4.tcp_fastopen = 3

# ========== TIME_WAIT 优化 ==========
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000

# ========== TCP Keepalive ==========
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# ========== MTU 优化 ==========
# 关闭 MTU 自动探测（在隧道环境中可能有问题）
# net.ipv4.tcp_mtu_probing = 1

# ========== 本地端口范围 ==========
# 大量并发连接时提高可用端口数
net.ipv4.ip_local_port_range = 10240 65535

# 应用配置
sysctl -p
```

---

## 五、路由优化实战

### 5.1 识别当前路由问题

网络慢不一定全是 TCP 算法的问题——**路由本身可能才是瓶颈**。常见的回国路由问题：

1. **绕路**：VPS → 日本 → 美国 → 中国，比直连多 3-5 倍延迟
2. **运营商对等**：163 骨干网（默认）对海外优化差，CN2 GIA 对中国优化好
3. **高负载节点**：晚高峰挤占严重
4. **BGP 劫持**：极少数情况下，路由被错误引导

**诊断第一步：测路由**

```bash
# 使用 besttrace 查看完整路由路径
wget https://github.com/zq/besttrace/releases/download/v1.3/besttrace-linux-amd64 -O besttrace
chmod +x besttrace
./besttrace -q 1 -g cn 202.96.209.133   # 测电信
./besttrace -q 1 -g cn 221.12.1.227     # 测联通
./besttrace -q 1 -g cn 211.136.112.50   # 测移动
```

**路由关键节点判断**：
- `59.43.*.*` → CN2 GIA 线路（最优）
- `202.97.*.*` → 163 骨干网（普通）
- `219.158.*.*` → 联通 169（较差）
- `221.176.*.*` → 移动自有（尚可）

### 5.2 线路类型详解

| 线路类型 | 标识 IP 段 | 延迟(上海) | 带宽 | 价格 | 推荐度 |
|----------|-----------|-----------|------|------|--------|
| **CN2 GIA** | 59.43.x.x | 120-160ms | 1-5Gbps | ¥150+/月 | ⭐⭐⭐⭐⭐ |
| **CN2 GT** | 59.43.x.x | 140-180ms | 0.1-1Gbps | ¥60-100/月 | ⭐⭐⭐ |
| **9929** | 172.16.x.x | 130-170ms | 1Gbps | ¥80+/月 | ⭐⭐⭐⭐ |
| **10099** | 100.64.x.x | 140-190ms | 1Gbps | ¥50+/月 | ⭐⭐⭐ |
| **163 优化** | AS4134 | 160-250ms | 1-10Gbps | ¥20-50/月 | ⭐⭐ |
| **普通 163** | AS4134 | 180-300ms | 1-10Gbps | ¥10-30/月 | ⭐ |

### 5.3 路由测试方法

#### 综合路由测试脚本：

```bash
cat > /tmp/route-test.sh << 'SCRIPT'
#!/bin/bash
echo "===== VPS 回国路由测试 ====="

# 目标 IP
declare -A TARGETS=(
  ["北京电信"]="202.96.209.133"
  ["上海电信"]="202.96.209.5"
  ["广州电信"]="202.96.128.86"
  ["北京联通"]="221.12.1.227"
  ["上海联通"]="221.12.33.227"
  ["广州移动"]="211.136.112.50"
)

for city in "${!TARGETS[@]}"; do
  ip="${TARGETS[$city]}"
  echo ""
  echo ">>> $city ($ip)"
  
  # 延迟测试（5次取平均）
  rtt=$(ping -c 5 -W 3 $ip 2>/dev/null | tail -1 | awk -F'/' '{print $5}')
  if [ -n "$rtt" ]; then
    echo "平均延迟: ${rtt}ms"
  else
    echo "超时或不可达"
  fi
  
  # 路由跳数（限前10跳）
  traceroute -m 10 -n -q 1 $ip 2>/dev/null | tail -3
done
SCRIPT

chmod +x /tmp/route-test.sh
bash /tmp/route-test.sh
```

#### 判断回国线路质量：

```bash
# 判断是否为 CN2 GIA
traceroute -m 3 -n 202.96.209.133 | grep "59.43" && echo "✓ 检测到 CN2 GIA" || echo "✗ 非 CN2 GIA"

# 判断是否为 163 网络
traceroute -m 3 -n 202.96.209.133 | grep "202.97" && echo "检测到 163 骨干网"
```

### 5.4 路由策略建议

**如果当前路由差（163 线路）**：

| 方案 | 实施难度 | 效果 | 成本 |
|------|---------|------|------|
| 换用 CN2 GIA VPS | 低（换商家） | ⭐⭐⭐⭐⭐ | 较高 |
| 使用回国优化 CDN | 低 | ⭐⭐⭐⭐ | 中 |
| 使用 UDP 协议（WireGuard） | 中 | ⭐⭐⭐⭐ | 低 |
| 购买优质优化线路 | 低 | ⭐⭐⭐⭐⭐ | 中 |

**重要提醒**：BBR 等 TCP 优化对 **路由本身差** 的问题改善有限。路由差时，先考虑更换 VPS 商家或线路类型，再叠加 TCP 优化才能事半功倍。

---

## 六、网络监控与测速

### 6.1 综合性能测试

**推荐脚本 bench.sh**（teddysun 出品，覆盖 CPU/内存/磁盘/网络）：

```bash
# 方法1：标准版
wget -qO- bench.sh | bash

# 方法2：更详细版本（推荐）
bash <(wget -qO- https://raw.githubusercontent.com/teddysun/across/master/bench.sh)
```

**自行搭建 speedtest**（避免测速节点限制）：

```bash
# 在本地 VPS 上搭建 speedtest 服务端
docker run -d -p 80:80 ghcr.io/alexandre-t Roxalid/speedtest:latest

# 或使用 python 快速自建
pip3 install speedtest-cli
speedtest-cli --server YOUR_SERVER_ID
```

### 6.2 回国延迟测试

```bash
# 脚本：测试到国内三大运营商核心节点的延迟
cat > /tmp/china-latency.sh << 'EOF'
#!/bin/bash
declare -A NODES=(
  ["北京电信"]="202.96.209.133"
  ["上海电信"]="202.96.209.5"
  ["广州电信"]="202.96.128.86"
  ["北京联通"]="221.12.1.227"
  ["上海联通"]="221.12.33.227"
  ["广州移动"]="211.136.112.50"
  ["北京教育网"]="202.112.27.2"
)

echo "===== 回国延迟测试 ====="
echo ""

for city in "${!NODES[@]}"; do
  ip="${NODES[$city]}"
  result=$(ping -c 10 -W 2 $ip 2>/dev/null)
  if [ $? -eq 0 ]; then
    avg=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $5}')
    min=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $4}')
    max=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $6}')
    printf "%-12s %-16s avg:%-8s min:%-8s max:%s\n" "$city" "$ip" "${avg}ms" "${min}ms" "${max}ms"
  else
    echo "$city ($ip): 超时"
  fi
done
EOF

bash /tmp/china-latency.sh
```

### 6.3 带宽压测

```bash
# 使用 iperf3 测试实际带宽上限
# VPS 端：
apt install -y iperf3
iperf3 -s -p 5201

# 客户端（另一台机器）：
iperf3 -c <VPS_IP> -p 5201 -t 30 -R   # 下载测试
iperf3 -c <VPS_IP> -p 5201 -t 30      # 上传测试
```

### 6.4 实时监控工具

```bash
# 安装 bmon（实时带宽监控）
apt install -y bmon
bmon

# 安装 nethogs（按进程监控流量）
apt install -y nethogs
nethogs

# 安装 tcptrack（按连接监控）
apt install -y tcptrack
tcptrack -i eth0
```

---

## 七、故障排查

### 7.1 BBR 未生效

**检查清单**：

```bash
# 1. 确认内核版本 >= 4.9
uname -r

# 2. 确认 BBR 模块已加载
lsmod | grep bbr
# 如果没有输出，执行：
modprobe tcp_bbr

# 3. 确认 sysctl 参数已写入
sysctl net.ipv4.tcp_congestion_control
# 应为：net.ipv4.tcp_congestion_control = bbr

# 4. 确认 fq qdisc 已加载
sysctl net.core.default_qdisc
# 应为：net.core.default_qdisc = fq

# 5. 确认服务器端和客户端都启用了 BBR
# （仅一侧启用效果有限，但仍有效）
```

**常见问题**：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `lsmod` 无 bbr | 内核不支持 | 升级内核至 4.9+ |
| `sysctl` 显示 cubic | 参数未生效 | 重新运行 `sysctl -p`，检查 `/etc/sysctl.conf` |
| 临时启用后重启失效 | 未写入配置文件 | 将参数写入 `/etc/sysctl.conf` |
| BBR 反而更慢 | 对端不支持 BBR | 退回 CUBIC：`sysctl -w net.ipv4.tcp_congestion_control=cubic` |

### 7.2 网速无明显提升

**逐步排查**：

```bash
# 1. 检查实际带宽（iperf3 测试，排除 VPS 本身带宽上限）
# 如果 iperf3 也只能跑几百 Mbps，说明是 VPS 商家限速

# 2. 检查网络类型（路由问题，TCP 优化无法解决）
traceroute -m 5 202.96.209.133
# 如果路由绕路严重（如经过美国），优先解决路由问题

# 3. 检查是否开启了 QoS 或流量整形
tc qdisc show

# 4. 检查是否有防火墙规则影响
iptables -L -n | head -20

# 5. 测试 UDP 性能（UDP 不受 TCP 拥塞控制影响）
iperf3 -u -c <SERVER> -b 1G -t 10
# 如果 UDP 快但 TCP 慢 → TCP 优化有效
# 如果 UDP 也慢 → 网络本身或 VPS 带宽限制
```

### 7.3 连接频繁断开

```bash
# 1. 检查 keepalive 是否开启
sysctl net.ipv4.tcp_keepalive_time

# 2. 检查 MTU 是否匹配（VPN/隧道常见 MTU 问题）
# 常见值：1500（以太网）、1400（PPPoE）、1300（VPN 隧道）
ping -M do -s 1400 8.8.8.8    # 排查过大数据包
ping -M do -s 1300 8.8.8.8    # 尝试更小的 MTU

# 3. 检查 TCP 超时设置
sysctl net.ipv4.tcp_fin_timeout
# 建议值：15-30

# 4. 检查 TIME_WAIT 连接数是否过高
netstat -an | grep TIME_WAIT | wc -l
# 如果 > 50000，考虑优化 TIME_WAIT
```

---

## 八、效果对比与验证

优化完成后，用以下方法验证效果：

```bash
# 创建对比测试脚本
cat > /tmp/compare.sh << 'EOF'
#!/bin/bash
echo "===== BBR 优化效果对比测试 ====="
echo ""

# 测试目标（选择国内节点）
TARGET="202.96.209.133"

echo ">> 当前拥塞控制算法: $(sysctl -n net.ipv4.tcp_congestion_control)"
echo ">> 当前队列算法: $(sysctl -n net.core.default_qdisc)"
echo ""

echo ">> 延迟测试（10次）:"
ping -c 10 $TARGET | tail -1

echo ""
echo ">> 带宽测试:"
speedtest-cli --simple 2>/dev/null || echo "speedtest-cli 未安装"

echo ""
echo ">> 参考方案对比（可复制到其他机器测试）:"
echo "优化前: sysctl -w net.ipv4.tcp_congestion_control=cubic"
echo "优化后: sysctl -w net.ipv4.tcp_congestion_control=bbr"
EOF

bash /tmp/compare.sh
```

**效果预期**：

| 场景 | 优化前（CUBIC） | 优化后（BBR） | 提升幅度 |
|------|----------------|--------------|---------|
| 50Mbps 回国线，150ms 延迟 | 10-15 Mbps | 30-40 Mbps | 2-3x |
| 100Mbps 回国线，130ms 延迟 | 30-40 Mbps | 70-90 Mbps | 2x |
| 1Gbps 优质 CN2 GIA | 300-500 Mbps | 700-900 Mbps | 1.5-2x |

---

## 九、常见问题

**Q: 开了 BBR 后反而更慢了？**

A: BBR 在两端对等网络（两端都支持 BBR）时效果最佳。如果访问的服务器端不支持 BBR，或你的网络本身是高带宽低延迟的优质直连，BBR 可能没有明显收益甚至略微降低性能。尝试退回 CUBIC：`sysctl -w net.ipv4.tcp_congestion_control=cubic`

**Q: BBR 和锐速（LotServer）哪个好？**

A: 锐速在单连接高吞吐场景（如大文件下载）通常更优，但需要额外安装且为闭源软件。BBR 完全免费、开源、已集成在内核中，通用性更好。**建议**：先试 BBR，效果不好再试锐速。

**Q: 开了 BBR 后，YouTube 等平台速度反而变慢？**

A: 这是因为 YouTube 等平台的服务端本身有拥塞控制逻辑，它们探测到你的连接"太快"（接近带宽上限）时会主动降低发包速率。可以通过限制 BBR 的 pacing rate 来解决，但通常不必要。

**Q: 内核升级失败，VPS 无法启动怎么办？**

A: 大多数 KVM VPS 提供 VNC/KVM console，可以通过 GRUB 恢复旧内核。OVZ 架构不支持内核升级，只能换用预装高版本内核的模板。

**Q: sysctl -p 报错 "No such file or directory"？**

A: 检查参数名是否拼写错误，例如 `net.ipv4.tcp_congestion_control` 不能写成 `net.tcp.congestion_control`。

**Q: 我的 VPS 是 OpenVZ 架构，能用 BBR 吗？**

A: OpenVZ 通常不支持自定义内核模块，BBR 能否使用取决于宿主机内核是否已支持。如果 `sysctl net.ipv4.tcp_available_congestion_control` 输出包含 bbr，则可用。

---

## 十、相关资源

| 资源 | 地址 | 说明 |
|------|------|------|
| **VPSVIP** | [https://vpsvip.net](https://vpsvip.net) | VPS 推荐与评测 |
| **ClashVIP** | [https://clashvip.net](https://clashvip.net) | Clash 订阅与配置教程 |
| **ClashHub** | [https://clashhub.net](https://clashhub.net) | 规则集与配置中心 |
| **BBR 官方 GitHub** | [google/bbr](https://github.com/google/bbr) | BBR 官方源码与文档 |
| **BBR Plus** | [github.com/UJX6N/bbr-plus](https://github.com/UJX6N/bbr-plus) | BBR Plus 内核仓库 |
| **teddysun 脚本** | [teddysun/across](https://raw.githubusercontent.com/teddysun/across/master/bench.sh) | 综合测试脚本 |
| **Linux 内核下载** | [kernel.org](https://www.kernel.org) | 官方内核存档 |

---

## 📄 许可证

MIT 许可证 - 详见 [LICENSE](LICENSE)

---

**最后更新**：2026-08-21
