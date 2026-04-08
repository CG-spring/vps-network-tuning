# VPS Network Tuning Guide

> TCP/BBR optimization, routing and latency improvement for VPS servers

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/CG-spring/vps-network-tuning)](https://github.com/CG-spring/vps-network-tuning)

**[中文](README.md)** | **[English](README_EN.md)**

---

## 📋 Table of Contents

- [Quick Start](#quick-start)
- [BBR Installation](#bbr-installation)
- [TCP Optimization](#tcp-optimization)
- [Routing Optimization](#routing-optimization)
- [Benchmark Tools](#benchmark-tools)
- [FAQ](#faq)

---

## 🚀 Quick Start

One-command BBR installation:

```bash
wget -qO- https://raw.githubusercontent.com/CG-spring/vps-network-tuning/main/install-bbr.sh | bash
```

Or manually:

```bash
# Check current TCP congestion control
sysctl net.ipv4.tcp_congestion_control

# Enable BBR (requires kernel 4.9+)
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

---

## 🔧 BBR Installation

### Method 1: Official BBR (Kernel 4.9+)

```bash
# Add to /etc/sysctl.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# Apply
sysctl -p

# Verify
sysctl net.ipv4.tcp_congestion_control
# Output: net.ipv4.tcp_congestion_control = bbr
```

### Method 2: BBR Plus (Better for high latency)

```bash
# Download and install BBR Plus kernel
wget https://github.com/cx9208/bbrplus/raw/master/ok_bbrplus_centos.sh
chmod +x ok_bbrplus_centos.sh
./ok_bbrplus_centos.sh

# Enable after reboot
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbrplus" >> /etc/sysctl.conf
sysctl -p
```

### Method 3: LotServer (Commercial, best performance)

```bash
# One-click install
wget --no-check-certificate -qO /tmp/appex.sh "https://raw.githubusercontent.com/0oVicero0/serverSpeeder_Install/master/appex.sh" && bash /tmp/appex.sh
```

---

## ⚙️ TCP Optimization

### Recommended sysctl.conf

```bash
# File: /etc/sysctl.conf

# TCP memory settings
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# Connection tracking
net.netfilter.nf_conntrack_max = 655350
net.netfilter.nf_conntrack_tcp_timeout_established = 1200

# TCP optimization
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_mtu_probing = 1

# Enable BBR
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

Apply with: `sysctl -p`

---

## 🌐 Routing Optimization

### Test current routing

```bash
# Install besttrace
wget https://github.com/zq/besttrace/releases/download/v1.3/besttrace-linux-amd64 -O besttrace
chmod +x besttrace

# Test to China Telecom
./besttrace -q 1 -g cn 202.96.209.133

# Test to China Unicom
./besttrace -q 1 -g cn 221.12.1.227

# Test to China Mobile
./besttrace -q 1 -g cn 211.136.112.50
```

### Common optimization methods

| Method | Use Case | Difficulty |
|--------|----------|------------|
| BGP Optimized Route | Best for China | Easy (buy) |
| CN2 GIA | Premium China route | Easy (buy) |
| 9929/10099 | Budget China route | Easy (buy) |
| BGP Hijack | Advanced users | Hard |

---

## 📊 Benchmark Tools

### Speed Test

```bash
# Install speedtest-cli
apt install -y speedtest-cli

# Run test
speedtest-cli
```

### Network Quality Test

```bash
# Install bench.sh
wget -qO- bench.sh | bash

# Or use this comprehensive script
wget -qO- https://raw.githubusercontent.com/teddysun/across/master/bench.sh | bash
```

### Latency Test to China

```bash
# Install pingtest
pip install ping3

# Test script
python3 << 'EOF'
from ping3 import ping
import statistics

hosts = {
    'Beijing CT': '202.96.209.133',
    'Shanghai CT': '202.96.209.5',
    'Guangzhou CT': '202.96.128.86',
    'Beijing CU': '221.12.1.227',
    'Shanghai CU': '221.12.33.227',
}

for name, ip in hosts.items():
    times = [ping(ip, timeout=2) for _ in range(5) if ping(ip, timeout=2)]
    if times:
        avg = statistics.mean(times) * 1000
        print(f"{name}: {avg:.1f}ms")
    else:
        print(f"{name}: Timeout")
EOF
```

---

## ❓ FAQ

**Q: Which BBR version should I use?**
A: Start with official BBR. If you have high latency (>150ms) or packet loss, try BBR Plus.

**Q: Will BBR increase my speed?**
A: BBR improves throughput on high-latency or lossy networks. On clean, low-latency networks, improvement may be minimal.

**Q: Is BBR safe?**
A: Yes, BBR is developed by Google and included in Linux kernel 4.9+.

**Q: Why is my VPS slow to China?**
A: Common causes: 1) Poor routing, 2) No TCP optimization, 3) Oversold VPS, 4) Network congestion.

---

## 🔗 Related Resources

- [VPSVIP](https://vpsvip.net) - VPS recommendations
- [ClashVIP](https://clashvip.net) - Clash tutorials
- [ClashHub](https://clashhub.net) - Rules and configs

---

## 📄 License

MIT License - See [LICENSE](LICENSE)

---

**Last Updated**: 2026-04-08
