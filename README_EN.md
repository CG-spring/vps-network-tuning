# VPS Network Tuning Complete Guide

> Systematic optimization of VPS cross-border network performance: from kernel parameters to routing strategies

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/CG-spring/vps-network-tuning)](https://github.com/CG-spring/vps-network-tuning)
[![Kernel](https://img.shields.io/badge/kernel-4.9%2B-blue.svg)](https://www.kernel.org)
[![BBR](https://img.shields.io/badge/BBR-Ready-brightgreen.svg)](https://github.com/google/bbr)

**[English](README_EN.md)** | **[中文](README.md)**

---

## 📋 Table of Contents

- [I. Overview & Principles](#i-overview--principles)
  - [1.1 Why Your VPS Network Needs Tuning](#11-why-your-vps-network-needs-tuning)
  - [1.2 TCP Congestion Control Algorithm Comparison](#12-tcp-congestion-control-algorithm-comparison)
  - [1.3 How BBR Works](#13-how-bbr-works)
- [II. Quick Start](#ii-quick-start)
- [III. BBR Installation Deep Dive](#iii-bbr-installation-deep-dive)
  - [3.1 Official BBR (Recommended)](#31-official-bbr-recommended)
  - [3.2 BBR Plus](#32-bbr-plus)
  - [3.3 LotServer (ServerSpeeder)](#33-lotserver-serverspeeder)
  - [3.4 Kernel Upgrade (If Needed)](#34-kernel-upgrade-if-needed)
- [IV. Advanced TCP Parameter Tuning](#iv-advanced-tcp-parameter-tuning)
  - [4.1 Buffer and Memory Configuration](#41-buffer-and-memory-configuration)
  - [4.2 Connection Tracking Optimization](#42-connection-tracking-optimization)
  - [4.3 TCP Timestamps & Fast Open](#43-tcp-timestamps--fast-open)
  - [4.4 TIME_WAIT State Optimization](#44-time_wait-state-optimization)
  - [4.5 Complete sysctl.conf Reference](#45-complete-sysctlconf-reference)
- [V. Routing Optimization in Practice](#v-routing-optimization-in-practice)
  - [5.1 Identifying Routing Problems](#51-identifying-routing-problems)
  - [5.2 Understanding China Route Types](#52-understanding-china-route-types)
  - [5.3 Routing Test Methods](#53-routing-test-methods)
  - [5.4 Routing Strategy Recommendations](#54-routing-strategy-recommendations)
- [VI. Network Monitoring & Speed Testing](#vi-network-monitoring--speed-testing)
  - [6.1 Comprehensive Performance Tests](#61-comprehensive-performance-tests)
  - [6.2 China Latency Testing](#62-china-latency-testing)
  - [6.3 Bandwidth Stress Testing](#63-bandwidth-stress-testing)
  - [6.4 Real-time Monitoring Tools](#64-real-time-monitoring-tools)
- [VII. Troubleshooting](#vii-troubleshooting)
  - [7.1 BBR Not Working](#71-bbr-not-working)
  - [7.2 No Significant Speed Improvement](#72-no-significant-speed-improvement)
  - [7.3 Frequent Connection Drops](#73-frequent-connection-drops)
- [VIII. Benchmark & Verification](#viii-benchmark--verification)
- [IX. FAQ](#ix-faq)
- [X. Related Resources](#x-related-resources)

---

## I. Overview & Principles

### 1.1 Why Your VPS Network Needs Tuning

After renting an overseas VPS, many users notice a frustrating gap: the plan claims 1Gbps bandwidth, yet actual speeds to international websites barely reach a few Mbps. Latency looks fine, yet downloads crawl. The root cause isn't the bandwidth itself — it's **how well the TCP congestion control mechanism matches your actual network path**.

By default, Linux uses the CUBIC or Reno congestion control algorithm. These algorithms were designed for wired networks and are extremely sensitive to packet loss. The moment a packet is dropped — whether from a congested submarine cable, an overloaded international router, or ISP QoS throttling — CUBIC immediately halves its sending window. In modern跨境 network environments with complex routing, "silent packet drops" (where packets are quietly discarded at some intermediate node) cause CUBIC to severely underperform.

**The core goal of tuning**: Make the TCP congestion control algorithm more accurately perceive real network capacity, fully utilizing bandwidth while avoiding self-throttling due to misjudgments.

### 1.2 TCP Congestion Control Algorithm Comparison

| Algorithm | Developer | Best For | Kernel Required | Stability |
|-----------|-----------|----------|-----------------|-----------|
| **CUBIC** (default) | Linux Kernel | Low-loss wired networks | 2.6.19+ | ⭐⭐⭐⭐⭐ |
| **BBR** | Google | High-latency/packet-loss cross-border | 4.9+ | ⭐⭐⭐⭐ |
| **BBR Plus** | Community | High-latency China routes | 4.14+ | ⭐⭐⭐ |
| **LotServer** | @oxxt | Maximum optimization | Any | ⭐⭐⭐ |
| **Westwood** | Linux Kernel | Wireless/high-jitter networks | Any | ⭐⭐⭐⭐ |

**Recommendation**: For overseas VPS cross-border usage, try BBR first. For China return routes with latency >150ms or noticeable packet loss, try BBR Plus. Consider LotServer for extreme scenarios.

### 1.3 How BBR Works

BBR (Bottleneck Bandwidth and Round-trip propagation time) is Google's 2016 congestion control algorithm with a fundamentally different approach from traditional algorithms:

**Traditional algorithms (loss-driven)**:
```
Send Rate ↔ Loss Event
Loss → Halve window → Slow recovery
```

**BBR (model-driven)**:
```
Send Rate ↔ Bandwidth Estimate + RTT Estimate
Does NOT rely on packet loss.
Instead, actively measures bottleneck bandwidth and RTT.
```

BBR's core logic: Maintains a pacing rate always equal to `min(Bottleneck Bandwidth, Send Window) / RTT`. Even with slight packet loss, as long as RTT doesn't spike dramatically, BBR continues sending at maximum bandwidth — it won't self-throttle.

**BBR works best when**:
- Bandwidth > 10Mbps
- One-way latency > 20ms (higher latency = more noticeable improvement)
- Some packet loss or network fluctuation exists

---

## II. Quick Start

Want to skip the details? One command to enable BBR:

```bash
wget -qO- https://raw.githubusercontent.com/CG-spring/vps-network-tuning/main/install-bbr.sh | bash
```

Manual setup (3 minutes):

```bash
# 1. Check current congestion control algorithm
sysctl net.ipv4.tcp_congestion_control

# 2. Check kernel version (BBR requires 4.9+)
uname -r

# 3. Enable BBR (append to /etc/sysctl.conf)
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

# 4. Apply configuration
sysctl -p

# 5. Verify BBR is loaded
sysctl net.ipv4.tcp_congestion_control
# Expected: net.ipv4.tcp_congestion_control = bbr

# 6. Verify qdisc is active
sysctl net.core.default_qdisc
# Expected: net.core.default_qdisc = fq
```

> ⚠️ **Note**: BBR requires kernel 4.9+. If `uname -r` shows a version below 4.9, you need to upgrade the kernel first (see section 3.4).

---

## III. BBR Installation Deep Dive

### 3.1 Official BBR (Recommended)

Official BBR is built into Linux kernel 4.9+. No additional installation needed — just enable it via sysctl:

```bash
# Full enable procedure
cat >> /etc/sysctl.conf << 'EOF'
# BBR Congestion Control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF

sysctl -p

# Four-way verification
echo "=== Verify BBR module is loaded ==="
lsmod | grep bbr

echo "=== Verify congestion control algorithm ==="
sysctl net.ipv4.tcp_congestion_control

echo "=== Verify qdisc ==="
sysctl net.core.default_qdisc

echo "=== Verify available algorithms (should include bbr) ==="
sysctl net.ipv4.tcp_available_congestion_control
```

**Expected output examples**:
```
tcp_congestion_control = bbr
default_qdisc = fq
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```

### 3.2 BBR Plus

BBR Plus is a community-improved version of BBR that performs better on certain high-latency China return routes. **Note**: BBR Plus requires a dedicated kernel.

#### Debian/Ubuntu Installation:

```bash
# Download BBR Plus kernel (x86_64 example)
wget https://github.com/UJX6N/bbr-plus/releases/latest/download/linux-headers-6.1.57-bbrplus.x86_64.rpm \
     -O /tmp/bbrplus-headers.rpm 2>/dev/null || \
wget https://github.com/UJX6N/bbr-plus/releases/latest/download/bbrplus-6.1.57-x86_64.rpm \
     -O /tmp/bbrplus-kernel.rpm

# Convert and install (requires alien)
apt install -y alien
alien -i /tmp/bbrplus-kernel.rpm
alien -i /tmp/bbrplus-headers.rpm

# Update GRUB
update-grub

# Reboot
reboot
```

#### CentOS 7 One-Click Install:

```bash
wget -N --no-check-certificate https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh && chmod +x tcp.sh && ./tcp.sh
```

> 💡 Compared to official BBR, BBR Plus on China return routes (especially 163-optimized BGP routes) can increase bandwidth utilization from 20-30% to 60-80%.

### 3.3 LotServer (ServerSpeeder)

LotServer (formerly ServerSpeeder) is a commercial closed-source TCP acceleration software that delivers the best performance in specific scenarios, but requires purchasing a license or using community builds.

**Installation (Debian/Ubuntu)**:

```bash
# Automated install script
wget --no-check-certificate -qO /tmp/appex.sh "https://raw.githubusercontent.com/0oVicero0/serverSpeeder_Install/master/appex.sh" && \
chmod +x /tmp/appex.sh && \
bash /tmp/appex.sh install

# Start
/appex/bin/lotServer.sh start

# Enable on boot
systemctl enable lotserver || update-rc.d lotserver defaults
```

**Common Commands**:

```bash
/appex/bin/lotServer.sh status    # Check status
/appex/bin/lotServer.sh restart   # Restart
/appex/bin/lotServer.sh uninstall # Uninstall
```

> ⚠️ **Warning**: LotServer conflicts with BBR. Disable BBR before enabling LotServer: `sysctl -w net.ipv4.tcp_congestion_control=cubic`

### 3.4 Kernel Upgrade (If Needed)

If current kernel < 4.9, you need to upgrade. Here are upgrade methods for common VPS systems:

#### Debian 10/11 Kernel Upgrade:

```bash
# Install 5.10 LTS kernel
apt update && apt install -y gnupg2
wget -qO- https://www.elrepo.org/elrepo-release-11.el7.elrepo.noarch.rpm | bash
yum install -y elrepo-release
yum install -y kernel-ml-5.10.0

# List installed kernels
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg

# Set default boot kernel (replace number as needed)
grub2-set-default 0

# Reboot
reboot
```

#### Ubuntu 20.04/22.04 Kernel Upgrade:

```bash
# Install HWE kernel (Hardware Enablement Stack)
apt install -y --install-recommends linux-generic-hwe-20.04
# For Ubuntu 22.04:
# apt install -y --install-recommends linux-generic-hwe-22.04

# Reboot
reboot

# Verify
uname -r
```

---

## IV. Advanced TCP Parameter Tuning

### 4.1 Buffer and Memory Configuration

TCP send/receive buffer size directly affects throughput. Default values are typically conservative (<256KB), severely limiting performance on high-speed VPS:

```bash
# Check current TCP memory parameters
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem
sysctl net.core.rmem_max
sysctl net.core.wmem_max
```

**Parameter meaning (`tcp_rmem` / `tcp_wmem` — three values)**:
- First value: Minimum buffer (4KB)
- Second value: Default (~16KB, too small!)
- Third value: Maximum buffer (auto-tuning upper limit)

**Recommended configuration for high-bandwidth VPS**:

```bash
# Receive buffer
net.core.rmem_max = 16777216      # 16MB
net.core.rmem_default = 8388608   # 8MB
net.ipv4.tcp_rmem = 4096 87380 16777216

# Send buffer
net.core.wmem_max = 16777216      # 16MB
net.core.wmem_default = 8388608   # 8MB
net.ipv4.tcp_wmem = 4096 65536 16777216

# Bidirectional memory cap (prevents process from consuming too much)
net.core.optmem_max = 65536
```

> 📌 **Rule of thumb**: Set buffer to 2-3× the bandwidth-delay product. Example: 100Mbps × 150ms ≈ 2.4MB minimum buffer.

### 4.2 Connection Tracking Optimization

When a VPS handles many concurrent connections (e.g., running proxy services), the nf_conntrack table grows rapidly. When it exceeds the limit, new connections get dropped:

```bash
# Check current connection tracking count
cat /proc/sys/net/netfilter/nf_conntrack_count

# Check limit
sysctl net.netfilter.nf_conntrack_max
```

**Typical symptom**: When connection count reaches the limit, `dmesg` shows `nf_conntrack: table full, dropping packet`.

**Optimization configuration**:

```bash
# Increase connection tracking limit
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30
```

> ⚠️ **Note**: nf_conntrack is stateful — disabling it removes connection tracking capability. Not recommended in certain NAT scenarios.

### 4.3 TCP Timestamps & Fast Open

```bash
# TCP timestamps (more accurate RTT measurement, required by BBR)
net.ipv4.tcp_timestamps = 1

# TCP SACK (Selective Acknowledgment, improves retransmission efficiency)
net.ipv4.tcp_sack = 1
net.core.somaxconn = 8192

# TCP Fast Open (reduces handshake latency, Linux 3.7+)
# 0=disabled, 1=client only, 3=both client and server
net.ipv4.tcp_fastopen = 3
```

### 4.4 TIME_WAIT State Optimization

In high-volume short-connection scenarios (e.g., HTTP proxies), TIME_WAIT connections accumulate:

```bash
# Allow reusing ports in TIME_WAIT state (critical!)
net.ipv4.tcp_tw_reuse = 1

# Shorten FIN_WAIT_2 timeout
net.ipv4.tcp_fin_timeout = 30

# Shorten KEEPALIVE probe interval (detect broken connections faster)
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
```

### 4.5 Complete sysctl.conf Reference

Replace `/etc/sysctl.conf` with the following content (**backup first**: `cp /etc/sysctl.conf /etc/sysctl.conf.bak`):

```bash
# ====== /etc/sysctl.conf ======
# Kernel Parameter Optimization - VPS Network Tuning

# ========== Network Basics ==========
# Enable forwarding (required for proxy/VPN)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Disable ICMP redirects (security + reduces interference)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0

# ========== TCP BBR Congestion Control ==========
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# ========== TCP Buffer Configuration ==========
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_mem = 786432 1048576 1572864

# ========== Connection Tracking ==========
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30

# ========== TCP Time Parameters ==========
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_sack = 1
net.core.somaxconn = 8192
net.ipv4.tcp_fastopen = 3

# ========== TIME_WAIT Optimization ==========
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000

# ========== TCP Keepalive ==========
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# ========== Local Port Range ==========
# Increase available ports for massive concurrent connections
net.ipv4.ip_local_port_range = 10240 65535

# Apply configuration
sysctl -p
```

---

## V. Routing Optimization in Practice

### 5.1 Identifying Routing Problems

Slow network isn't always a TCP algorithm problem — **routing itself may be the bottleneck**. Common China return route issues:

1. **Detour**: VPS → Japan → USA → China, adding 3-5× latency vs. direct
2. **ISP peering**: 163 backbone (default) has poor overseas optimization; CN2 GIA is well-optimized for China
3. **High-load nodes**: Severe congestion during peak evening hours
4. **BGP hijacking**: Rare, but happens

**First diagnostic step: Route test**

```bash
# Use besttrace to view complete routing path
wget https://github.com/zq/besttrace/releases/download/v1.3/besttrace-linux-amd64 -O besttrace
chmod +x besttrace
./besttrace -q 1 -g cn 202.96.209.133   # Test China Telecom
./besttrace -q 1 -g cn 221.12.1.227     # Test China Unicom
./besttrace -q 1 -g cn 211.136.112.50    # Test China Mobile
```

**Key node identification**:
- `59.43.*.*` → CN2 GIA route (optimal)
- `202.97.*.*` → 163 backbone (regular)
- `219.158.*.*` → Unicom 169 (poor)
- `221.176.*.*` → China Mobile self-owned (acceptable)

### 5.2 Understanding China Route Types

| Route Type | IP Range Marker | Latency (Shanghai) | Bandwidth | Price | Rating |
|------------|----------------|-------------------|-----------|-------|--------|
| **CN2 GIA** | 59.43.x.x | 120-160ms | 1-5Gbps | ¥150+/mo | ⭐⭐⭐⭐⭐ |
| **CN2 GT** | 59.43.x.x | 140-180ms | 0.1-1Gbps | ¥60-100/mo | ⭐⭐⭐ |
| **9929** | 172.16.x.x | 130-170ms | 1Gbps | ¥80+/mo | ⭐⭐⭐⭐ |
| **10099** | 100.64.x.x | 140-190ms | 1Gbps | ¥50+/mo | ⭐⭐⭐ |
| **163 Optimized** | AS4134 | 160-250ms | 1-10Gbps | ¥20-50/mo | ⭐⭐ |
| **Regular 163** | AS4134 | 180-300ms | 1-10Gbps | ¥10-30/mo | ⭐ |

### 5.3 Routing Test Methods

#### Comprehensive routing test script:

```bash
cat > /tmp/route-test.sh << 'SCRIPT'
#!/bin/bash
echo "===== VPS China Routing Test ====="

# Target IPs
declare -A TARGETS=(
  ["Beijing CT"]="202.96.209.133"
  ["Shanghai CT"]="202.96.209.5"
  ["Guangzhou CT"]="202.96.128.86"
  ["Beijing CU"]="221.12.1.227"
  ["Shanghai CU"]="221.12.33.227"
  ["Guangzhou CM"]="211.136.112.50"
)

for city in "${!TARGETS[@]}"; do
  ip="${TARGETS[$city]}"
  echo ""
  echo ">>> $city ($ip)"

  # Latency test (5 pings, average)
  rtt=$(ping -c 5 -W 3 $ip 2>/dev/null | tail -1 | awk -F'/' '{print $5}')
  if [ -n "$rtt" ]; then
    echo "Avg latency: ${rtt}ms"
  else
    echo "Timeout or unreachable"
  fi

  # Route hops (first 10)
  traceroute -m 10 -n -q 1 $ip 2>/dev/null | tail -3
done
SCRIPT

chmod +x /tmp/route-test.sh
bash /tmp/route-test.sh
```

#### Judging China route quality:

```bash
# Check if CN2 GIA
traceroute -m 3 -n 202.96.209.133 | grep "59.43" && echo "✓ CN2 GIA detected" || echo "✗ Not CN2 GIA"

# Check if 163 network
traceroute -m 3 -n 202.96.209.133 | grep "202.97" && echo "163 backbone detected"
```

### 5.4 Routing Strategy Recommendations

**If current routing is poor (163 route)**:

| Solution | Difficulty | Effect | Cost |
|----------|-----------|--------|------|
| Switch to CN2 GIA VPS | Low (change provider) | ⭐⭐⭐⭐⭐ | Higher |
| Use China-optimized CDN | Low | ⭐⭐⭐⭐ | Medium |
| Use UDP protocols (WireGuard) | Medium | ⭐⭐⭐⭐ | Low |
| Purchase premium optimized route | Low | ⭐⭐⭐⭐⭐ | Medium |

**Important reminder**: BBR and other TCP optimizations have **limited improvement** on problems caused by poor routing itself. When routing is bad, prioritize switching VPS provider/route type first, then layer TCP optimization on top for best results.

---

## VI. Network Monitoring & Speed Testing

### 6.1 Comprehensive Performance Tests

**Recommended: bench.sh** (by teddysun, covers CPU/memory/disk/network):

```bash
# Method 1: Standard version
wget -qO- bench.sh | bash

# Method 2: More detailed version (recommended)
bash <(wget -qO- https://raw.githubusercontent.com/teddysun/across/master/bench.sh)
```

**Self-hosted speedtest** (avoids node limitations):

```bash
# Run speedtest server on your VPS
docker run -d -p 80:80 ghcr.io/alexandre-t Roxalid/speedtest:latest

# Or use Python-based quick server
pip3 install speedtest-cli
speedtest-cli --server YOUR_SERVER_ID
```

### 6.2 China Latency Testing

```bash
# Script: Test latency to China major ISP core nodes
cat > /tmp/china-latency.sh << 'EOF'
#!/bin/bash
declare -A NODES=(
  ["Beijing CT"]="202.96.209.133"
  ["Shanghai CT"]="202.96.209.5"
  ["Guangzhou CT"]="202.96.128.86"
  ["Beijing CU"]="221.12.1.227"
  ["Shanghai CU"]="221.12.33.227"
  ["Guangzhou CM"]="211.136.112.50"
  ["Beijing CERNET"]="202.112.27.2"
)

echo "===== China Return Latency Test ====="
echo ""

for city in "${!NODES[@]}"; do
  ip="${NODES[$city]}"
  result=$(ping -c 10 -W 2 $ip 2>/dev/null)
  if [ $? -eq 0 ]; then
    avg=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $5}')
    min=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $4}')
    max=$(echo "$result" | tail -1 | awk -F'/' '{printf "%.1f", $6}')
    printf "%-16s %-16s avg:%-8s min:%-8s max:%s\n" "$city" "$ip" "${avg}ms" "${min}ms" "${max}ms"
  else
    echo "$city ($ip): Timeout"
  fi
done
EOF

bash /tmp/china-latency.sh
```

### 6.3 Bandwidth Stress Testing

```bash
# Use iperf3 to test actual bandwidth ceiling
# VPS side:
apt install -y iperf3
iperf3 -s -p 5201

# Client side (another machine):
iperf3 -c <VPS_IP> -p 5201 -t 30 -R   # Download test
iperf3 -c <VPS_IP> -p 5201 -t 30      # Upload test
```

### 6.4 Real-time Monitoring Tools

```bash
# Install bmon (real-time bandwidth monitoring)
apt install -y bmon
bmon

# Install nethogs (per-process traffic monitoring)
apt install -y nethogs
nethogs

# Install tcptrack (per-connection monitoring)
apt install -y tcptrack
tcptrack -i eth0
```

---

## VII. Troubleshooting

### 7.1 BBR Not Working

**Checklist**:

```bash
# 1. Confirm kernel version >= 4.9
uname -r

# 2. Confirm BBR module is loaded
lsmod | grep bbr
# If no output, run:
modprobe tcp_bbr

# 3. Confirm sysctl parameter is set
sysctl net.ipv4.tcp_congestion_control
# Should show: net.ipv4.tcp_congestion_control = bbr

# 4. Confirm fq qdisc is loaded
sysctl net.core.default_qdisc
# Should show: net.core.default_qdisc = fq

# 5. Confirm both server and client have BBR enabled
# (enabling only one side still helps, but full effect needs both)
```

**Common issues**:

| Problem | Cause | Solution |
|---------|-------|----------|
| `lsmod` shows no bbr | Kernel doesn't support it | Upgrade kernel to 4.9+ |
| `sysctl` shows cubic | Parameter didn't take effect | Re-run `sysctl -p`, check `/etc/sysctl.conf` |
| Temporary enable resets after reboot | Not saved to config | Write parameters to `/etc/sysctl.conf` |
| BBR makes things slower | Peer doesn't support BBR | Revert to CUBIC: `sysctl -w net.ipv4.tcp_congestion_control=cubic` |

### 7.2 No Significant Speed Improvement

**Step-by-step diagnosis**:

```bash
# 1. Test actual bandwidth (iperf3 test, rule out VPS bandwidth cap)
# If iperf3 also only reaches hundreds of Mbps, VPS provider is limiting speed

# 2. Check network type (routing problem — TCP optimization can't solve this)
traceroute -m 5 202.96.209.133
# If routing is seriously circuitous (e.g., via USA), routing is the priority issue

# 3. Check if QoS or traffic shaping is enabled
tc qdisc show

# 4. Check if firewall rules are affecting performance
iptables -L -n | head -20

# 5. Test UDP performance (UDP unaffected by TCP congestion control)
iperf3 -u -c <SERVER> -b 1G -t 10
# If UDP is fast but TCP is slow → TCP optimization is working
# If both are slow → Network itself or VPS bandwidth limit
```

### 7.3 Frequent Connection Drops

```bash
# 1. Check if keepalive is enabled
sysctl net.ipv4.tcp_keepalive_time

# 2. Check if MTU is matched (common MTU issue with VPN/tunnels)
# Common values: 1500 (Ethernet), 1400 (PPPoE), 1300 (VPN tunnel)
ping -M do -s 1400 8.8.8.8    # Test for oversized packets
ping -M do -s 1300 8.8.8.8    # Try smaller MTU

# 3. Check TCP timeout settings
sysctl net.ipv4.tcp_fin_timeout
# Recommended: 15-30

# 4. Check TIME_WAIT connection count
netstat -an | grep TIME_WAIT | wc -l
# If > 50000, consider optimizing TIME_WAIT
```

---

## VIII. Benchmark & Verification

After optimization, verify results with:

```bash
# Create comparison test script
cat > /tmp/compare.sh << 'EOF'
#!/bin/bash
echo "===== BBR Optimization Effect Comparison ====="
echo ""

# Test target (choose a China node)
TARGET="202.96.209.133"

echo ">> Current congestion control: $(sysctl -n net.ipv4.tcp_congestion_control)"
echo ">> Current queue discipline: $(sysctl -n net.core.default_qdisc)"
echo ""

echo ">> Latency test (10 pings):"
ping -c 10 $TARGET | tail -1

echo ""
echo ">> Speed test:"
speedtest-cli --simple 2>/dev/null || echo "speedtest-cli not installed"

echo ""
echo ">> Reference comparison (copy to other machines for testing):"
echo "Before: sysctl -w net.ipv4.tcp_congestion_control=cubic"
echo "After:  sysctl -w net.ipv4.tcp_congestion_control=bbr"
EOF

bash /tmp/compare.sh
```

**Expected results**:

| Scenario | Before (CUBIC) | After (BBR) | Improvement |
|----------|---------------|-------------|-------------|
| 50Mbps China route, 150ms latency | 10-15 Mbps | 30-40 Mbps | 2-3x |
| 100Mbps China route, 130ms latency | 30-40 Mbps | 70-90 Mbps | 2x |
| 1Gbps CN2 GIA | 300-500 Mbps | 700-900 Mbps | 1.5-2x |

---

## IX. FAQ

**Q: Speed got slower after enabling BBR?**

A: BBR performs best in symmetric-peer networks (both ends support BBR). If the destination server doesn't support BBR, or your network is already a high-bandwidth low-latency direct connection, BBR may not provide noticeable benefits or may slightly reduce performance. Revert to CUBIC: `sysctl -w net.ipv4.tcp_congestion_control=cubic`

**Q: Which is better, BBR or ServerSpeeder (LotServer)?**

A: ServerSpeeder typically performs better in single-connection high-throughput scenarios (like large file downloads), but requires additional installation and is closed-source. BBR is completely free, open-source, and built into the kernel. **Recommendation**: Try BBR first. If it doesn't work well, try ServerSpeeder.

**Q: After enabling BBR, YouTube speeds actually got slower?**

A: This happens because YouTube and similar platforms have their own server-side congestion control logic. When they detect your connection "too fast" (near bandwidth limit), they proactively reduce their sending rate. You can address this by limiting BBR's pacing rate, but it's usually unnecessary.

**Q: Kernel upgrade failed and VPS won't boot?**

A: Most KVM VPS providers offer VNC/KVM console access, allowing you to recover to the old kernel via GRUB. OpenVZ architecture doesn't support kernel upgrades — you can only switch to a template with a pre-installed newer kernel.

**Q: `sysctl -p` reports "No such file or directory"?**

A: Check for typos in parameter names, e.g., `net.ipv4.tcp_congestion_control` cannot be written as `net.tcp.congestion_control`.

**Q: Is my VPS OpenVZ architecture — can I use BBR?**

A: OpenVZ typically doesn't support custom kernel modules. Whether BBR works depends on whether the host's kernel already supports it. If `sysctl net.ipv4.tcp_available_congestion_control` output includes bbr, it's available.

---

## X. Related Resources

| Resource | URL | Description |
|----------|-----|-------------|
| **VPSVIP** | [https://vpsvip.net](https://vpsvip.net) | VPS recommendations & reviews |
| **ClashVIP** | [https://clashvip.net](https://clashvip.net) | Clash subscription & config tutorials |
| **ClashHub** | [https://clashhub.net](https://clashhub.net) | Rule sets & config hub |
| **BBR Official GitHub** | [google/bbr](https://github.com/google/bbr) | BBR official source & docs |
| **BBR Plus** | [github.com/UJX6N/bbr-plus](https://github.com/UJX6N/bbr-plus) | BBR Plus kernel repository |
| **teddysun Scripts** | [teddysun/across](https://raw.githubusercontent.com/teddysun/across/master/bench.sh) | Comprehensive benchmark scripts |
| **Linux Kernel Archive** | [kernel.org](https://www.kernel.org) | Official kernel archives |

---

## 📄 License

MIT License — See [LICENSE](LICENSE)

---

**Last Updated**: 2026-08-21
