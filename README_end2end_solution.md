# VPN Setup Guide with Traffic Obfuscation

This guide describes how to set up a private VPN system that disguises traffic as normal HTTPS connections, suitable for bypassing deep packet inspection (DPI).

## Architecture Overview

```
┌─────────────────┐      Disguised Traffic      ┌─────────────────┐      Raw Traffic      ┌─────────────────┐
│                 │  ─────────────────────────>  │                 │  ─────────────────>  │                 │
│  Client Device  │    (Looks like HTTPS/TLS)   │  VPN Server     │                      │  Target Server  │
│  (Clash Verge)  │                              │  (VPS/Droplet)  │                      │  (e.g., Netflix)│
│                 │  <─────────────────────────  │                 │  <─────────────────  │                 │
└─────────────────┘      Response Traffic        └─────────────────┘      Response       └─────────────────┘
```

## Prerequisites

- A VPS (e.g., DigitalOcean Droplet, Vultr, Linode)
  - Recommended: Ubuntu 22.04 LTS
  - Minimum: 1 vCPU, 1GB RAM
- A domain name (optional but recommended for TLS)
- Basic Linux command line knowledge

---

## Technology Overview: Xray vs Mihomo

This guide uses **Xray** on the server and **Mihomo** (via Clash Verge) on the client. Understanding their differences helps you make informed decisions.

### Core Comparison

| Aspect | Xray | Mihomo (Clash.Meta) |
|--------|------|---------------------|
| **Type** | Proxy platform core | Rule-based proxy client/core |
| **Config Format** | JSON | YAML |
| **Primary Role** | Server & client protocol core | Client-side traffic routing |
| **Developed By** | XTLS Team | MetaCubeX |

### Protocol Support

| Protocol | Xray | Mihomo |
|----------|------|--------|
| VLESS + Reality + Vision | ✅ **Native, best support** | ✅ Supported (limited) |
| XTLS Technology | ✅ **Original creator** | ⚠️ Basic support |
| VMess | ✅ | ✅ |
| Shadowsocks / Trojan | ✅ | ✅ |
| Hysteria2 / TUIC | ❌ Not natively | ✅ Supported |
| Restls-V1 | ❌ | ✅ |

### Strengths Comparison

**Xray Advantages:**
- First to implement new protocols (Reality, Vision, XTLS)
- Best VLESS+Reality performance and stability
- Ideal for server deployment
- More protocol customization options
- Native XTLS with optimal throughput

**Mihomo Advantages:**
- Powerful rule-based routing system
- Rule-providers for dynamic rule updates
- Rich GUI ecosystem (Clash Verge, Mihomo Party)
- Lightweight, lower resource usage
- Better subscription/airport service support
- TUN mode built-in
- Script override support

### Popular Clients

| Core | Desktop Client | Mobile Client |
|------|----------------|---------------|
| **Xray** | v2rayN, NekoRay | v2rayNG, NekoBox |
| **Mihomo** | Clash Verge Rev, Mihomo Party | Clash Meta for Android |

### Use Case Recommendations

| Use Case | Recommended Core |
|----------|------------------|
| Server deployment | **Xray** |
| Client device (with GUI) | **Mihomo** (via Clash Verge) |
| VLESS+Reality+Vision | **Xray** (best compatibility) |
| Subscription/airport services | **Mihomo** |
| Latest protocol features | **Xray** (adopts faster) |
| Mobile/low-power devices | **Mihomo** (lighter weight) |
| Complex routing rules | **Mihomo** |

### Recommended Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        RECOMMENDED SETUP                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   SERVER (VPS)              CLIENT (Your Device)                │
│   ┌─────────────┐           ┌─────────────────────┐            │
│   │             │           │   Clash Verge Rev   │            │
│   │    Xray     │◄────────► │   (Mihomo Core)     │            │
│   │   Server    │           │                     │            │
│   │             │           │   - YAML Config     │            │
│   │ VLESS+REAL  │           │   - Rule Routing    │            │
│   └─────────────┘           │   - GUI Management  │            │
│                             └─────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

This combination gives you:
- **Best server performance** with Xray's native Reality implementation
- **Best client experience** with Clash Verge's user-friendly GUI and powerful routing

---

## Part 1: Server Setup

### 1.1 Initial Server Configuration

```bash
# Update system
apt update && apt upgrade -y

# Install necessary packages
apt install -y curl wget unzip

# Configure firewall
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 1.2 Install Xray Core

Xray is the core engine that handles protocol conversion and traffic obfuscation.

```bash
# Install Xray
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

# Verify installation
xray version
```

> **Why Xray for Server?** Xray provides the best VLESS+Reality implementation with native XTLS support, offering optimal performance and stability for server deployment. Mihomo is better suited for client-side use with its powerful routing capabilities.

### 1.3 Generate UUID and Keys

```bash
# Generate UUID for client authentication
xray uuid
# Example output: 5b3a1c2d-4e5f-6789-abcd-ef1234567890

# Generate X25519 key pair for Reality (recommended obfuscation method)
xray x25519
# Example output:
# Private key: qK7x2Y9zW3vN5mB8cD4fG6hJ1kL3pM7nQ9rT2vX5yZ8
# Public key:  aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0uV1wX2yZ3aB4c
```

### 1.4 Configure Xray Server

Create the configuration file at `/usr/local/etc/xray/config.json`:

#### Option A: VLESS + Reality (Recommended - No Domain Required)

Reality protocol makes traffic look like visiting a legitimate website without needing a domain or certificate.

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "YOUR_UUID_HERE",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": [
            "www.microsoft.com",
            "microsoft.com"
          ],
          "privateKey": "YOUR_PRIVATE_KEY_HERE",
          "shortIds": [
            "",
            "0123456789abcdef"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```

**Key Parameters:**
- `dest`: The website to impersonate (choose popular, legitimate sites)
- `serverNames`: Must match the dest website
- `privateKey`: Generated from `xray x25519` command
- `flow`: Must be `xtls-rprx-vision` for Reality

#### Option B: VLESS + WebSocket + TLS (Requires Domain)

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "YOUR_UUID_HERE"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/xray/cert.pem",
              "keyFile": "/etc/xray/key.pem"
            }
          ]
        },
        "wsSettings": {
          "path": "/your-secret-path"
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```

### 1.5 Start Xray Service

```bash
# Enable and start Xray
systemctl enable xray
systemctl start xray

# Check status
systemctl status xray

# View logs if needed
journalctl -u xray -f
```

---

## Part 2: Client Configuration (Clash Verge)

### 2.1 Install Clash Verge

Download from: https://github.com/clash-verge-rev/clash-verge-rev/releases

Or install via package manager:

```bash
# Windows (using winget)
winget install clasp-verge-rev.clash-verge-rev

# macOS (using Homebrew)
brew install --cask clash-verge-rev

# Linux (AUR)
yay -S clash-verge-rev-bin
```

### 2.2 Configure Proxy Profile

Create a YAML configuration file for Clash Verge:

#### For VLESS + Reality:

```yaml
mixed-port: 7890
allow-lan: true
mode: rule
log-level: info

proxies:
  - name: "VPN-Reality"
    type: vless
    server: YOUR_VPS_IP
    port: 443
    uuid: YOUR_UUID_HERE
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    reality-opts:
      public-key: YOUR_PUBLIC_KEY_HERE
      short-id: 0123456789abcdef
    client-fingerprint: chrome

proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - VPN-Reality
      - DIRECT

rules:
  # Streaming services
  - DOMAIN-SUFFIX,netflix.com,Proxy
  - DOMAIN-SUFFIX,nflxvideo.net,Proxy
  - DOMAIN-SUFFIX,nflxso.net,Proxy
  - DOMAIN-SUFFIX,netflix.net,Proxy
  - DOMAIN-SUFFIX,youtube.com,Proxy
  - DOMAIN-SUFFIX,googlevideo.com,Proxy

  # GeoIP and final rules
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

#### For VLESS + WebSocket + TLS:

```yaml
mixed-port: 7890
allow-lan: true
mode: rule
log-level: info

proxies:
  - name: "VPN-WSS"
    type: vless
    server: your-domain.com
    port: 443
    uuid: YOUR_UUID_HERE
    network: ws
    tls: true
    udp: true
    ws-opts:
      path: /your-secret-path
      headers:
        Host: your-domain.com

proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - VPN-WSS
      - DIRECT

rules:
  - DOMAIN-SUFFIX,netflix.com,Proxy
  - DOMAIN-SUFFIX,youtube.com,Proxy
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

### 2.3 Import Configuration

1. Open Clash Verge
2. Go to **Profiles** tab
3. Click **New** > **Local**
4. Select your YAML configuration file
5. Click on the profile to activate it

### 2.4 Enable System Proxy

1. In Clash Verge main interface, toggle **System Proxy**
2. Set proxy mode to **Rule** (for selective routing) or **Global** (all traffic through VPN)

---

## Part 3: Verification & Testing

### 3.1 Test Server Connectivity

```bash
# From client machine, test if server port is open
nc -zv YOUR_VPS_IP 443

# Or use telnet
telnet YOUR_VPS_IP 443
```

### 3.2 Test VPN Functionality

1. **Check IP Address**
   - Visit https://ifconfig.me or https://ip.sb
   - Should show your VPS IP, not your real IP

2. **Test DNS Leak**
   - Visit https://dnsleaktest.com
   - Should show DNS servers in your VPS region, not your ISP

3. **Test Streaming Services**
   - Access Netflix/YouTube
   - Content should reflect the VPS region

### 3.3 Debugging

```bash
# On server, check Xray logs
journalctl -u xray -f

# Check if Xray is listening on port 443
ss -tlnp | grep 443

# On client, check Clash Verge logs
# Located in: ~/.config/clash-verge/logs/
```

---

## Part 4: Advanced Configuration

### 4.1 Multiple Clients

Add additional clients in server config:

```json
"clients": [
  {
    "id": "UUID_FOR_CLIENT_1",
    "flow": "xtls-rprx-vision"
  },
  {
    "id": "UUID_FOR_CLIENT_2",
    "flow": "xtls-rprx-vision"
  }
]
```

Generate new UUIDs with `xray uuid` for each client.

### 4.2 Routing Rules Examples

```yaml
rules:
  # Direct connections for local services
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT

  # Block ads and trackers
  - DOMAIN-KEYWORD,admarvel,REJECT
  - DOMAIN-KEYWORD,admaster,REJECT
  - DOMAIN-SUFFIX,googlesyndication.com,REJECT

  # Route streaming through VPN
  - DOMAIN-SUFFIX,netflix.com,Proxy
  - DOMAIN-SUFFIX,disneyplus.com,Proxy
  - DOMAIN-SUFFIX,hbomax.com,Proxy
  - DOMAIN-SUFFIX,primevideo.com,Proxy

  # Default rules
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

### 4.3 Performance Optimization

Server-side `/usr/local/etc/xray/config.json`:

```json
{
  "log": { "loglevel": "warning" },
  "policy": {
    "system": {
      "statsInboundUplink": true,
      "statsInboundDownlink": true
    },
    "levels": {
      "0": {
        "handshake": 4,
        "connIdle": 300,
        "uplinkOnly": 2,
        "downlinkOnly": 5,
        "bufferSize": 10240
      }
    }
  }
  // ... rest of config
}
```

---

## Part 5: Security Hardening

### 5.1 Server Security

```bash
# Change SSH port (optional)
sed -i 's/#Port 22/Port YOUR_CUSTOM_PORT/' /etc/ssh/sshd_config

# Disable root login
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Restart SSH
systemctl restart sshd

# Configure fail2ban
apt install fail2ban -y
systemctl enable fail2ban
systemctl start fail2ban
```

### 5.2 Regular Maintenance

```bash
# Update Xray periodically
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

# Update system
apt update && apt upgrade -y

# Check logs for suspicious activity
journalctl -u xray --since "1 day ago" | grep -i error
```

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Cannot connect | Firewall blocking | Check `ufw status`, ensure port 443 is open |
| Connection timeout | Wrong server IP | Verify VPS IP in client config |
| TLS handshake failed | Reality config mismatch | Verify public key, server name match |
| Slow speeds | Server location | Choose VPS closer to target services |
| DNS leaks | DNS not through VPN | Enable DNS hijacking in Clash config |
| Netflix proxy error | IP blacklisted | Use different VPS provider/IP |

---

## Quick Reference

### Useful Commands

```bash
# Restart Xray
systemctl restart xray

# View real-time logs
journalctl -u xray -f

# Generate new UUID
xray uuid

# Generate new Reality keys
xray x25519

# Check port status
ss -tlnp | grep 443

# Test connection
curl -x http://127.0.0.1:7890 https://ifconfig.me
```

### Configuration File Locations

| Component | Path |
|-----------|------|
| Xray Server Config | `/usr/local/etc/xray/config.json` |
| Xray Logs | `journalctl -u xray` |
| Clash Verge Config (Windows) | `%USERPROFILE%\.config\clash-verge\` |
| Clash Verge Config (macOS/Linux) | `~/.config/clash-verge/` |

---

## Resources

### Official Documentation
- [Xray Documentation](https://xtls.github.io/)
- [Mihomo Documentation](https://wiki.metacubex.one/)
- [Reality Protocol Paper](https://github.com/XTLS/REALITY)
- [VLESS Protocol Specification](https://github.com/XTLS/Xray-core/blob/main/docs/config/proxies/vless.md)

### GitHub Repositories
- [Xray-core](https://github.com/XTLS/Xray-core) - Xray core engine
- [Mihomo](https://github.com/MetaCubeX/mihomo) - Mihomo (Clash.Meta) core
- [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev) - Desktop GUI client

### Related Projects
- [v2rayN](https://github.com/2dust/v2rayN) - Windows client (Xray-based)
- [Clash Meta for Android](https://github.com/MetaCubeX/ClashMetaForAndroid) - Android client
- [sing-box](https://github.com/SagerNet/sing-box) - Alternative universal proxy platform
