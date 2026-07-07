# Mail Server Setup Locally on PC
### Zimbra & Axigen Mail Server Deployment on accesswt.com
### Complete Setup, Configuration, Verification & Operations Guide

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Available Resources](#2-available-resources)
3. [Network & NAT Configuration (Proxmox Host)](#3-network--nat-configuration-proxmox-host)
4. [DNS Server Setup (ns1 / ns2)](#4-dns-server-setup-ns1--ns2)
5. [Axigen Mail Server Setup (Ubuntu 24)](#5-axigen-mail-server-setup-ubuntu-24)
6. [Zimbra Mail Server Setup (Rocky Linux 9)](#6-zimbra-mail-server-setup-rocky-linux-9)
7. [SSL/TLS Configuration](#7-ssltls-configuration)
8. [Axigen Hardening](#8-axigen-hardening)
9. [Multi-Tenant Client Provisioning](#9-multi-tenant-client-provisioning)
10. [Mail Flow — Detailed Chain Sequence](#10-mail-flow--detailed-chain-sequence)
    - [10.1 Lab Environment — Step List](#101-lab-environment-step-list)
    - [10.2 Production Environment — Step List)](#102-production-environment-step-list)
    - [10.3 Detailed Explanation — Lab Steps](#103-detailed-explanation--lab-steps)
    - [10.4 Detailed Explanation — Production Steps](#104-detailed-explanation--production-steps)
    - [10.5 DNS Record Types — Complete Reference](#105-dns-record-types-complete-reference)
    - [10.6 Port Reference Table](#106-port-reference-table)
    - [10.7 Mail Queuing — Complete Detail](#107-mail-queuing-complete-detail)
    - [10.8 TLS Handshake — Step by Step](#108-tls-handshake-step-by-step)
    - [10.9 Greylisting Flow](#109-greylisting-flow)
    - [10.10 SPF / DKIM / DMARC Validation Chain](#1010-spf--dkim--dmarc-validation-chain)
    - [10.11 Lab vs Production Comparison](#1011-lab-vs-production-comparison)
11. [Test Cases — All Tried Scenarios](#11-test-cases--all-tried-scenarios)
    - [11.1 SMTP Connectivity Tests](#111-smtp-connectivity-tests)
    - [11.2 Zimbra CLI Send Tests](#112-zimbra-cli-send-tests)
    - [11.3 Python SMTP Bulk Send Tests](#113-python-smtp-bulk-send-tests)
    - [11.4 Cross-Server Mail Flow Tests](#114-cross-server-mail-flow-tests)
    - [11.5 swaks Test](#115-swaks-test)
    - [11.6 mailutils Test (Failed — No local MTA)](#116-mailutils-test-failed--no-local-mta)
    - [11.7 Mail Transfer test ( Zimbra ~ Axigen)](#117-mail-transfer-test--zimbra--axigen)
    - [11.8 Update DKIM](#118-update-dkim)
12. [Bulk Mail Operations (Push & Delete)](#12-bulk-mail-operations-push--delete)
13. [Monitoring Commands](#13-monitoring-commands)
14. [Troubleshooting](#14-troubleshooting)
    - [14.1 Common Failure Points — Both Environments](#141-common-failure-points--both-environments)
    - [14.2 Zimbra-Specific Troubleshooting](#142-zimbra-specific-troubleshooting)
    - [14.3 Axigen-Specific Troubleshooting](#143-axigen-specific-troubleshooting)
    - [14.4 Step-by-Step Diagnostic Runbook](#144-step-by-step-diagnostic-runbook)
    - [14.5 Quick Reference Card](#145-quick-reference-card)
15. [Axigen Mail Operations — Complete Script Reference](#15-axigen-mail-operations--complete-script-reference)
16. [Bulk Operations — Full Scenario Walkthrough](#16-bulk-operations--full-scenario-walkthrough)
17. [Security Hardening Summary](#17-security-hardening-summary)
18. [Environment Summary — Final State](#18-environment-summary--final-state)

---

## 1. Infrastructure Overview

<img width="500" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/lab-mail-architecture.png">

```
Internet
    │
    ▼
ISP Router (wlo1: 192.168.1.3/27 or 172.18.0.0/23)
    │
    ▼
awtserver (Proxmox PVE 9.2.2)
├── vmbr1: 10.10.10.1/24  [Internal VM Bridge]
│
├── ns1.accesswt.com     10.10.10.106  [Primary DNS - Ubuntu]
├── ns2.accesswt.com     10.10.10.107  [Secondary DNS - Ubuntu]
├── ns3.accesswt.com     10.10.10.108  [Resolver DNS]
├── ns4.accesswt.com     10.10.10.109  [Resolver DNS]
├── zimbra.accesswt.com  10.10.10.110  [Zimbra 10.1.16 - Rocky Linux 9.8]
└── axigen.accesswt.com  10.10.10.111  [Axigen 10.6.35 - Ubuntu 24]
```
<img width="900" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/proxmox-hypervisor2.png">

### Domain & Client Assignment

| Mail Server | IP | Hosted Client Domains |
|---|---|---|
| Axigen | 10.10.10.111 | school.accesswt.com, fitness.accesswt.com |
| Zimbra | 10.10.10.110 | consultancy.accesswt.com, microfinance.accesswt.com, dairy.accesswt.com |

---

## 2. Available Resources

### Deployed : awtserver (Proxmox Host)
| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Storage | 100 GB SSD | 500 GB SSD |
| Network | 1 NIC (WiFi/wlo1) | 2 NICs (wired + WiFi) |
| OS | Proxmox PVE 7+ | PVE 9.2.2 |

### Zimbra VM (zimbra.accesswt.com)
| Resource | Minimum | Recommended | Actual Used |
|---|---|---|---|
| vCPU | 2 cores | 4 cores | 4 cores |
| RAM | 4 GB | 8 GB | 3.6 GB + 2 GB swap |
| Storage | 40 GB | 100 GB | 20 GB root |
| OS | RHEL 8/9, Rocky 9 | Rocky Linux 9.8 | Rocky Linux 9.8 |
| Zimbra Version | 10.x | 10.1.16 / 10.1.15 | 10.1.15 GA |

> **Note:** Zimbra installation is CPU/RAM intensive. Installation will fail or be interrupted on 2 vCPU / 2 GB RAM. Minimum 4 vCPU recommended for install phase.

### Axigen VM (axigen.accesswt.com)
| Resource | Minimum | Recommended | Actual Used |
|---|---|---|---|
| vCPU | 1 core | 2 cores | 2 cores |
| RAM | 1 GB | 2 GB | 2 GB |
| Storage | 20 GB | 50 GB | 20 GB |
| OS | Ubuntu 20.04+ | Ubuntu 24.04 LTS | Ubuntu 24.04 |
| Axigen Version | 10.x | 10.6.35 | 10.6.35 |

### DNS Servers (ns1/ns2)
| Resource | Minimum |
|---|---|
| vCPU | 1 core |
| RAM | 512 MB |
| Storage | 10 GB |
| OS | Ubuntu 22.04+ |
| Software | BIND9 |

---

## 3. Network & NAT Configuration (Proxmox Host)

### 3.1 Network Interface Layout

**File:** `/etc/network/interfaces` on `awtserver`

The Proxmox host uses:
- `wlo1` — WiFi uplink to LAN (192.168.1.3/27), provides internet access
- `vmbr1` — Internal bridge (10.10.10.1/24), connects all VMs
- NAT MASQUERADE — allows VMs to reach the internet through wlo1
- DNAT rules — port-forward external access into specific VMs

### 3.2 Complete interfaces file

```bash
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o wlo1 -j MASQUERADE

    # SSH port forwards to VMs
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2108 -j DNAT --to-destination 10.10.10.108:22
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2109 -j DNAT --to-destination 10.10.10.109:22
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2110 -j DNAT --to-destination 10.10.10.110:22
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2111 -j DNAT --to-destination 10.10.10.111:22

    # Axigen WebAdmin HTTP/HTTPS
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9000 -j DNAT --to-destination 10.10.10.111:9000
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9443 -j DNAT --to-destination 10.10.10.111:9443

    # Axigen Webmail & Mail ports
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 443 -j DNAT --to-destination 10.10.10.111:443
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 25 -j DNAT --to-destination 10.10.10.111:25
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 587 -j DNAT --to-destination 10.10.10.111:587
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 993 -j DNAT --to-destination 10.10.10.111:993

    # Zimbra Webmail & Admin
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 8443 -j DNAT --to-destination 10.10.10.110:8443
    post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 7071 -j DNAT --to-destination 10.10.10.110:7071
```

### 3.3 Apply rules immediately (without reboot)

```bash
# Apply all DNAT rules immediately on awtserver
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9000 -j DNAT --to-destination 10.10.10.111:9000
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9443 -j DNAT --to-destination 10.10.10.111:9443
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 443  -j DNAT --to-destination 10.10.10.111:443
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 25   -j DNAT --to-destination 10.10.10.111:25
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 587  -j DNAT --to-destination 10.10.10.111:587
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 993  -j DNAT --to-destination 10.10.10.111:993
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 8443 -j DNAT --to-destination 10.10.10.110:8443
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 7071 -j DNAT --to-destination 10.10.10.110:7071
```

### 3.4 Verification

```bash
# Verify NAT rules are active
iptables -t nat -L PREROUTING -n --line-numbers | grep -E '111|110'

# Verify IP forwarding is on
cat /proc/sys/net/ipv4/ip_forward
# Expected output: 1

# Test Axigen reachable from host
curl -sk https://10.10.10.111:9443 | grep -i "axigen"

# Test Zimbra reachable from host
curl -sk https://10.10.10.110:8443 | grep -i "zimbra"
```

<img width="900" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/iptables-dnat-rules.png">

---

## 4. DNS Server Setup (ns1 / ns2)

### 4.1 Server Assignments

| Server | IP | Role |
|---|---|---|
| ns1.accesswt.com | 10.10.10.106 | Primary (Master) authoritative |
| ns2.accesswt.com | 10.10.10.107 | Secondary (Slave) authoritative |
| ns3.accesswt.com | 10.10.10.108 | Resolver |
| ns4.accesswt.com | 10.10.10.109 | Resolver |

### 4.2 BIND9 named.conf.options (ns1)

```bash
options {
    directory "/var/cache/bind";
    recursion no;
    allow-query { 10.10.10.0/24; };
    listen-on { 10.10.10.106; };
    dnssec-validation auto;
    listen-on-v6 { none; };
};
```

### 4.3 named.conf.local — Zone Declarations (ns1)

```bash
key "transfer-key" {
    algorithm hmac-sha256;
    secret "wRyVUMMk6/Z73d9HXlUNTllK/+/3FmBPB5NfIa49XX8=";
};

key "ddns-key" {
    algorithm hmac-sha256;
    secret "XndcFlx9OmGC24jD/xhRmxaZFi0DKm5njLt3QHMuhD4=";
};

zone "accesswt.com" {
    type master;
    file "/etc/bind/zones/db.accesswt.com";
    allow-transfer { key "transfer-key"; };
    update-policy { grant ddns-key zonesub ANY; };
    also-notify { 10.10.10.107; 10.10.10.108; };
    notify yes;
};

zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.10.10";
    allow-transfer { key "transfer-key"; };
    update-policy { grant ddns-key zonesub ANY; };
    also-notify { 10.10.10.107; 10.10.10.108; };
    notify yes;
};
```

### 4.4 Forward Zone File — db.accesswt.com

```dns
$ORIGIN .
$TTL 604800
accesswt.com    IN SOA  ns1.accesswt.com. admin.accesswt.com. (
                        2026063002 ; serial
                        604800     ; refresh
                        86400      ; retry
                        2419200    ; expire
                        604800     ; minimum
                        )
                NS  ns1.accesswt.com.
                NS  ns2.accesswt.com.
                NS  ns3.accesswt.com.
                NS  ns4.accesswt.com.
                MX  10 mail1.accesswt.com.
                MX  20 mail2.accesswt.com.
                TXT "v=spf1 mx ip4:10.10.10.180 ip4:10.10.10.181 ~all"

$ORIGIN accesswt.com.
_dmarc          TXT  "v=DMARC1; p=quarantine"
axigen           A   10.10.10.111
zimbra           A   10.10.10.110
ns1              A   10.10.10.106
ns2              A   10.10.10.107
ns3              A   10.10.10.108
ns4              A   10.10.10.109
mail1            A   10.10.10.180
mail2            A   10.10.10.181

; ── Client subdomains (Axigen hosted) ──
school           A   10.10.10.111
fitness          A   10.10.10.111

; ── Client subdomains (Zimbra hosted) ──
consultancy      A   10.10.10.110
microfinance     A   10.10.10.110
dairy            A   10.10.10.110

; ── MX records ──
school.accesswt.com.        IN MX 10 axigen.accesswt.com.
fitness.accesswt.com.       IN MX 10 axigen.accesswt.com.
consultancy.accesswt.com.   IN MX 10 zimbra.accesswt.com.
microfinance.accesswt.com.  IN MX 10 zimbra.accesswt.com.
dairy.accesswt.com.         IN MX 10 zimbra.accesswt.com.

; ── SPF records ──
school.accesswt.com.        IN TXT "v=spf1 mx ip4:10.10.10.111 ~all"
fitness.accesswt.com.       IN TXT "v=spf1 mx ip4:10.10.10.111 ~all"
consultancy.accesswt.com.   IN TXT "v=spf1 mx ip4:10.10.10.110 ~all"
microfinance.accesswt.com.  IN TXT "v=spf1 mx ip4:10.10.10.110 ~all"
dairy.accesswt.com.         IN TXT "v=spf1 mx ip4:10.10.10.110 ~all"

; ── DMARC records ──
_dmarc.school.accesswt.com.       IN TXT "v=DMARC1; p=quarantine"
_dmarc.fitness.accesswt.com.      IN TXT "v=DMARC1; p=quarantine"
_dmarc.consultancy.accesswt.com.  IN TXT "v=DMARC1; p=quarantine"
_dmarc.microfinance.accesswt.com. IN TXT "v=DMARC1; p=quarantine"
_dmarc.dairy.accesswt.com.        IN TXT "v=DMARC1; p=quarantine"
```

### 4.5 Reverse Zone File — db.10.10.10

```dns
$ORIGIN .
$TTL 604800
10.10.10.in-addr.arpa   IN SOA  ns1.accesswt.com. admin.accesswt.com. (
                                2024010103 ; serial
                                604800 86400 2419200 604800 )
                        NS  ns1.accesswt.com.
                        NS  ns2.accesswt.com.
                        NS  ns3.accesswt.com.
                        NS  ns4.accesswt.com.

$ORIGIN 10.10.10.in-addr.arpa.
106     PTR     ns1.accesswt.com.
107     PTR     ns2.accesswt.com.
108     PTR     ns3.accesswt.com.
109     PTR     ns4.accesswt.com.
110     PTR     zimbra.accesswt.com.
111     PTR     axigen.accesswt.com.
180     PTR     mail1.accesswt.com.
181     PTR     mail2.accesswt.com.
```

> **PTR Note:** Subdomains (school, fitness, consultancy, microfinance, dairy) share IPs 10.10.10.110 and 10.10.10.111. PTR records point to the canonical server hostname used in SMTP EHLO banners. No additional PTR entries needed for client subdomains — this is correct and expected behavior.

### 4.6 Applying zone changes (Dynamic Zone — CRITICAL)

Since the zone has `update-policy` (DDNS), we cannot use `rndc reload` directly. Use freeze/thaw:

```bash
# Check zone syntax first
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
# Expected: OK

# Freeze → reload → thaw (dynamic zone safe workflow)
sudo rndc freeze accesswt.com
sudo rndc reload accesswt.com
sudo rndc thaw accesswt.com

# If reload fails after freeze (journal conflict), remove journal and restart
sudo systemctl stop bind9
sudo rm /etc/bind/zones/db.accesswt.com.jnl
sudo systemctl start bind9
```

### 4.7 DNS Verification Commands

```bash
# Verify MX records for all client domains
dig @10.10.10.106 school.accesswt.com MX
dig @10.10.10.106 fitness.accesswt.com MX
dig @10.10.10.106 consultancy.accesswt.com MX
dig @10.10.10.106 microfinance.accesswt.com MX
dig @10.10.10.106 dairy.accesswt.com MX

# Verify A records
dig @10.10.10.106 axigen.accesswt.com A
dig @10.10.10.106 zimbra.accesswt.com A

# Verify SPF
dig @10.10.10.106 school.accesswt.com TXT

# Verify PTR (reverse DNS)
dig @10.10.10.106 -x 10.10.10.111
dig @10.10.10.106 -x 10.10.10.110

# Verify propagation to ns2
dig @10.10.10.107 school.accesswt.com MX
```

*[INSERT SCREENSHOT: dig @10.10.10.106 school.accesswt.com MX showing ANSWER SECTION with axigen.accesswt.com MX 10]*

*[INSERT SCREENSHOT: dig @10.10.10.106 consultancy.accesswt.com MX showing zimbra.accesswt.com MX 10]*

---

## 5. Axigen Mail Server Setup (Ubuntu 24)

### 5.1 System Preparation

```bash
# Set hostname
sudo hostnamectl set-hostname axigen.accesswt.com
echo "10.10.10.111  axigen.accesswt.com  axigen" | sudo tee -a /etc/hosts

# Fix broken cdrom apt source
sudo nano /etc/apt/sources.list
# Remove/comment line: deb cdrom:[Ubuntu...]/ noble main restricted

# Update and install prerequisites
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl net-tools swaks python3
```

### 5.2 Download and Install Axigen

```bash
cd /tmp
sudo wget https://download.axigen.com/mail-server/axigen_10.6.35-1_amd64.deb
# Expected: 130.28M downloaded (200 OK)

sudo dpkg -i /tmp/axigen_10.6.35-1_amd64.deb
# Expected output:
# Thank we for installing AXIGEN Mail Server
# * Starting AXIGEN Mail Server... [ OK ]
```

*[INSERT SCREENSHOT: dpkg -i output showing "Starting AXIGEN Mail Server... OK"]*

### 5.3 Service Management

```bash
sudo systemctl enable axigen
sudo systemctl start axigen
sudo systemctl status axigen
# Expected: Active: active (exited) — axigen manages its own process model

# Verify Axigen process is running
ps aux | grep axigen
```

*[INSERT SCREENSHOT: systemctl status axigen showing active]*

### 5.4 Firewall Configuration

```bash
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 25/tcp    # SMTP
sudo ufw allow 587/tcp   # Submission
sudo ufw allow 993/tcp   # IMAPS
sudo ufw allow 995/tcp   # POP3S
sudo ufw allow 443/tcp   # Webmail HTTPS
sudo ufw allow 9000/tcp  # WebAdmin HTTP
sudo ufw allow 9443/tcp  # WebAdmin HTTPS
sudo ufw enable

# Verify
sudo ufw status
```

*[INSERT SCREENSHOT: ufw status showing all mail ports ALLOW]*

### 5.5 Port Verification

```bash
sudo ss -tlnp | grep axigen
# Expected — Axigen listening on:
# 0.0.0.0:25    (SMTP)
# 0.0.0.0:993   (IMAPS)
# 0.0.0.0:443   (Webmail)
# 0.0.0.0:9000  (WebAdmin HTTP)
# 0.0.0.0:9443  (WebAdmin HTTPS)
# 0.0.0.0:80    (HTTP redirect)
# 0.0.0.0:21    (FTP Backup — optional)
# 0.0.0.0:143   (IMAP plain)
# 0.0.0.0:465   (SMTPS)
```

*[INSERT SCREENSHOT: ss -tlnp | grep axigen output showing all listening ports]*

### 5.6 Initial WebAdmin Configuration

Access WebAdmin from LAN machine:

```
https://192.168.1.3:9443
```

> This works because the DNAT rule on awtserver forwards 192.168.1.3:9443 → 10.10.10.111:9443

**Initial Configuration Wizard — Primary Domain Screen:**

```
Primary Domain:             accesswt.com
Domain Location:            /var/opt/axigen/domains/  (default)
Postmaster Account Password: (set strong password)
```

Click CONTINUE → set Admin password → set hostname to `axigen.accesswt.com` → Finish.

*[INSERT SCREENSHOT: Axigen WebAdmin Initial Configuration wizard — Primary Domain screen]*

*[INSERT SCREENSHOT: Axigen WebAdmin Dashboard after successful initial configuration]*

### 5.7 Verify WebAdmin from VM and Host

```bash
# From axigen VM itself
curl -sk https://127.0.0.1:9443 | grep -i "axigen"
# Expected: <title>Axigen WebAdmin</title>

# From Proxmox host (awtserver)
curl -sk https://10.10.10.111:9443 | grep -i "axigen"
# Expected: same output
```

```bash
# From MacBook (192.168.1.x) — after DNAT applied on awtserver
curl -sk https://192.168.1.3:9443 | grep -i "axigen"
# Expected: Axigen WebAdmin HTML
```

*[INSERT SCREENSHOT: Browser showing https://192.168.1.3:9443 — Axigen WebAdmin login page]*

### 5.8 Current Running Services (as verified)

```
Queue Processing     ● Running
SMTP Receiving       ● Running
SMTP Sending         ● Running
POP3                 ✗ Stopped (disabled by design)
Remote POP           ✗ Stopped
IMAP                 ● Running
WebMail              ● Running
WebAdmin             ● Running (Manage via CLI)
CLI                  ● Running
Log                  ● Running
FTP Backup & Restore ● Running
Axigen TNEF Decoder  ● Running
```

*[INSERT SCREENSHOT: Axigen Services Management page showing running/stopped status]*

---

## 6. Zimbra Mail Server Setup (Rocky Linux 9)

### 6.1 OS Information

```
NAME="Rocky Linux"
VERSION="9.8 (Blue Onyx)"
ID="rocky"
ID_LIKE="rhel centos fedora"
SUPPORT_END="2032-05-31"
```

### 6.2 System Preparation

```bash
# Set hostname (critical for Zimbra)
sudo hostnamectl set-hostname zimbra.accesswt.com
echo "10.10.10.110  zimbra.accesswt.com  zimbra" >> /etc/hosts

# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config

# Disable firewalld temporarily (re-enable after install)
sudo systemctl disable --now firewalld

# Set timezone
sudo timedatectl set-timezone Asia/Kathmandu
sudo timedatectl set-ntp true

# Add swap (critical — prevents OOM kill during heavy install)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# Install screen (critical — prevents SSH drop killing install)
sudo dnf install -y screen net-tools wget curl perl tar
```

> **CRITICAL LESSON LEARNED:** The Zimbra install was interrupted 3 times because the SSH session dropped during the heavy `Starting servers...` phase. Using `screen` prevents this — the install continues even if SSH disconnects.

```bash
# Always start install inside screen
screen -S zimbra-install
# If SSH drops, reconnect with: screen -r zimbra-install
```

### 6.3 Download Zimbra (RHEL9 Compatible Build)

The official Zimbra 8.8.15 builds no longer exist at the original URLs and are not compatible with RHEL9/Rocky9. Use the community FOSS build:

```bash
cd /opt
sudo wget https://github.com/maldua/zimbra-foss/releases/download/zimbra-foss-build-rhel-9/10.1.16.p1/zcs-10.1.16_GA_4200001.RHEL9_64.20260310121522.tgz
# Expected: 225.46M downloaded
```

*[INSERT SCREENSHOT: wget showing 100% download complete for zcs-10.1.16...RHEL9_64.tgz]*

### 6.4 Extract and Install

```bash
sudo tar -xzvf zcs-10.1.16_GA_4200001.RHEL9_64.20260310121522.tgz
cd zcs-10.1.16_GA_4200001.RHEL9_64.20260310121522/

sudo ./install.sh --platform-override
```

**Installer prompts answered:**

```
License agreement:              Y
Use Zimbra's package repository: Y
Install zimbra-ldap:            Y
Install zimbra-logger:          Y
Install zimbra-mta:             Y
Install zimbra-dnscache:        N  (we have our own DNS)
Install zimbra-snmp:            Y
Install zimbra-store:           Y
Install zimbra-apache:          Y
Install zimbra-spell:           Y
Install zimbra-memcached:       Y
Install zimbra-proxy:           Y
Install p7zip-plugins:          Y
The system will be modified. Continue: Y
```

**DNS warning during install:**
```
DNS ERROR resolving MX for zimbra.accesswt.com
Change domain name? [Yes] No
```
> This is expected — our internal DNS resolves it correctly; the installer's DNS check uses external resolvers which can't see our internal zone.

**Setting Admin Password (Store Configuration menu):**
```
Select 6 → zimbra-store → Select 4 → Admin Password
Password: (set password — minimum 6 characters)
Press r → return to main menu
Press a → apply configuration
Save config: Yes
The system will be modified: Yes
```

*[INSERT SCREENSHOT: Zimbra installer Main Menu showing all components Enabled]*

*[INSERT SCREENSHOT: Zimbra installer — CONFIGURATION COMPLETE screen before pressing 'a']*

### 6.5 Verify Installation Complete

```bash
sudo su - zimbra
zmcontrol status
```

**Expected — All services Running:**
```
Host zimbra.accesswt.com
    amavis          Running
    antispam        Running
    antivirus       Running
    ldap            Running
    logger          Running
    mailbox         Running
    memcached       Running
    mta             Running
    opendkim        Running
    proxy           Running
    service webapp  Running
    snmp            Running
    spell           Running
    stats           Running
    zimbra webapp   Running
    zimbraAdmin webapp Running
    zimlet webapp   Running
    zmconfigd       Running
```

*[INSERT SCREENSHOT: zmcontrol status showing all services Running]*

### 6.6 Resource Check After Install

```bash
free -h
# Output:
#            total   used    free    shared  buff/cache  available
# Mem:       3.6Gi   2.3Gi   1.1Gi   14Mi    358Mi       1.2Gi
# Swap:      2.0Gi   1.2Gi   824Mi

nproc
# Output: 4

uptime
# load average: 0.30, 0.53, 0.64

dmesg | grep -i "killed process\|oom"
# Expected: (empty — no OOM kills)
```

### 6.7 Firewall Rules (Post-Install)

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port={25,80,110,143,443,465,587,993,995,7071,8443}/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

### 6.8 Access Zimbra

```
Webmail:        https://192.168.1.3:8443
Admin Console:  https://192.168.1.3:7071
Login:          admin@zimbra.accesswt.com
```

*[INSERT SCREENSHOT: Zimbra Admin Console login page at https://192.168.1.3:7071]*

*[INSERT SCREENSHOT: Zimbra Admin Console Dashboard after login]*

---

## 7. SSL/TLS Configuration

### 7.1 Axigen — Generate Self-Signed Certificate

```bash
# Create SSL directory
sudo mkdir -p /var/opt/axigen/ssl
cd /var/opt/axigen/ssl

# Generate 2-year self-signed certificate
sudo openssl req -x509 -nodes -days 730 -newkey rsa:2048 \
  -keyout axigen.key \
  -out axigen.crt \
  -subj "/C=NP/ST=Bagmati/L=Kathmandu/O=AccessWT/CN=axigen.accesswt.com"

# Set ownership
sudo chown axigen:axigen axigen.key axigen.crt
sudo chmod 600 axigen.key

# Verify certificate
openssl x509 -in axigen.crt -noout -subject -dates
# Expected:
# subject=C=NP, ST=Bagmati, L=Kathmandu, O=AccessWT, CN=axigen.accesswt.com
# notBefore=Jun 30 06:00:19 2026 GMT
# notAfter=Jun 29 06:00:19 2028 GMT
```

*[INSERT SCREENSHOT: openssl x509 output showing CN=axigen.accesswt.com and 2-year validity]*

### 7.2 Combine Certificate and Key for Axigen

Axigen expects cert+key in a single combined PEM file:

```bash
# Combine cert + key correctly using sudo bash -c to avoid permission issues
sudo bash -c 'cat /var/opt/axigen/ssl/axigen.crt /var/opt/axigen/ssl/axigen.key > /var/opt/axigen/axigenmail.pem'
sudo chown axigen:axigen /var/opt/axigen/axigenmail.pem
sudo chmod 600 /var/opt/axigen/axigenmail.pem

# Verify both sections present
sudo grep -c "BEGIN CERTIFICATE" /var/opt/axigen/axigenmail.pem  # should be 1
sudo grep -c "BEGIN PRIVATE KEY" /var/opt/axigen/axigenmail.pem  # should be 1
```

### 7.3 Import Certificate in Axigen WebAdmin

```
WebAdmin → Security & Filtering → SSL Certificates → + ADD
```

The certificate `axigen.accesswt.com` appears in the list, issued by "AccessWT", expiring Jun 29 2028.

*[INSERT SCREENSHOT: Axigen SSL Certificates page showing both "axigen" (old) and "axigen.accesswt.com" (new) entries]*

### 7.4 Verify Certificate on Each Port

```bash
# Check IMAPS (993)
openssl s_client -connect 10.10.10.111:993 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Check WebAdmin (9443)
openssl s_client -connect 10.10.10.111:9443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

### 7.5 Zimbra — Built-in SSL

Zimbra automatically generates and manages its own SSL certificates during installation (self-signed CA). These are applied to all services automatically:

```bash
# Verify Zimbra SSL cert
openssl s_client -connect 10.10.10.110:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
# Expected: CN=zimbra.accesswt.com
```

---

## 8. Axigen Hardening

### 8.1 Restrict WebAdmin to Internal Networks Only

```bash
# Remove open WebAdmin rule
sudo ufw delete allow 9443/tcp

# Allow only from management networks
sudo ufw allow from 10.10.10.0/24 to any port 9443 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 9443 proto tcp
```

### 8.2 Test for Open Relay (Most Critical Security Check)

```bash
# Test from Axigen VM itself
telnet 10.10.10.111 25
EHLO test
MAIL FROM:<test@external.com>
RCPT TO:<anyone@gmail.com>
# Expected: should be REJECTED — not accepted for relay to external domain
# If accepted: OPEN RELAY — fix immediately in WebAdmin → SMTP → Relay Restrictions
```

### 8.3 Greylisting (Confirmed Active)

During testing, greylisting correctly triggered:
```
RCPT TO:<admin@school.accesswt.com>
250 Recipient accepted
DATA
451 Temporary rejected by greylisting
```

On retry, it passed:
```
DATA
354 Ready to receive data
250 Mail queued for delivery
```

This proves greylisting is working as anti-spam protection.

### 8.4 System Hardening

```bash
# Install fail2ban
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# Disable root SSH login
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Backup Axigen config before changes
sudo tar -czf /root/axigen-config-backup-$(date +%F).tar.gz /var/opt/axigen/
```

### 8.5 Ports to Disable (if not using POP3)

```bash
sudo ufw delete allow 995/tcp   # POP3S
sudo ufw delete allow 9000/tcp  # WebAdmin HTTP (keep only HTTPS 9443)
```

---

## 9. Multi-Tenant Client Provisioning

### 9.1 Client Domain Summary

| Server | Client | Domain | Users |
|---|---|---|---|
| Axigen | School | school.accesswt.com | admin, info, principal |
| Axigen | Fitness | fitness.accesswt.com | admin, info, membership |
| Zimbra | Consultancy | consultancy.accesswt.com | admin, info, user1-8 |
| Zimbra | Microfinance | microfinance.accesswt.com | admin, info, user1-8 |
| Zimbra | Dairy | dairy.accesswt.com | admin, info |

### 9.2 Axigen — Add Client Domains (WebAdmin)

```
WebAdmin → DOMAINS & ACCOUNTS → Manage Domains → + ADD

Domain 1: school.accesswt.com    → /var/opt/axigen/domains/
Domain 2: fitness.accesswt.com   → /var/opt/axigen/domains/
```

### 9.3 Axigen — Create Accounts (WebAdmin)

```
WebAdmin → DOMAINS & ACCOUNTS → Manage Accounts → + ADD ACCOUNT

school.accesswt.com:
  admin       → Account type: Premium
  info        → Account type: Premium
  principal   → Account type: Premium

fitness.accesswt.com:
  admin       → Account type: Premium
  info        → Account type: Premium
  membership  → Account type: Premium
```

*[INSERT SCREENSHOT: Axigen Manage Accounts showing school.accesswt.com with admin, info, principal listed]*

*[INSERT SCREENSHOT: Axigen Manage Accounts showing fitness.accesswt.com accounts]*

### 9.4 Zimbra — Add Client Domains (Admin Console GUI)

```
Admin Console → Configure → Domains → New

Domain 1: consultancy.accesswt.com  → Status: Active
Domain 2: microfinance.accesswt.com → Status: Active
Domain 3: dairy.accesswt.com        → Status: Active (added as third client)
```

*[INSERT SCREENSHOT: Zimbra Admin Console — Configure → Domains showing consultancy, microfinance, dairy, zimbra.accesswt.com all Active]*

### 9.5 Zimbra — Create Initial Users (CLI)

```bash
su - zimbra

# Consultancy
zmprov ca admin@consultancy.accesswt.com 'Cons@dm1n2026!' displayName 'Consultancy Admin'
zmprov ca info@consultancy.accesswt.com 'Cons1nf02026!' displayName 'Consultancy Info'

# Microfinance
zmprov ca admin@microfinance.accesswt.com 'Micr0Adm1n2026!' displayName 'Microfinance Admin'
zmprov ca info@microfinance.accesswt.com 'Micr0Inf02026!' displayName 'Microfinance Info'

# Dairy (created via GUI — also reproducible via CLI)
zmprov ca admin@dairy.accesswt.com 'admin123' displayName 'Dairy Admin'
zmprov ca info@dairy.accesswt.com 'info123' displayName 'Dairy Info'
```

### 9.6 Zimbra — Bulk Create Dummy Users (Script)

```bash
cat > /tmp/create_dummy_users.sh << 'SCRIPT'
#!/bin/bash
DOMAINS=("consultancy.accesswt.com" "microfinance.accesswt.com")
PASSWORD="DummyP@ss2026!"

for DOMAIN in "${DOMAINS[@]}"; do
  for i in $(seq 1 8); do
    USER="user${i}@${DOMAIN}"
    NAME="Dummy User ${i}"
    echo "Creating ${USER}..."
    zmprov ca "${USER}" "${PASSWORD}" displayName "${NAME}"
  done
done
SCRIPT

chmod +x /tmp/create_dummy_users.sh
/tmp/create_dummy_users.sh
```

### 9.7 Verify All Zimbra Accounts

```bash
zmprov -l gaa
# Expected output — 28 accounts total:
# admin@zimbra.accesswt.com
# testuser@zimbra.accesswt.com
# galsync@dairy.accesswt.com
# admin@consultancy.accesswt.com
# info@consultancy.accesswt.com
# admin@microfinance.accesswt.com
# info@microfinance.accesswt.com
# admin@dairy.accesswt.com
# info@dairy.accesswt.com
# user1-8@consultancy.accesswt.com (8 users)
# user1-8@microfinance.accesswt.com (8 users)
# + spam/ham/virus-quarantine system accounts
```

*[INSERT SCREENSHOT: zmprov -l gaa output showing all 28+ accounts across all domains]*

---


## 10. Mail Flow — Detailed Chain Sequence

> **Scenario used throughout:** `info@consultancy.accesswt.com` (Zimbra, 10.10.10.110) sends to `admin@school.accesswt.com` (Axigen, 10.10.10.111)

---

### 10.1 Lab Environment — Step List

Every line below is one discrete, traceable step in chronological order.

#### Sender Side — Browser to Zimbra

```
S-01  User opens browser, navigates to https://192.168.1.3:8443
S-02  Browser initiates TCP connection to 192.168.1.3:8443
S-03  Browser performs TLS ClientHello with 192.168.1.3:8443
S-04  Proxmox awtserver receives packet on wlo1 (192.168.1.3)
S-05  iptables PREROUTING DNAT rule fires:
        192.168.1.3:8443 rewritten to 10.10.10.110:8443
S-06  Kernel routes packet via vmbr1 bridge to Zimbra VM
S-07  Zimbra nginx proxy receives HTTPS on 10.10.10.110:8443
S-08  nginx performs TLS handshake, presents Zimbra self-signed certificate
S-09  User accepts certificate warning (self-signed in lab)
S-10  nginx terminates TLS, decrypts request payload
S-11  nginx forwards plain HTTP internally to Zimbra Jetty on 127.0.0.1:8080
S-12  Zimbra Jetty (mailbox service) receives HTTP request
S-13  Jetty validates ZM_AUTH_TOKEN session cookie
S-14  Zimbra webmail UI loads in browser
S-15  User composes email: To, Subject, Body
S-16  User clicks Send
S-17  Browser sends HTTPS POST /service/soap/SendMsgRequest to 192.168.1.3:8443
S-18  POST tunnels through DNAT to Zimbra Jetty at 127.0.0.1:8080
```

#### Zimbra MTA Processing

```
S-19  Jetty validates recipient address format
S-20  Jetty generates unique Message-ID header
S-21  Jetty saves copy of message to Sent folder via internal LMTP (port 7025)
S-22  Jetty passes outgoing message to Postfix via /opt/zimbra/common/sbin/sendmail
S-23  Postfix pickup daemon collects message from sendmail pipe
S-24  Postfix cleanup daemon adds/normalizes headers (Date, Message-ID, etc.)
S-25  Postfix qmgr assigns Queue ID
S-26  qmgr places message in ACTIVE queue
S-27  qmgr checks: is school.accesswt.com in mydestination?          NO
S-28  qmgr checks: is school.accesswt.com in relay_domains?          NO
S-29  qmgr checks: is school.accesswt.com in virtual_mailbox_domains? NO
S-30  Decision: REMOTE delivery required — hand to smtp transport
S-31  Postfix smtp client process starts for this delivery attempt
```

#### DNS Resolution

```
S-32  Postfix smtp queries DNS resolver: school.accesswt.com MX?
S-33  Query sent UDP port 53 to 10.10.10.108 (ns3 resolver)
S-34  ns3 checks local cache — NOT cached
S-35  ns3 forwards query to ns1 (10.10.10.106) authoritative server
S-36  ns1 receives query
S-37  ns1 reads /etc/bind/zones/db.accesswt.com zone file
S-38  ns1 finds: school.accesswt.com MX 10 axigen.accesswt.com
S-39  ns1 returns answer to ns3
S-40  ns3 caches the MX answer with TTL=604800
S-41  ns3 returns MX answer to Postfix
S-42  Postfix receives: MX priority 10, hostname axigen.accesswt.com
S-43  Postfix queries DNS again: axigen.accesswt.com A?
S-44  ns3 checks cache — may be cached
S-45  ns1 returns: axigen.accesswt.com A 10.10.10.111
S-46  Postfix now has delivery target: 10.10.10.111 port 25
```

#### Network Layer — Zimbra to Axigen

```
S-47  Postfix smtp opens TCP socket to 10.10.10.111:25
S-48  TCP SYN packet sent from 10.10.10.110:random to 10.10.10.111:25
S-49  Both IPs are on vmbr1 bridge (10.10.10.0/24) — no NAT needed
S-50  Axigen responds with TCP SYN-ACK
S-51  Zimbra sends TCP ACK — 3-way handshake complete
S-52  TCP session established
```

#### SMTP Handshake

```
S-53  Axigen sends: 220 axigen.accesswt.com Axigen ESMTP ready
S-54  Zimbra sends: EHLO zimbra.accesswt.com
S-55  Axigen responds 250 with capability list:
        250-PIPELINING
        250-AUTH PLAIN LOGIN CRAM-MD5 DIGEST-MD5 GSSAPI
        250-8BITMIME / BINARYMIME / CHUNKING
        250-SIZE 10485760
        250-STARTTLS
        250 OK
S-56  Zimbra detects STARTTLS in capabilities
S-57  Zimbra sends: STARTTLS
S-58  Axigen responds: 220 Go ahead
S-59  TLS ClientHello sent by Zimbra (TLS 1.2/1.3 offered)
S-60  TLS ServerHello sent by Axigen (version + cipher selected)
S-61  Axigen sends Certificate (CN=axigen.accesswt.com, self-signed in lab)
S-62  TLS key exchange performed (ECDHE or RSA)
S-63  Both sides send ChangeCipherSpec
S-64  Both sides send Finished (encrypted handshake hash)
S-65  TLS session fully established — all further data encrypted
S-66  Zimbra sends EHLO again (required after STARTTLS — RFC 3207)
S-67  Axigen responds 250 capabilities (fresh, over TLS)
```

#### SMTP Envelope

```
S-68  Zimbra sends: MAIL FROM:<info@consultancy.accesswt.com>
S-69  Axigen validates sender address format
S-70  Axigen responds: 250 Sender accepted
S-71  Zimbra sends: RCPT TO:<admin@school.accesswt.com>
S-72  Axigen checks: is admin@school.accesswt.com a valid local account? YES
S-73  Axigen checks greylisting: is sender+recipient+IP tuple new?
S-74a IF new tuple: Axigen responds 451 Temporary rejected by greylisting
S-74b Zimbra Postfix receives 451 (temporary failure) — queues for retry
S-74c Postfix places message in DEFERRED queue
S-74d Postfix retries after ~5 minutes
S-74e On retry: greylisted tuple is now whitelisted — passes
S-75  Axigen responds: 250 Recipient accepted
S-76  Zimbra sends: DATA
S-77  Axigen responds: 354 Ready to receive data; remember CRLF.CRLF
```

#### Message Transfer

```
S-78  Zimbra transmits full RFC 5322 message over TLS:
        Received: from zimbra.accesswt.com ...
        Message-ID: <20260701.070201.abc123@zimbra.accesswt.com>
        From: info@consultancy.accesswt.com
        To: admin@school.accesswt.com
        Subject: Confidential
        Date: Tue, 01 Jul 2026 07:02:01 +0545
        MIME-Version: 1.0
        Content-Type: text/plain; charset=UTF-8
        [blank line — header/body separator — REQUIRED]
        [message body text]
S-79  Zimbra sends: . (single dot on its own line — end of DATA)
S-80  Axigen responds: 250 Mail queued for delivery
S-81  Zimbra sends: QUIT
S-82  Axigen responds: 221 Good bye
S-83  TCP FIN — connection gracefully closed
S-84  Zimbra Postfix logs: status=sent (250 Mail queued for delivery)
S-85  Message removed from Postfix active queue
```

#### Axigen Internal Processing

```
S-86  Axigen Queue Processing picks up message from incoming queue
S-87  Axigen checks SPF: does 10.10.10.110 match SPF for consultancy.accesswt.com?
        DNS lookup: consultancy.accesswt.com TXT
        Record: v=spf1 mx ip4:10.10.10.110 ~all
        Result: PASS
S-88  Axigen antispam engine analyzes headers and body
S-89  Antispam score calculated — below threshold
S-90  Axigen antivirus engine scans body and any attachments
S-91  Antivirus result: CLEAN
S-92  Axigen resolves admin@school.accesswt.com to local mailbox path
S-93  Axigen checks mailbox quota — under limit
S-94  Axigen writes .eml file to:
        /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX/
S-95  Axigen updates mailbox folder index and message count
S-96  Axigen logs successful delivery with timestamp and Message-ID
```

#### Receiver Side — Axigen to Browser

```
S-97   Recipient opens browser, navigates to https://192.168.1.3:443
S-98   Browser initiates TCP connection to 192.168.1.3:443
S-99   Browser performs TLS ClientHello with 192.168.1.3:443
S-100  Proxmox awtserver receives packet on wlo1
S-101  iptables PREROUTING DNAT rule fires:
          192.168.1.3:443 rewritten to 10.10.10.111:443
S-102  Kernel routes packet via vmbr1 to Axigen VM
S-103  Axigen Webmail HTTPS service receives connection on 10.10.10.111:443
S-104  Axigen presents SSL certificate (CN=axigen.accesswt.com)
S-105  TLS handshake completes — connection encrypted
S-106  Axigen serves webmail login page
S-107  Recipient enters: admin@school.accesswt.com + password
S-108  Axigen authenticates credentials against local account store
S-109  Authentication success — session created
S-110  Axigen webmail reads INBOX from local file store
S-111  Axigen renders INBOX — shows "Confidential" from info@consultancy.accesswt.com
S-112  Browser displays message to recipient — mail flow complete
```

*[INSERT SCREENSHOT: Axigen webmail inbox of admin@school.accesswt.com showing "Confidential" message with correct From, Subject, and Date]*

---

### 10.2 Production Environment — Step List

> Steps changed from lab are marked `[PROD-CHANGE]`. New production-only steps are marked `[PROD-NEW]`.

#### Sender Side

```
P-01  User opens browser → https://mail.consultancy.accesswt.com           [PROD-CHANGE]
P-02  Browser queries public DNS (8.8.8.8) for A record                    [PROD-NEW]
P-03  Cloudflare/Route53 returns: mail.consultancy.accesswt.com A 1.2.3.4  [PROD-NEW]
P-04  Browser connects to 1.2.3.4:443                                       [PROD-CHANGE]
P-05  ISP/cloud firewall: allow 443 inbound, forward to Zimbra server       [PROD-NEW]
P-06  Zimbra nginx presents Let's Encrypt certificate (trusted by browsers) [PROD-CHANGE]
P-07  Browser trusts cert — no security warning                             [PROD-CHANGE]
P-08  nginx terminates TLS, forwards to Jetty on 127.0.0.1:8080
P-09  Jetty validates session, user composes email, clicks Send
P-10  Browser sends HTTPS POST /service/soap/SendMsgRequest
P-11  Jetty processes request, generates Message-ID, saves to Sent folder
P-12  Jetty passes message to Postfix via sendmail
```

#### Zimbra MTA (Production Additions)

```
P-13  Postfix pickup collects message
P-14  Postfix cleanup normalizes headers
P-15  Postfix milter calls OpenDKIM                                         [PROD-NEW]
P-16  OpenDKIM signs message with private key for consultancy.accesswt.com  [PROD-NEW]
        DKIM-Signature: v=1; a=rsa-sha256; d=consultancy.accesswt.com;
          s=mail; h=from:to:subject:date:message-id; b=<signature>
P-17  DKIM-Signature header added to message                                [PROD-NEW]
P-18  qmgr routes message as remote delivery
P-19  Postfix smtp client starts
```

#### DNS Resolution (Production — Full Recursive)

```
P-20  Postfix queries public resolver for school.accesswt.com MX            [PROD-CHANGE]
P-21  Resolver initiates full recursive resolution                           [PROD-NEW]
P-22  Resolver queries root nameservers (a.root-servers.net etc.)           [PROD-NEW]
P-23  Root servers refer to .com TLD nameservers                            [PROD-NEW]
P-24  .com TLD refers to Cloudflare nameservers for accesswt.com            [PROD-NEW]
P-25  Cloudflare returns: school.accesswt.com MX 10 axigen.accesswt.com     [PROD-CHANGE]
P-26  Resolver caches with production TTL (300-3600 sec)                    [PROD-CHANGE]
P-27  Postfix receives: MX 10 axigen.accesswt.com
P-28  Postfix queries A record: axigen.accesswt.com A 5.6.7.8 (public IP)  [PROD-CHANGE]
P-29  Postfix will deliver to 5.6.7.8:25 over public internet              [PROD-CHANGE]
```

#### Network — Public Internet Transit

```
P-30  Postfix opens TCP to 5.6.7.8:25                                       [PROD-CHANGE]
P-31  Packets traverse ISP network, multiple BGP router hops                [PROD-NEW]
P-32  Axigen cloud/ISP firewall: allow port 25 inbound from anywhere        [PROD-NEW]
P-33  TCP 3-way handshake complete over public internet
```

#### SMTP Handshake + Enhanced Security Checks

```
P-34  220 banner, EHLO, STARTTLS — same as lab
P-35  Axigen presents Let's Encrypt certificate                              [PROD-CHANGE]
P-36  Certificate trusted by Zimbra — no verification errors                [PROD-CHANGE]
P-37  TLS session established
P-38  Zimbra sends: MAIL FROM:<info@consultancy.accesswt.com>
P-39  Axigen performs PTR lookup on connecting IP 1.2.3.4                   [PROD-NEW]
P-40  PTR result: 1.2.3.4 PTR zimbra.accesswt.com                          [PROD-NEW]
P-41  FCrDNS check: zimbra.accesswt.com A 1.2.3.4? YES — match             [PROD-NEW]
P-42  Axigen checks RBL (Spamhaus, SpamCop) for 1.2.3.4                    [PROD-NEW]
P-43  RBL result: CLEAN                                                      [PROD-NEW]
P-44  250 Sender accepted
P-45  Zimbra sends: RCPT TO:<admin@school.accesswt.com>
P-46  Greylisting check (same as lab)
P-47  250 Recipient accepted
P-48  DATA command, message transmitted including DKIM-Signature header     [PROD-CHANGE]
P-49  250 Mail queued — QUIT — TCP closed
```

#### Axigen Internal Processing (Full Validation)

```
P-50  Queue Processing picks up message
P-51  SPF check: 1.2.3.4 matches v=spf1 ip4:1.2.3.4 -all?  PASS          [PROD-CHANGE]
P-52  DKIM: extract DKIM-Signature header                                   [PROD-NEW]
P-53  DNS lookup: mail._domainkey.consultancy.accesswt.com TXT → public key [PROD-NEW]
P-54  Verify signature against message headers+body using public key         [PROD-NEW]
P-55  DKIM result: PASS                                                      [PROD-NEW]
P-56  DMARC: lookup _dmarc.consultancy.accesswt.com TXT                     [PROD-NEW]
P-57  Policy: p=quarantine                                                   [PROD-NEW]
P-58  Alignment: SPF domain == From domain — ALIGNED                        [PROD-NEW]
P-59  Alignment: DKIM domain == From domain — ALIGNED                       [PROD-NEW]
P-60  DMARC result: PASS                                                     [PROD-NEW]
P-61  Authentication-Results header added to message:                        [PROD-NEW]
        spf=pass dkim=pass dmarc=pass
P-62  Antispam scoring (boosted by auth-pass results)
P-63  Antivirus scan: CLEAN
P-64  Write to /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX/
P-65  Index updated, delivery logged
```

#### Receiver Side (Production)

```
P-66  Recipient opens browser → https://mail.school.accesswt.com            [PROD-CHANGE]
P-67  Public DNS: mail.school.accesswt.com A 5.6.7.8                        [PROD-CHANGE]
P-68  Cloud firewall: allow 443 inbound                                      [PROD-NEW]
P-69  Axigen presents Let's Encrypt cert — browser trusts it                [PROD-CHANGE]
P-70  Recipient logs in, Axigen serves INBOX
P-71  Message shown with Authentication-Results: spf=pass dkim=pass dmarc=pass [PROD-NEW]
P-72  Recipient reads mail — flow complete
```

---

### 10.3 Detailed Explanation — Lab Steps

#### S-01 to S-06 — Browser to Proxmox NAT

The MacBook sends HTTPS to `192.168.1.3:8443`. This IP belongs to `wlo1` on `awtserver` — the Proxmox host acting purely as a NAT gateway. The Linux kernel processes `iptables PREROUTING` **before routing**, so the DNAT rule fires first:

```bash
# Rule effect:
#   BEFORE: destination = 192.168.1.3:8443
#   AFTER:  destination = 10.10.10.110:8443
# Then FORWARD chain routes via vmbr1 to Zimbra VM

# Verify on awtserver
iptables -t nat -L PREROUTING -n --line-numbers | grep 8443
cat /proc/sys/net/ipv4/ip_forward   # must output: 1
```

*[INSERT SCREENSHOT: iptables -t nat -L PREROUTING -n showing DNAT rule for port 8443 to 10.10.10.110]*

#### S-07 to S-18 — Zimbra nginx and Jetty

nginx listens on port 8443, terminates TLS, and forwards plain HTTP to Jetty on `127.0.0.1:8080`. Jetty handles the SOAP API, generates the Message-ID, saves to Sent folder via LMTP port 7025, then hands the outgoing message to Postfix via the local sendmail binary.

```bash
# Watch the SOAP call arrive in real time
sudo tail -f /opt/zimbra/log/mailbox.log | grep "SendMsgRequest\|info@consultancy"
```

*[INSERT SCREENSHOT: mailbox.log showing SendMsgRequest with account=info@consultancy.accesswt.com]*

#### S-19 to S-31 — Postfix Queue Processing

Postfix uses a multi-daemon architecture — each daemon does exactly one job:

| Daemon | Responsibility |
|---|---|
| `pickup` | Reads messages from local sendmail pipe |
| `cleanup` | Adds missing headers, normalizes addresses |
| `qmgr` | Schedules and manages delivery queue |
| `smtp` | Makes outbound SMTP connections |
| `smtpd` | Accepts inbound SMTP connections |

```bash
# Verify routing decision parameters
su - zimbra
postconf mydestination
postconf relay_domains
postconf mynetworks
watch -n1 "postqueue -p | head -20"
```

*[INSERT SCREENSHOT: postconf mydestination output confirming school.accesswt.com is NOT listed as local]*

#### S-32 to S-46 — DNS Resolution in Lab

```
Postfix → ns3 (10.10.10.108) [cache miss]
       → ns1 (10.10.10.106) authoritative
       → ns1 reads db.accesswt.com
       → Returns: MX 10 axigen.accesswt.com
       → ns3 caches with TTL=604800
       → Returns to Postfix

Postfix → ns3: axigen.accesswt.com A?
       → ns1: 10.10.10.111
       → Postfix connects to 10.10.10.111:25
```

```bash
# Trace the full resolution
dig @10.10.10.108 school.accesswt.com MX +trace
dig @10.10.10.106 school.accesswt.com MX +norecurse
```

*[INSERT SCREENSHOT: dig @10.10.10.108 school.accesswt.com MX showing ANSWER SECTION with axigen.accesswt.com MX 10]*

#### S-47 to S-52 — TCP Connection (Internal)

Both servers are on `10.10.10.0/24` via `vmbr1` — no NAT, no internet traversal.

```bash
# Monitor live connections
ss -tn state established "( dport = :25 )"
```

#### S-53 to S-67 — SMTP Handshake and STARTTLS

STARTTLS upgrades a plain TCP connection to TLS without changing ports. The client MUST send EHLO again after TLS upgrade (RFC 3207 requirement).

```bash
# Manually test the full SMTP+STARTTLS handshake
openssl s_client -connect 10.10.10.111:25 -starttls smtp 2>/dev/null \
  | grep -E "Protocol|Cipher|subject|issuer|notBefore|notAfter"
```

*[INSERT SCREENSHOT: openssl s_client -starttls smtp output showing Protocol TLS 1.2/1.3, Cipher, and CN=axigen.accesswt.com]*

#### S-68 to S-85 — Envelope, Data Transfer, Greylisting

The SMTP **envelope** (MAIL FROM / RCPT TO) is separate from **headers** (From: / To:). Greylisting temporarily rejects the first delivery attempt from any new sender+recipient+IP combination — legitimate MTAs retry, spam software does not.

```bash
# On Axigen — see greylisting
sudo grep -i "greylist\|451" /var/log/axigen/smtp-in.log | tail -10

# On Zimbra — watch delivery
sudo grep "status=" /var/log/zimbra.log | tail -5
# Expected: status=sent (250 Mail queued for delivery)
```

*[INSERT SCREENSHOT: /var/log/axigen/smtp-in.log showing 451 greylisting then 250 accepted on retry]*
*[INSERT SCREENSHOT: /var/log/zimbra.log showing postfix status=sent for admin@school.accesswt.com]*

#### S-86 to S-96 — Axigen Internal Processing

```bash
# Watch processing in real time
sudo tail -f /var/log/axigen/smtp-in.log

# Verify .eml file written to disk
sudo find /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX \
  -name "*.eml" | sort | tail -3
```

*[INSERT SCREENSHOT: find command showing .eml file in /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX/]*

#### S-97 to S-112 — Recipient Reads Mail

Same DNAT pattern: `192.168.1.3:443` → `10.10.10.111:443`. Axigen Webmail reads directly from the local file store.

```bash
curl -sk https://10.10.10.111:443/webmail | grep -i "axigen\|login"
```

---

### 10.4 Detailed Explanation — Production Steps

#### P-01 to P-06 — Public DNS and Certificate Trust

In production three things are required that do not exist in the lab:
- A record in **public DNS** (Cloudflare/Route53)
- A **publicly routable IP address**
- A **TLS certificate from a trusted CA** (Let's Encrypt or commercial)

```bash
# Set up Let's Encrypt on Axigen
sudo apt install -y certbot
sudo certbot certonly --standalone -d axigen.accesswt.com

# Auto-renewal hook
sudo tee /etc/letsencrypt/renewal-hooks/deploy/axigen-reload.sh << 'HOOK'
#!/bin/bash
bash -c 'cat /etc/letsencrypt/live/axigen.accesswt.com/fullchain.pem \
  /etc/letsencrypt/live/axigen.accesswt.com/privkey.pem \
  > /var/opt/axigen/axigenmail.pem'
chown axigen:axigen /var/opt/axigen/axigenmail.pem
systemctl reload axigen
HOOK
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/axigen-reload.sh
sudo certbot renew --dry-run
```

#### P-15 to P-17 — DKIM Signing

Every outgoing message in production must be DKIM signed. Zimbra includes OpenDKIM which signs before Postfix sends.

```bash
# Generate DKIM keys
su - zimbra
/opt/zimbra/libexec/zmdkimkeyutil -a -d consultancy.accesswt.com
# Output: TXT record must be added to PUBLIC DNS:
# mail._domainkey.consultancy.accesswt.com TXT "v=DKIM1; k=rsa; p=..."

zmcontrol status | grep opendkim
# Expected: opendkim   Running
```

#### P-21 to P-26 — Full Recursive DNS (Production)

In production the resolver does full recursion through root → TLD → authoritative. TTL should be 300–3600 seconds (not 604800 as in lab) to allow faster failover.

```bash
# Verify full resolution path
dig school.accesswt.com MX +trace

# Check production TTL
dig school.accesswt.com MX | grep "IN.*MX"
```

#### P-39 to P-43 — PTR, FCrDNS, and RBL (Production Critical)

These three checks do not exist in the lab but are critical in production. Gmail and Outlook reject mail without PTR.

```bash
# Verify PTR is set (ask ISP to configure)
dig -x 1.2.3.4 +short
# Expected: zimbra.accesswt.com.

# Verify FCrDNS (both directions must agree)
dig -x 1.2.3.4 +short        # should: zimbra.accesswt.com.
dig zimbra.accesswt.com A +short  # should: 1.2.3.4

# Check blacklist status
host 4.3.2.1.zen.spamhaus.org
# NXDOMAIN = not blacklisted (good)
```

#### P-50 to P-61 — Full SPF + DKIM + DMARC

```bash
# Verify all three records in PUBLIC DNS
dig +short TXT consultancy.accesswt.com | grep spf
dig +short TXT mail._domainkey.consultancy.accesswt.com
dig +short TXT _dmarc.consultancy.accesswt.com
```

After all three pass, Axigen adds to the message:
```
Authentication-Results: axigen.accesswt.com;
  spf=pass smtp.mailfrom=consultancy.accesswt.com;
  dkim=pass header.d=consultancy.accesswt.com;
  dmarc=pass header.from=consultancy.accesswt.com
```

---

### 10.5 DNS Record Types — Complete Reference

| Record | Lab Example | Purpose |
|---|---|---|
| MX | school.accesswt.com MX 10 axigen.accesswt.com | Which server receives mail |
| A | axigen.accesswt.com A 10.10.10.111 | IP of MX host |
| PTR | 111.10.10.10.in-addr.arpa PTR axigen.accesswt.com | Reverse DNS — required in production |
| TXT/SPF | school.accesswt.com TXT "v=spf1 ip4:10.10.10.111 ~all" | Sender authorization |
| TXT/DKIM | mail._domainkey.school.accesswt.com TXT "v=DKIM1; k=rsa; p=..." | Signing key |
| TXT/DMARC | _dmarc.school.accesswt.com TXT "v=DMARC1; p=quarantine" | Policy enforcement |

```bash
# Verify all mail-related records for a domain
DOMAIN="school.accesswt.com"
NS="10.10.10.106"

for TYPE in MX TXT; do
  echo "=== $DOMAIN $TYPE ==="
  dig @$NS $DOMAIN $TYPE +short
done

echo "=== PTR ==="
dig @$NS -x 10.10.10.111 +short

echo "=== DKIM ==="
dig @$NS mail._domainkey.$DOMAIN TXT +short

echo "=== DMARC ==="
dig @$NS _dmarc.$DOMAIN TXT +short
```

---

### 10.6 Port Reference Table

| Port | Protocol | Service | Lab | Production |
|---|---|---|---|---|
| 25 | TCP | SMTP server-to-server | Internal only | Open to internet |
| 80 | TCP | HTTP redirect | Internal | Open |
| 143 | TCP | IMAP plain | Avoid | Disabled |
| 443 | TCP | HTTPS Webmail (Axigen) | DNAT from 192.168.1.3 | Public |
| 465 | TCP | SMTPS implicit TLS | Internal | Open |
| 587 | TCP | SMTP Submission STARTTLS | Internal | Open |
| 993 | TCP | IMAPS | DNAT from 192.168.1.3 | Public |
| 995 | TCP | POP3S | Optional | Optional |
| 7071 | TCP | Zimbra Admin Console | DNAT from 192.168.1.3 | VPN only |
| 8443 | TCP | Zimbra Webmail HTTPS | DNAT from 192.168.1.3 | Public |
| 9443 | TCP | Axigen WebAdmin HTTPS | DNAT from 192.168.1.3 | VPN only |

---

### 10.7 Mail Queuing — Complete Detail

```
Message submitted via sendmail pipe
  MAILDROP queue (pickup reads)
  INCOMING queue (cleanup processes)
  ACTIVE queue (qmgr schedules, max 20k messages)
    SUCCESS  → removed, logged
    4xx fail → DEFERRED queue
               Retry: 5min, 10min, 20min ... up to 5 days
               After 5 days → permanent bounce to sender
    5xx fail → immediate bounce, removed
```

```bash
su - zimbra
postqueue -p          # view queue
postqueue -f          # force flush all deferred
postcat -q QUEUEID    # read specific message + reason
postsuper -d QUEUEID  # delete specific message
postsuper -d ALL deferred   # delete all deferred (use carefully)
```

---

### 10.8 TLS Handshake — Step by Step

```
1.  TCP connection established (plain)
2.  220 SMTP banner
3.  EHLO → 250-STARTTLS in capabilities
4.  STARTTLS → 220 Go ahead
    --- TLS handshake ---
5.  ClientHello: TLS versions + ciphers offered
6.  ServerHello: version + cipher selected
7.  Server sends Certificate (CN=axigen.accesswt.com)
8.  Key exchange (ECDHE)
9.  Both send ChangeCipherSpec + Finished
    --- All data now encrypted ---
10. Client sends EHLO again (RFC 3207 requirement)
11. Server sends 250 capabilities over TLS
```

```bash
# Full TLS inspection
openssl s_client -connect 10.10.10.111:25 -starttls smtp 2>/dev/null \
  | grep -E "Protocol|Cipher|subject|notBefore|notAfter"

# Verify weak TLS rejected
openssl s_client -connect 10.10.10.111:993 -tls1_1 2>&1 | grep -i error
# Expected: handshake failure

# Verify TLS 1.2 accepted
openssl s_client -connect 10.10.10.111:993 -tls1_2 2>&1 | grep "Cipher is"
```

---

### 10.9 Greylisting Flow

```
First attempt (new tuple: IP + MAIL FROM + RCPT TO):
  Axigen → 451 Temporary rejected by greylisting
  Postfix → receives 4xx → DEFERRED queue → retry in 5 min

Second attempt (same tuple, 5+ min later):
  Axigen → tuple whitelisted → 250 Recipient accepted
  Delivery proceeds

Why it works:
  Legitimate MTA (Postfix, Exchange) → always retries 4xx
  Spam software → usually does not retry
  Result: spam blocked; legitimate mail delayed ~5 min on first contact only
```

---

### 10.10 SPF / DKIM / DMARC Validation Chain

```
SPF:
  Connecting IP: 10.10.10.110
  MAIL FROM domain: consultancy.accesswt.com
  DNS TXT lookup: v=spf1 mx ip4:10.10.10.110 ~all
  Is 10.10.10.110 in record? YES → PASS

DKIM (production only):
  Extract DKIM-Signature header from message
  DNS lookup: mail._domainkey.consultancy.accesswt.com → public key
  Verify b= signature against headers+body using public key
  Match → PASS

DMARC:
  From: header domain = consultancy.accesswt.com
  DNS lookup: _dmarc.consultancy.accesswt.com
  Policy: p=quarantine
  SPF domain aligned with From domain? YES
  DKIM domain aligned with From domain? YES
  At least one aligned and passes → DMARC PASS
  Policy action: deliver normally
```

---

### 10.11 Lab vs Production Comparison

| Item | Lab | Production |
|---|---|---|
| User URL | `https://192.168.1.3:8443` | `https://mail.consultancy.accesswt.com` |
| DNS | Internal BIND only | Public (Cloudflare/Route53) |
| Certificate | Self-signed (browser warns) | Let's Encrypt (trusted) |
| PTR record | Not needed | Required by Gmail/Outlook |
| DKIM | Optional | Required for external delivery |
| SPF/DMARC | Internal DNS | Public DNS |
| RBL checks | Not applicable | Active (Spamhaus, SpamCop) |
| FCrDNS | Not needed | Required |
| Firewall | ufw + iptables DNAT | Hardware/cloud firewall |
| Port 25 access | Internal only | Open to internet |
| Backup MX | Not configured | Recommended |
| IP warming | Not needed | Required for new IPs |
| TTL | 604800 (1 week) | 300–3600 (5 min to 1 hour) |

## 11. Test Cases — All Tried Scenarios

### 11.1 SMTP Connectivity Tests

**Test 1 — Manual SMTP via telnet (initial, failed due to idle timeout)**
```bash
telnet 10.10.10.111 25
220 axigen.accesswt.com Axigen ESMTP ready
421 axigen.accesswt.com timeout expired waiting for data
# RESULT: FAILED — typed too slowly, Axigen idle timeout triggered
```

**Test 2 — Manual SMTP via telnet (correct, typed fast)**
```bash
telnet 10.10.10.111 25
220 axigen.accesswt.com Axigen ESMTP ready
EHLO test
250-axigen.accesswt.com Axigen ESMTP hello
250-PIPELINING
250-AUTH PLAIN LOGIN CRAM-MD5 DIGEST-MD5 GSSAPI
250-STARTTLS
250 OK
MAIL FROM:<test@accesswt.com>
250 Sender accepted
RCPT TO:<admin@school.accesswt.com>
250 Recipient accepted
DATA
451 Temporary rejected by greylisting   ← greylisting active
# RESULT: Expected — greylisting working
```

**Test 3 — Manual SMTP retry after greylisting**
```bash
DATA
354 Ready to receive data; remember <CRLF>.<CRLF>
Subject: Test

This test message.
.
250 Mail queued for delivery
# RESULT: SUCCESS — mail accepted on retry
```

**Test 4 — Mail body missing (RFC822 formatting error)**
```
Subject: Test
This test message.
.
# RESULT: Mail arrived with empty body — missing blank line between header and body
# FIX: Must have empty line between Subject: header and body text
```

**Test 5 — Correct RFC822 format with body**
```
Subject: Test with body

This is the actual message body.
.
# RESULT: SUCCESS — body displayed correctly in webmail
```

**Test 6 — nc heredoc method (automated)**
```bash
{
echo -e "EHLO test\r"
sleep 1
echo -e "MAIL FROM:<test@accesswt.com>\r"
sleep 1
echo -e "RCPT TO:<admin@school.accesswt.com>\r"
sleep 1
echo -e "DATA\r"
sleep 1
echo -e "Subject: Test\r\n\r\nBody text\r\n.\r"
} | timeout 15 nc 10.10.10.111 25
# RESULT: SUCCESS
```

*[INSERT SCREENSHOT: Axigen webmail showing received test email with subject and body visible]*

### 11.2 Zimbra CLI Send Tests

**Test 7 — Send via Zimbra sendmail binary**
```bash
echo -e "Subject: Test Mail\nFrom: info@consultancy.accesswt.com\nTo: admin@microfinance.accesswt.com\n\nThis is the message transfer test." \
  | /opt/zimbra/common/sbin/sendmail -t

# Verify delivery
zmmailbox -z -m admin@microfinance.accesswt.com search "in:inbox subject:\"Test Mail\""
# Expected: num: 1, more: false
# RESULT: SUCCESS
```

**Test 8 — Single message delete on Zimbra**
```bash
MSG_ID=$(zmmailbox -z -m admin@microfinance.accesswt.com s -v 'in:inbox subject:"Test Mail"' \
  | grep '"messageIds":' | tr -dc '0-9')
zmmailbox -z -m admin@microfinance.accesswt.com deleteMessage "$MSG_ID"

# Verify deleted
zmmailbox -z -m admin@microfinance.accesswt.com search "in:inbox subject:\"Test Mail\""
# Expected: num: 0, more: false
# RESULT: SUCCESS
```

*[INSERT SCREENSHOT: zmmailbox search before delete showing num:1, and after showing num:0]*

### 11.3 Python SMTP Bulk Send Tests

**Test 9 — Python bulk send to all users across both servers**
```python
# Script: /tmp/send_confidential_bulk.py
# Sender: info@consultancy.accesswt.com
# Server: 10.10.10.110 (Zimbra MTA)
# Recipients: All Zimbra + Axigen users (30+ recipients)
# Subject: Confidential
# RESULT: All delivered — ✓ shown for each recipient
```

*[INSERT SCREENSHOT: Python script output showing ✓ for each recipient email address]*

### 11.4 Cross-Server Mail Flow Tests

**Test 10 — Zimbra to Axigen delivery**
```
From: info@consultancy.accesswt.com (Zimbra)
To:   admin@school.accesswt.com (Axigen)
Path: Zimbra MTA → DNS MX lookup → Axigen SMTP 10.10.10.111:25
RESULT: SUCCESS — mail appeared in Axigen webmail inbox
```

**Test 11 — Axigen webmail receipt verification**
```
Login: admin@school.accesswt.com at https://192.168.1.3:443/webmail
Check: INBOX
Found: "Confidential" from info@consultancy.accesswt.com
RESULT: SUCCESS
```

*[INSERT SCREENSHOT: Axigen webmail inbox of admin@school.accesswt.com showing "Confidential" message from info@consultancy.accesswt.com]*

### 11.5 swaks Test

**Test 12 — swaks single-user push with verbose output**
```bash
swaks --to admin@school.accesswt.com \
      --from test@accesswt.com \
      --server 10.10.10.111 \
      --port 25 \
      --header "Subject: Single Push Test" \
      --body "This is a single targeted test message." \
      --verbose
# RESULT: Full SMTP conversation displayed, 250 OK received
```

### 11.6 mailutils Test (Failed — No local MTA)

**Test 13 — mail command (failed, no sendmail binary)**
```bash
echo "Body" | mail -s "Test" -r test@accesswt.com admin@school.accesswt.com
# ERROR: mail: Cannot open mailer: No such file or directory
# REASON: mailutils requires local sendmail binary — not installed on Axigen
# FIX: Use -S smtp-host=10.10.10.111 flag, or use swaks instead
```

### 11.7 Mail Transfer test ( Zimbra ~ Axigen)

***Test 11.7.1 - message transfer Zimbra to Axigen***
***Message send from info@consultancy.accesswt.com (Zimbra) to principal@school.accesswt.com (principal)***
<img width="600" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-trf-zimbra2axigen.png">

***verifying using CLI***
```bash
~/libexec/zmmsgtrace -s info@consultancy.accesswt.com -r principal@school.accesswt.com /var/log/zimbra.log

# OR

/opt/zimbra/libexec/zmmsgtrace -debug   -s info@consultancy.accesswt.com   -r principal@school.accesswt.com   /var/log/zimbra.log
```

<img width="800" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-trf-log3.png">

***Checking Logs in Zimbra Server***

<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-trf-log1.png">

***Line 1 — Postfix smtpd receives the message from Zimbra's own webmail***

What it means:

- smtpd = Postfix SMTP server daemon receiving an inbound connection
- NOQUEUE = not yet assigned a queue ID — still in the filter decision phase
- filter: RCPT = the smtpd policy filter fired on this RCPT TO
- triggers FILTER smtp-amavis:[127.0.0.1]:10026 = Postfix decided to route this through Amavis on port 10026 (the outbound Amavis port — 10026 is for mail originating locally, 10024 is for inbound from internet)
- The sender is info@consultancy.accesswt.com, originating from 10.10.10.110 (Zimbra itself — this is a locally-submitted message from a Zimbra user)

***Line 2 — Queue ID assigned, message enters active queue***

What it means:

- qmgr = queue manager daemon
- 1D568202F8CE = Queue ID 1 — the first queue ID this message gets. Think of it as a tracking number.
- size=1325 = message is 1,325 bytes at this point (headers + body)
- nrcpt=1 = 1 recipient
- queue active = message is being processed right now, not deferred

> Important: Queue IDs change as the message passes through different stages. This same message will get 3 different queue IDs before it leaves. This is normal — track the Message-ID instead for end-to-end tracing.

***Lines 3–4 — Amavis receives the message on port 10026 (first pass)***

What it means:

- Amavis worker [5164] received this message on port 10026
- ORIGINATING/MYNETS = Amavis recognized this as outbound mail from a trusted local network (MYNETS = 10.10.10.0/24 in config)
- 2WJBQ0bh4YSV = Amavis's own internal mail ID (different from Postfix Queue ID)
- This is the first Amavis pass — checks if it should be signed, scanned, relayed

***Line 5 — OpenDKIM: no signing key found (important — note this)***

What it means:

- OpenDKIM looked for a DKIM private key for consultancy.accesswt.com
- Found nothing — no DKIM key has been generated for this domain yet
- Result: message goes out without a DKIM signature — this will hurt deliverability in production
- 6B091202F8E4 = Queue ID 2 — a new queue ID assigned after Amavis processed it

> Fix: su - zimbra && /opt/zimbra/libexec/zmdkimkeyutil -a -d consultancy.accesswt.com then add the resulting TXT record to DNS.
> 
> Updating or enabling DKIM (DomainKeys Identified Mail) allows mail server to digitally sign outgoing emails so that receiving mail servers can verify they are authentic and haven't been altered in transit.

***Line 6 — Amavis forwards message back to Postfix (first pass complete)***

What it means:

- Amavis finished its first pass and forwarded the message back to Postfix on port 10030
- Postfix accepted it and assigned Queue ID 6B091202F8E4
- BODY=7BIT = message body is plain 7-bit ASCII (no attachments, no special encoding)

***Line 7 — Amavis first pass result: CLEAN, Relayed***

What it means:

- Passed CLEAN = passed antivirus and antispam checks
- {RelayedOutbound} = Amavis action: relay this outbound
- Hits: - = no spam hits (dash means no rules triggered at all — very clean)
- 1276 ms = this Amavis check took 1.276 seconds
- queued_as: 6B091202F8E4 = new queue ID after Amavis
- The Message-ID is the stable identifier we can use to trace across all log lines: 506982069.7.1783322704734.JavaMail.zimbra@consultancy.accesswt.com

***Lines 8–9 — SECOND Amavis pass (port 10032)***

What it means:

- This is a second Amavis pass on port 10032 — ORIGINATING_POST means this is Zimbra's post-queue content filter (a second check after the message was re-injected)
- Zimbra runs mail through Amavis twice for outbound: once at submission (10026), once after queue injection (10032). This is by design for thorough scanning.
- Different Amavis worker (5165) handles this pass

***Line 10 — Queue ID 2 enters active queue***
- Queue ID 6B091202F8E4 is now 1826 bytes — grew from 1325 because Amavis added headers (X-Spam-Status, Received, Authentication-Results, etc.)

***Lines 11–12 — Second Amavis pass result***

What it means:

- Hits: 6.276 = SpamAssassin score of 6.276 on the second pass — this is above the default 5.0 threshold for tagging. The message is being treated as borderline spam on the second scan.
- queued_as: E1DA7202F8CE = Queue ID 3 — the final queue ID for delivery
- size: 1791 = now 1,791 bytes after second Amavis added more headers

***Line 13 — Final queue ID enters active queue***

What it means:

Queue ID E1DA7202F8CE is now 2211 bytes — grew again from more headers
Status: queue active — Postfix is about to attempt delivery to 10.10.10.111:25


***The problem: the log cuts off here***
The last line shows queue active — Postfix is about to deliver, but the delivery result line is missing. we need to see one of these:

***Run this to find the final delivery line

```bash
grep "E1DA7202F8CE" /var/log/zimbra.log
```
<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-trf-log2.png">

| Field | What to look for | Meaning |
|------|------------------|--------|
| `status=sent` | In `postfix/smtp` line | Mail left Zimbra successfully |
| `status=deferred` | In `postfix/smtp` line | Temporary failure — will retry |
| `status=bounced` | In `postfix/smtp` line | Permanent failure — bounced to sender |
| `relay=axigen.accesswt.com[10.10.10.111]:25` | In `postfix/smtp` line | Delivered to Axigen specifically |
| `dsn=2.0.0` | In `postfix/smtp` line | Delivery Status Notification: success |
| `250 response` | In `postfix/smtp` line | Remote server accepted the message |


```postfix/qmgr[5615]: E1DA7202F8CE: removed```

Queue ID removed — Postfix is done with this message, nothing deferred.
And postqueue -p shows Mail queue is empty — nothing stuck.

```bash
postqueue -p
```

***11.7.2 Update DKIM***

***Step-1 Generate key***
```bash
su - zimbra && /opt/zimbra/libexec/zmdkimkeyutil -a -d consultancy.accesswt.com
```
<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/update-dkim1.png">

***Step-2 Add to zone file on ns1***
```bash
sudo rndc freeze accesswt.com
sudo vim /etc/bind/zones/db.accesswt.com
```

***Step-3 Add this block (the long key must stay on separate quoted lines exactly as shown):***
mail._domainkey.consultancy.accesswt.com.  IN TXT "v=DKIM1; k=rsa; p=<paste key here>"

```bash
; DKIM for consultancy.accesswt.com (Zimbra)
D4E580C2-7931-11F1-A322-0673842193A9._domainkey.consultancy.accesswt.com.  IN  TXT  ( "v=DKIM1; k=rsa; "
  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAubWRBZue/zwkjBjLesSFP/+f7QzfYGMwom1U3dCP43d9hkE8QF42p+1hk5G4FtN29bQf0pFqFbIXGg6zVgo+0jqiIDauE+iYkLSTzlEYsFnoEybHHlhzaA6Wy8rWVjvX5W07bCwfnEbMNIFrPEnQWxLG+xjS+ZoWHjNkO7MLAd3fMmZJ6o33W1B2E9U8CfA7DayxM86i0zzQ3d"
  "GRQdGtmXxFpIc3tbUaobLXNtKLs8BR6VW1XMios2XcVsF1Oc4Nd+B2NPZHiwoe/4XUDHii7Ij5rog2mSoeo6MrjSnD/z0bz7WirAum3bpYWsPxgTvVzeQuxGNoTF9auM7lvKQeoQIDAQAB" )
```
Also increment the serial number in the SOA:

***Step-4 Validate and reload***
```bash
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
# Must show: OK

sudo rndc reload accesswt.com
sudo rndc thaw accesswt.com
```

***Step-5 Verify the record resolves***
```bash
dig @10.10.10.106 D4E580C2-7931-11F1-A322-0673842193A9._domainkey.consultancy.accesswt.com TXT +short
```
<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/update-dkim2.png">

***Step-6 Send fresh mail and check the log:***
```bash
grep info@consultancy.accesswt.com /var/log/zimbra.log
```
<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/update-dkim3.png">

***11.7.3 Message Receive in Axigen***

```bash
sudo grep -i "info@consultancy" /var/opt/axigen/log/everything.txt | tail -30
```
<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-rcvd-in-axigen-log1.png">

Axigen `everything.txt` — How to Read It

The log format is:

```text
TIMESTAMP                     LEVEL  PROCESS  SERVICE:WORKER_ID:    MESSAGE

2026-07-06 13:10:08 +0545     08     axigen  PROCESSING:002C2605:   ...

│                             │      │        │
│                             │      │        └── Internal mail object (worker) ID
│                             │      └────────── Axigen subsystem that generated the log
│                             └───────────────── Log level (`08` = Informational)
└────────────────────────────── Nepal time (UTC+5:45)
```

| Subsystem | What it logs |
|------------|--------------|
| **SMTP-IN** | Inbound SMTP connections from other mail servers. |
| **SMTP-OUT** | Outbound SMTP delivery to remote mail servers. |
| **PROCESSING** | Queue processing, message filtering, and mailbox delivery. |
| **WEBMAIL** | HTTP/HTTPS requests generated from Axigen WebMail sessions. |
| **MAILBOXAPI** | Internal mailbox API operations (folder listing, conversation retrieval, message access, etc.). |
| **JOBLOG** | Background jobs such as search indexing, sorting, maintenance, and scheduled tasks. |

***Line 1 SPF check (first thing Axigen does)***
SPF (Sender Policy Framework) is a DNS-based authentication method that tells receiving mail servers which IP addresses are allowed to send email for a domain.

What it means: 

Axigen ran an SPF lookup on consultancy.accesswt.com, found v=spf1 mx ip4:10.10.10.110 ~all, confirmed the connecting IP 10.10.10.110 is listed — SPF PASS. This is your SPF record working exactly as designed.

This tells us:
    MAIL FROM = info@consultancy.accesswt.com
        This is the envelope sender used during the SMTP conversation.
    EHLO domain = zimbra.accesswt.com
        This is the hostname your Zimbra server introduced itself as.
    Connecting IP = 10.10.10.110
        This is the IP address that connected to the Axigen server.
    Result = Pass
The connecting IP is authorized to send mail for consultancy.accesswt.com

***Line 2-4 Blacklist and whitelist filter checks***

What it means: 

Axigen ran its built-in wmFilter script against the sender:
Not on blacklist → continue processing (good)
Not on whitelist → apply normal filtering rules (neutral — whitelist would bypass spam checks)
Keep requested = the filter script decided to keep/deliver the message, not reject it

***Line 5 New Mail Received***

SMTP-IN:0000001B: New mail
  <506982069.7.1783322704734.JavaMail.zimbra@consultancy.accesswt.com>
  received from zimbra.accesswt.com (10.10.10.110)
  with envelope from <info@consultancy.accesswt.com>
  recipients=1 (principal@school.accesswt.com)
  size=2210
  enqueued with id 2C2605

This is Axigen's "I received the mail" confirmation. Notice the Message-ID matches exactly what Zimbra sent — same 506982069.7.1783322704734 — proving this is the same message tracked across both logs.

The state machine progression:

```text
RECEIVED → PROCESSING → PROCESSING → PROCESSED-LOCAL → INBOX(id=6) → SENT
```
`state to SENT` here means Axigen finished its job — the message was handed off to the mailbox store. "SENT" in Axigen's processing context means "dispatched to destination" not "sent outbound".

<img width="1200" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/msg-rcvd-in-axigen-log1.png">

```text
13:07:16  Account 'principal@school.accesswt.com' has logged in
13:07:17  GET /api/v1/conversations → code=200
13:10:08  GET /api/v1/conversations → code=200 time=7   ← inbox refresh when mail arrived
13:10:11  GET /api/v1/conversations → code=200 time=26  ← webmail auto-refreshed, saw new mail
13:10:11  GET /api/v1/conversations/Nw → time=2         ← opened the conversation
13:10:11  GET /api/v1/mails/NzFfMTY3NzcyNDdfNg/parts   ← fetched message parts
13:10:11  GET /api/v1/mails/.../parts/details → code=200 ← rendered message body
13:10:16  PATCH /api/v1/conversations/Nw/ → code=200    ← user marked it as read
```

## 12. Bulk Mail Operations (Push & Delete)

### 12.1 Zimbra — Bulk Delete Script (Confirmed Working)

```bash
#!/bin/bash
# /tmp/bulk_delete_zimbra.sh
SEARCH_QUERY="in:inbox subject:Confidential"

for ACCOUNT in $(zmprov -l gaa); do
  echo "Checking ${ACCOUNT}..."
  MSG_IDS=$(zmmailbox -z -m "$ACCOUNT" search "$SEARCH_QUERY" \
    | awk '/^[0-9]+\./ {print $2}')

  if [ ! -z "$MSG_IDS" ]; then
    for ID in $MSG_IDS; do
      echo "  Found conversation ID ${ID} — deleting..."
      zmmailbox -z -m "$ACCOUNT" deleteConversation "$ID" >/dev/null 2>&1
    done
  fi
done
echo "Bulk purge complete."
```

**Result:** Ran across 28 accounts, found and deleted from all recipients.
```
zmmailbox -z -m info@microfinance.accesswt.com search "in:inbox subject:Confidential"
num: 0, more: false   ← CONFIRMED DELETED
```

### 12.2 Zimbra — Empty All Trash

```bash
for ACCOUNT in $(zmprov -l gaa); do
  echo "Emptying Trash for ${ACCOUNT}..."
  zmmailbox -z -m "$ACCOUNT" emptyFolder /Trash >/dev/null 2>&1
done
echo "All Trash folders successfully purged!"

# Verify
zmmailbox -z -m admin@consultancy.accesswt.com s -l 100 "in:trash"
# Expected: num: 0, more: false
```

*[INSERT SCREENSHOT: bulk_delete_zimbra.sh output showing "Found conversation ID" and deletion across all accounts]*

*[INSERT SCREENSHOT: emptyFolder script output showing all 28 accounts trash emptied]*

### 12.3 Axigen — Check Any Mail in Specific User

```bash
# /tmp/axigen_check_any_mail.py
# Lists all folders and message counts for a specific account
python3 /tmp/axigen_check_any_mail.py
# Expected: Shows each folder (INBOX, Sent, Trash, Drafts) with message count
```

### 12.4 Axigen — Check Specific Mail in Specific User

```bash
# /tmp/axigen_check_specific_mail.py
# Searches INBOX and Trash for specific subject
python3 /tmp/axigen_check_specific_mail.py
# Expected: Shows message details (Subject, From, Date) if found
```

### 12.5 Axigen — Bulk Move to Trash (IMAP)

```bash
# /tmp/axigen_move_to_trash.py
# Marks matching messages \Deleted + expunges from INBOX across all accounts
# Shows BEFORE/AFTER count per account + verifies in Trash
python3 /tmp/axigen_move_to_trash.py
```

### 12.6 Axigen — Empty Trash (IMAP)

```bash
# /tmp/axigen_empty_trash.py
# Shows contents before deletion, empties permanently, verifies 0 items after
python3 /tmp/axigen_empty_trash.py
```

### 12.7 Axigen — Permanent Delete (No Recovery)

```bash
# /tmp/axigen_permanent_delete.py
# DRY_RUN = True  → preview only, nothing deleted
# DRY_RUN = False → marks \Deleted + immediate expunge, bypasses Trash
python3 /tmp/axigen_permanent_delete.py
```

---

## 13. Monitoring Commands

### 13.1 Zimbra Monitoring

```bash
su - zimbra

# ── Service Status ──────────────────────────────────────────────
zmcontrol status
# Shows all services: Running / Stopped

# ── Mail Queue ──────────────────────────────────────────────────
postqueue -p
# Empty = all mail delivered
# Shows stuck/deferred messages if any

postqueue -f
# Force retry of deferred messages

# ── Log Monitoring ──────────────────────────────────────────────
# Main combined log
sudo tail -f /var/log/zimbra.log

# MTA/SMTP log
sudo tail -f /opt/zimbra/log/mailbox.log

# Real-time SMTP activity
sudo tail -f /var/log/zimbra.log | grep -E "postfix|amavis|smtp"

# ── Mailbox Search (per user) ────────────────────────────────────
zmmailbox -z -m user@domain.com search "in:inbox"
zmmailbox -z -m user@domain.com search "in:inbox subject:\"keyword\""
zmmailbox -z -m user@domain.com search "in:trash"
zmmailbox -z -m user@domain.com search "in:sent"

# ── Account Management ───────────────────────────────────────────
zmprov -l gaa              # list all accounts
zmprov gaa domain.com      # list accounts per domain
zmprov gad                 # list all domains

# ── Resource Usage ───────────────────────────────────────────────
free -h
df -h /opt/zimbra
uptime
top -u zimbra

# ── Process Check ────────────────────────────────────────────────
ps aux | grep zimbra | grep -v grep

# ── LDAP Health ──────────────────────────────────────────────────
su - zimbra -c "ldap status"

# ── Antispam/Antivirus ───────────────────────────────────────────
su - zimbra -c "zmantivirusctl status"
su - zimbra -c "zmantispamctl status"

# ── Certificates ─────────────────────────────────────────────────
openssl s_client -connect 10.10.10.110:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates

# ── SMTP Banner Test ─────────────────────────────────────────────
echo "QUIT" | nc 10.10.10.110 25
# Expected: 220 zimbra.accesswt.com ESMTP Postfix

# ── Disk Usage per Mailbox ───────────────────────────────────────
du -sh /opt/zimbra/store/0/*/
```

### 13.2 Axigen Monitoring

```bash
# ── Service Status ──────────────────────────────────────────────
sudo systemctl status axigen

# ── Port Verification ────────────────────────────────────────────
sudo ss -tlnp | grep axigen
# Shows all listening ports: 25, 143, 443, 465, 993, 9000, 9443, 7000, 8888

sudo netstat -tlnp | grep -E ':25|:443|:993|:9000|:9443'

# ── Process Check ────────────────────────────────────────────────
ps aux | grep axigen | grep -v grep

# ── Log Files ────────────────────────────────────────────────────
# Find all Axigen logs
sudo find /var/log/axigen -type f 2>/dev/null
sudo ls -la /var/log/axigen/

# SMTP incoming log
sudo tail -f /var/log/axigen/smtp-in.log

# SMTP outgoing log
sudo tail -f /var/log/axigen/smtp-out.log

# General log
sudo tail -f /var/log/axigen/axigen.log

# Delivery log
sudo tail -f /var/log/axigen/delivery.log

# ── SMTP Banner Test ─────────────────────────────────────────────
echo "QUIT" | nc 10.10.10.111 25
# Expected: 220 axigen.accesswt.com Axigen ESMTP ready

# ── IMAPS Certificate Check ──────────────────────────────────────
openssl s_client -connect 10.10.10.111:993 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates

# ── WebAdmin Health Check ─────────────────────────────────────────
curl -sk https://10.10.10.111:9443 | grep -i "axigen"
# Expected: HTML with Axigen WebAdmin title

# ── Mailbox File Check ───────────────────────────────────────────
sudo find /var/opt/axigen/domains -name "*.eml" | wc -l
# Total message files across all domains

sudo du -sh /var/opt/axigen/domains/*/
# Disk usage per client domain

# ── IMAP Mailbox Check via Python ────────────────────────────────
python3 << 'EOF'
import imaplib
mail = imaplib.IMAP4_SSL("10.10.10.111", 993)
mail.login("admin@school.accesswt.com", "your_password")
status, data = mail.select("INBOX", readonly=True)
print(f"INBOX message count: {data[0].decode()}")
mail.logout()
EOF

# ── Greylisting Status ───────────────────────────────────────────
# Check WebAdmin → Security & Filtering → Additional AntiSpam Methods

# ── Resource Usage ───────────────────────────────────────────────
free -h
df -h /var/opt/axigen
uptime

# ── Config Backup ────────────────────────────────────────────────
sudo tar -czf /root/axigen-backup-$(date +%F).tar.gz /var/opt/axigen/
```

### 13.3 DNS Monitoring (ns1)

```bash
# Service status
sudo systemctl status bind9

# Check all client domains resolve correctly
for DOMAIN in school fitness consultancy microfinance dairy; do
  echo -n "${DOMAIN}.accesswt.com MX → "
  dig @10.10.10.106 ${DOMAIN}.accesswt.com MX +short
done

# Verify zone transfer to ns2
dig @10.10.10.107 school.accesswt.com MX +short

# Check BIND is listening
sudo ss -tlnp | grep named

# BIND log
sudo tail -f /var/log/syslog | grep named

# Reload zone safely (dynamic zone)
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
sudo rndc freeze accesswt.com
sudo rndc reload accesswt.com
sudo rndc thaw accesswt.com
```

### 13.4 Proxmox Host (awtserver) Monitoring

```bash
# Check NAT rules active
iptables -t nat -L PREROUTING -n --line-numbers | grep -E '110|111'

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward   # must be 1

# Check VMs are reachable
ping -c 2 10.10.10.110   # zimbra
ping -c 2 10.10.10.111   # axigen
ping -c 2 10.10.10.106   # ns1

# Check all VM SSH ports
for PORT in 2106 2107 2108 2109 2110 2111; do
  nc -zv -w2 192.168.1.3 $PORT 2>&1 | grep -E "open|refused"
done

# Proxmox VM resource usage
qm list
qm status 110
qm status 111
```

---

## Quick Reference Card

### Access URLs
| Service | URL | Credentials |
|---|---|---|
| Axigen WebAdmin | https://192.168.1.3:9443 | admin / (set during install) |
| Axigen Webmail | https://192.168.1.3:443 | user@domain / password |
| Zimbra WebAdmin | https://192.168.1.3:7071 | admin@zimbra.accesswt.com |
| Zimbra Webmail | https://192.168.1.3:8443 | user@domain / password |

### Client Mail Settings (for Outlook/Thunderbird)
| Setting | Axigen | Zimbra |
|---|---|---|
| IMAP Server | axigen.accesswt.com | zimbra.accesswt.com |
| IMAP Port | 993 (SSL) | 993 (SSL) |
| SMTP Server | axigen.accesswt.com | zimbra.accesswt.com |
| SMTP Port | 587 (STARTTLS) | 587 (STARTTLS) |

### Emergency Recovery
```bash
# Zimbra — restart all services
sudo su - zimbra -c "zmcontrol stop && zmcontrol start"

# Axigen — restart service
sudo systemctl restart axigen

# DNS — restart BIND
sudo systemctl restart bind9

# Proxmox — reapply NAT rules without reboot
ifdown vmbr1 && ifup vmbr1
```

---
*Document generated: July 2026*
*Environment: accesswt.com internal lab — Proxmox PVE 9.2.2*
*Servers: Zimbra 10.1.16 (Rocky Linux 9.8) + Axigen 10.6.35 (Ubuntu 24.04)*

---

## 14. Troubleshooting

### 14.1 Common Failure Points — Both Environments

The table below lists every point in the mail flow where delivery can fail, the symptom we will see, and how to diagnose and fix it.

#### Failure Point 1 — Browser Cannot Reach Webmail

**Symptom:** `ERR_CONNECTION_REFUSED` or browser times out

```bash
# On awtserver — is DNAT rule present?
iptables -t nat -L PREROUTING -n --line-numbers | grep -E "8443|443|9443"

# Is IP forwarding on?
cat /proc/sys/net/ipv4/ip_forward
# Must output: 1

# Is VM reachable at all?
ping -c 3 10.10.10.110   # Zimbra
ping -c 3 10.10.10.111   # Axigen

# Is the port actually listening on the VM?
nc -zv 10.10.10.110 8443
nc -zv 10.10.10.111 443

# Fix — re-apply DNAT rules immediately
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 8443 \
  -j DNAT --to-destination 10.10.10.110:8443
iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 443 \
  -j DNAT --to-destination 10.10.10.111:443
```

*[INSERT SCREENSHOT: iptables -t nat -L PREROUTING -n showing all DNAT rules for 110 and 111 after fix]*

---

#### Failure Point 2 — TLS Certificate Warning in Browser

**Symptom:** Browser shows "Your connection is not private" or certificate error

```bash
# Check what certificate is being served
openssl s_client -connect 10.10.10.111:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Check if cert is expired
openssl x509 -in /var/opt/axigen/axigenmail.pem -noout -dates 2>/dev/null
# Look at notAfter — must be a future date

# Check cert has correct CN
openssl x509 -in /var/opt/axigen/axigenmail.pem -noout -subject
# Expected: CN=axigen.accesswt.com

# Fix Axigen — regenerate self-signed cert
sudo openssl req -x509 -nodes -days 730 -newkey rsa:2048 \
  -keyout /var/opt/axigen/ssl/axigen.key \
  -out /var/opt/axigen/ssl/axigen.crt \
  -subj "/C=NP/ST=Bagmati/L=Kathmandu/O=AccessWT/CN=axigen.accesswt.com"
sudo bash -c 'cat /var/opt/axigen/ssl/axigen.crt \
  /var/opt/axigen/ssl/axigen.key > /var/opt/axigen/axigenmail.pem'
sudo chown axigen:axigen /var/opt/axigen/axigenmail.pem
sudo chmod 600 /var/opt/axigen/axigenmail.pem
sudo systemctl restart axigen

# Fix Zimbra — redeploy self-signed cert
su - zimbra
zmcertmgr deploycrt self
zmcontrol restart
```

---

#### Failure Point 3 — Login Fails (Correct Credentials)

**Symptom:** Username/password correct but login rejected

```bash
# Zimbra — check LDAP is running (auth depends on LDAP)
su - zimbra
ldap status
zmprov gad   # Should return domain list without errors

# If LDAP is down — restart it first
ldap start
sleep 5
zmcontrol start

# Check account status (may be locked)
zmprov ga admin@consultancy.accesswt.com | grep zimbraAccountStatus
# Must show: zimbraAccountStatus: active

# Unlock account
zmprov ma admin@consultancy.accesswt.com zimbraAccountStatus active

# Reset password
zmprov sp admin@consultancy.accesswt.com 'NewP@ss2026!'

# Axigen — test login via IMAP
python3 << 'EOF'
import imaplib
try:
    m = imaplib.IMAP4_SSL("10.10.10.111", 993)
    m.login("admin@school.accesswt.com", "your_password")
    print("Login OK")
    m.logout()
except Exception as e:
    print(f"Login failed: {e}")
EOF
```

---

#### Failure Point 4 — DNS Resolution Fails

**Symptom:** Postfix logs `Host or domain name not found` or `Name service error`

```bash
# Test resolution from Zimbra VM
dig @10.10.10.108 school.accesswt.com MX
# Expected: ANSWER SECTION with axigen.accesswt.com

# If NXDOMAIN or SERVFAIL:
# Check BIND is running on ns1
ssh bsuman@10.10.10.106
sudo systemctl status bind9

# Check zone loaded
sudo rndc status | grep zones

# Check zone syntax
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
# Must show: OK

# Reload zone (dynamic zone safe method)
sudo rndc freeze accesswt.com
sudo rndc reload accesswt.com
sudo rndc thaw accesswt.com

# If journal conflict blocks reload
sudo systemctl stop bind9
sudo rm /etc/bind/zones/db.accesswt.com.jnl
sudo systemctl start bind9

# Re-test
dig @10.10.10.106 school.accesswt.com MX +short
# Expected: 10 axigen.accesswt.com.
```

*[INSERT SCREENSHOT: dig @10.10.10.106 school.accesswt.com MX showing correct ANSWER SECTION after fix]*

---

#### Failure Point 5 — SMTP Connection Refused (Port 25)

**Symptom:** Postfix logs `Connection refused` to 10.10.10.111:25

```bash
# Test port directly
nc -zv 10.10.10.111 25
echo "QUIT" | nc 10.10.10.111 25
# Expected: 220 axigen.accesswt.com Axigen ESMTP ready

# If connection refused:
# Check Axigen is running
sudo systemctl status axigen
ps aux | grep axigen | grep -v grep

# Check port is listening
sudo ss -tlnp | grep ":25"

# Check ufw is not blocking
sudo ufw status | grep 25
# Fix if blocked:
sudo ufw allow 25/tcp

# Check SMTP Receiving service in Axigen
# WebAdmin --> SERVICES --> Services Management --> SMTP Receiving --> Start if stopped

# Restart Axigen if needed
sudo systemctl restart axigen
```

---

#### Failure Point 6 — Mail Stuck in Postfix Queue

**Symptom:** `postqueue -p` shows messages in deferred state

```bash
su - zimbra

# View stuck messages with reasons
postqueue -p

# Get detailed diagnosis for specific message
postcat -q QUEUEID | grep -A 5 "Diagnostic\|error\|reject\|refused"

# Common reasons and fixes:

# Reason: "Connection refused" (port 25 closed on Axigen)
#   Fix: sudo ufw allow 25/tcp  (on Axigen)

# Reason: "Host or domain name not found"
#   Fix: Fix DNS -- see Failure Point 4

# Reason: "451 Temporary rejected by greylisting"
#   Normal -- Postfix retries automatically in ~5 min
#   Force immediate retry: postqueue -f

# Reason: "550 User unknown / No such user"
#   Fix: Create the recipient account in Axigen WebAdmin

# Reason: "452 Insufficient storage"
#   Fix: Check disk space on Axigen: df -h /var/opt/axigen

# Force retry all deferred immediately
postqueue -f

# Watch queue clear in real time
watch -n2 "postqueue -p | wc -l"
```

*[INSERT SCREENSHOT: postqueue -p showing deferred message with diagnostic reason, then empty queue after postqueue -f]*

---

#### Failure Point 7 — Greylisting Blocking (Stuck, Not Retrying)

**Symptom:** Same message gets 451 repeatedly, never accepted

```bash
# Normal behavior: Postfix retries after ~5 min and succeeds
# Abnormal: retries continue getting 451 indefinitely

# Check retry schedule on Zimbra
postconf minimal_backoff_time   # default 300 sec
postconf maximal_backoff_time   # default 4000 sec

# Check greylisting whitelist has not expired on Axigen
# WebAdmin --> Security & Filtering --> Additional AntiSpam Methods --> Greylisting
# View: whitelisted tuples

# Temporarily disable greylisting for testing
# WebAdmin --> Additional AntiSpam Methods --> Greylisting --> Disable

# Force immediate retry
su - zimbra
postqueue -f
```

---

#### Failure Point 8 — Mail Delivered to Spam Folder

**Symptom:** Mail arrives but goes to Spam/Junk, not Inbox

```bash
# Check SPF record exists and is correct
dig @10.10.10.106 consultancy.accesswt.com TXT +short | grep spf
# Expected: v=spf1 mx ip4:10.10.10.110 ~all

# Check DMARC record exists
dig @10.10.10.106 _dmarc.consultancy.accesswt.com TXT +short
# Expected: v=DMARC1; p=quarantine

# Check antispam score (Zimbra)
sudo grep "SA score\|spam score" /var/log/zimbra.log | tail -10

# Check if Axigen antispam is over-aggressive
# WebAdmin --> Security & Filtering --> AntiVirus & AntiSpam
# Review: spam threshold score, whitelist the sending domain

# Check message headers for Authentication-Results
# In webmail: open message --> View Source / Show Original
# Look for: spf=pass, dkim=pass, dmarc=pass
```

---

#### Failure Point 9 — Mail Body Empty in Webmail

**Symptom:** Email arrives, subject visible, but body is blank

```bash
# This is an RFC822 formatting issue
# The message MUST have a blank line between headers and body

# WRONG (no blank line between header and body):
echo "Subject: Test
From: sender@example.com
To: recipient@example.com
This is the body"

# CORRECT (blank line separates headers from body):
echo "Subject: Test
From: sender@example.com
To: recipient@example.com

This is the body"

# Fix for nc/telnet sending:
# After typing DATA and seeing 354 response,
# type headers, then press Enter TWICE before body text

# Fix for Python script:
# MIMEText automatically handles the blank line separator
# Use: msg.attach(MIMEText("body text", "plain"))
# Never build raw RFC822 manually unless we know exactly what we are doing
```

---

#### Failure Point 10 — STARTTLS Fails (TLS Handshake Error)

**Symptom:** Postfix logs `TLS handshake failed` or `no shared cipher`

```bash
# Test STARTTLS manually
openssl s_client -connect 10.10.10.111:25 -starttls smtp 2>&1 \
  | grep -E "error|alert|Protocol|Cipher|subject"

# Check if cert file is valid
sudo openssl x509 -in /var/opt/axigen/axigenmail.pem -noout -text 2>&1 \
  | grep -E "Subject|Issuer|Not Before|Not After|Public Key"

# Check cert contains both cert and key
sudo grep -c "BEGIN CERTIFICATE" /var/opt/axigen/axigenmail.pem  # must be 1
sudo grep -c "BEGIN PRIVATE KEY" /var/opt/axigen/axigenmail.pem  # must be 1

# If either count is 0 -- cert/key not combined correctly
sudo bash -c 'cat /var/opt/axigen/ssl/axigen.crt \
  /var/opt/axigen/ssl/axigen.key > /var/opt/axigen/axigenmail.pem'
sudo chown axigen:axigen /var/opt/axigen/axigenmail.pem
sudo chmod 600 /var/opt/axigen/axigenmail.pem
sudo systemctl restart axigen

# Verify fix
openssl s_client -connect 10.10.10.111:25 -starttls smtp 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

---

### 14.2 Zimbra-Specific Troubleshooting

#### Check All Services

```bash
su - zimbra

# Full status
zmcontrol status
# Every service should show: Running

# Quick restart of one failed service
zmcontrol restart mailbox
zmcontrol restart mta
zmcontrol restart ldap
zmcontrol restart proxy
zmcontrol restart antispam
zmcontrol restart antivirus

# Full stop and start (use only when multiple services down)
zmcontrol stop
sleep 10
zmcontrol start
zmcontrol status
```

#### Postfix Troubleshooting

```bash
su - zimbra

# View mail queue
postqueue -p

# Read a specific queued message
postcat -q QUEUEID

# Check Postfix routing config
postconf mydestination
postconf relay_domains
postconf mynetworks
postconf smtpd_relay_restrictions

# Test local send
echo "Subject: Test

Body line" | /opt/zimbra/common/sbin/sendmail \
  -f info@consultancy.accesswt.com \
  admin@microfinance.accesswt.com

# Watch Postfix log
sudo tail -f /var/log/zimbra.log | grep postfix

# Flush deferred queue
postqueue -f

# Delete specific stuck message
postsuper -d QUEUEID

# Check Postfix config for errors
postfix check 2>&1
```

#### Mailbox / Jetty Troubleshooting

```bash
su - zimbra

# Check Jetty is accepting connections
nc -zv localhost 7025   # LMTP (internal delivery)
nc -zv localhost 8080   # HTTP (internal)

# Search mailbox from CLI
zmmailbox -z -m user@domain.com search "in:inbox"
zmmailbox -z -m user@domain.com search "in:inbox subject:Confidential"
zmmailbox -z -m user@domain.com search "in:sent"
zmmailbox -z -m user@domain.com search "in:trash"

# Get mailbox statistics
zmmailbox -z -m user@domain.com getMailboxStats

# Check errors in mailbox log
sudo grep -i "ERROR\|FATAL\|Exception" /opt/zimbra/log/mailbox.log \
  | grep "$(date +%Y-%m-%d)" | tail -20
```

#### Account Management

```bash
su - zimbra

# List all accounts
zmprov -l gaa

# List accounts per domain
zmprov -l gaa consultancy.accesswt.com

# Check account details
zmprov ga admin@consultancy.accesswt.com \
  | grep -E "mail:|zimbraAccountStatus|zimbraMailQuota"

# Account status values:
#   active   = normal
#   locked   = locked out (too many failed logins)
#   closed   = disabled
#   maintenance = maintenance mode

# Unlock a locked account
zmprov ma user@domain.com zimbraAccountStatus active

# Reset password
zmprov sp user@domain.com 'NewP@ss2026!'

# Set mailbox quota (bytes, 0 = unlimited)
zmprov ma user@domain.com zimbraMailQuota 1073741824   # 1 GB
```

#### Certificate Management

```bash
su - zimbra

# View current deployed cert
zmcertmgr viewdeployedcrt mailboxd

# Check expiry
echo | openssl s_client -connect localhost:8443 2>/dev/null \
  | openssl x509 -noout -dates

# Redeploy self-signed (lab)
zmcertmgr deploycrt self
zmcontrol restart

# Deploy commercial/Let's Encrypt cert (production)
# First install cert files, then:
zmcertmgr verifycrt comm /path/cert.pem /path/key.pem /path/chain.pem
zmcertmgr deploycrt comm /path/cert.pem /path/key.pem /path/chain.pem
zmcontrol restart
```

#### Zimbra Log Files

```bash
# Main log — MTA, antispam, delivery (most useful)
sudo tail -f /var/log/zimbra.log

# Mailbox/Jetty — webmail, SOAP, account operations
sudo tail -f /opt/zimbra/log/mailbox.log

# nginx proxy access
sudo tail -f /opt/zimbra/log/nginx.access.log

# nginx proxy errors
sudo tail -f /opt/zimbra/log/nginx.error.log

# Admin console audit trail
sudo tail -f /opt/zimbra/log/audit.log

# Search for errors from today
sudo grep -i "ERROR\|FATAL" /opt/zimbra/log/mailbox.log \
  | grep "$(date +%Y-%m-%d)" | tail -30

sudo grep "$(date +%b %e)" /var/log/zimbra.log \
  | grep -i "error\|fatal\|reject" | tail -30
```

---

### 14.3 Axigen-Specific Troubleshooting

#### Service and Process Checks

```bash
# Check service status
sudo systemctl status axigen

# Check Axigen process is running
ps aux | grep axigen | grep -v grep
# Expected: axigen process + axigen-tnef process

# Check all ports Axigen should be listening on
sudo ss -tlnp | grep axigen
# Expected ports: 25, 80, 143, 443, 465, 993, 9000, 9443, 7000, 8888

# Restart Axigen
sudo systemctl stop axigen
sleep 3
sudo systemctl start axigen
sudo systemctl status axigen

# Check startup errors in system journal
sudo journalctl -u axigen --since "10 minutes ago" | tail -30
```

#### SMTP Tests

```bash
# Test SMTP banner
echo "QUIT" | nc 10.10.10.111 25
# Expected: 220 axigen.accesswt.com Axigen ESMTP ready

# Test full SMTP conversation with swaks
swaks --to admin@school.accesswt.com \
      --from test@accesswt.com \
      --server 10.10.10.111 \
      --port 25 \
      --header "Subject: Axigen SMTP Test" \
      --body "This is an Axigen SMTP test message." \
      --verbose

# Test STARTTLS specifically
swaks --to admin@school.accesswt.com \
      --from test@accesswt.com \
      --server 10.10.10.111 \
      --port 25 \
      --tls \
      --verbose

# Watch SMTP log during test
sudo tail -f /var/log/axigen/smtp-in.log
```

#### IMAP Tests

```bash
# Test IMAP SSL connection
openssl s_client -connect 10.10.10.111:993 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates

# Full IMAP test via Python
python3 << 'EOF'
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "admin@school.accesswt.com"
PASSWORD    = "your_password"

try:
    mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
    print(f"Connected to {IMAP_SERVER}:{IMAP_PORT}")

    mail.login(ACCOUNT, PASSWORD)
    print(f"Logged in as {ACCOUNT}")

    status, data = mail.list()
    print(f"Folders available: {len(data)}")
    for folder in data:
        print(f"  {folder.decode()}")

    mail.select("INBOX", readonly=True)
    status, data = mail.search(None, "ALL")
    msg_ids = data[0].split()
    print(f"INBOX messages: {len(msg_ids)}")

    mail.logout()
    print("Test PASSED")

except Exception as e:
    print(f"Test FAILED: {e}")
EOF
```

#### Account and Domain Checks

```bash
# Verify domain directory exists
sudo ls /var/opt/axigen/domains/
# Expected: accesswt.com, school.accesswt.com, fitness.accesswt.com

# Verify account directory exists
sudo ls /var/opt/axigen/domains/school.accesswt.com/accounts/
# Expected: admin, info, principal, postmaster

# Verify INBOX directory exists
sudo ls /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX/
# Expected: .eml files for delivered messages

# Count messages per user
for USER in admin info principal; do
    COUNT=$(sudo ls /var/opt/axigen/domains/school.accesswt.com/accounts/${USER}/INBOX/ 2>/dev/null | wc -l)
    echo "${USER}@school.accesswt.com: ${COUNT} messages in INBOX"
done

# Check disk usage per domain
sudo du -sh /var/opt/axigen/domains/*/
```

#### Delivery Verification

```bash
# Send a test message and verify delivery
swaks --to admin@school.accesswt.com \
      --from test@accesswt.com \
      --server 10.10.10.111 \
      --header "Subject: Delivery Test $(date)" \
      --body "Delivery test."

# Wait 5 seconds then check
sleep 5

# Check via filesystem
sudo find /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX \
  -name "*.eml" -newer /tmp -ls 2>/dev/null

# Check via IMAP
python3 << 'EOF'
import imaplib
m = imaplib.IMAP4_SSL("10.10.10.111", 993)
m.login("admin@school.accesswt.com", "your_password")
m.select("INBOX", readonly=True)
s, d = m.search(None, 'SUBJECT "Delivery Test"')
print(f"Delivery test messages found: {len(d[0].split())}")
m.logout()
EOF
```

#### Axigen Log Files

```bash
# Discover all log files
sudo find /var/log/axigen -type f -name "*.log" 2>/dev/null

# SMTP incoming — most useful for delivery issues
sudo tail -f /var/log/axigen/smtp-in.log

# SMTP outgoing — for sent mail issues
sudo tail -f /var/log/axigen/smtp-out.log

# Main Axigen application log
sudo tail -f /var/log/axigen/axigen.log

# Delivery log (local delivery results)
sudo tail -f /var/log/axigen/delivery.log 2>/dev/null

# Search all logs for errors today
sudo find /var/log/axigen -name "*.log" 2>/dev/null \
  | xargs grep -i "error\|fail\|reject" 2>/dev/null \
  | grep "$(date +%Y)" | tail -30
```

---

### 14.4 Step-by-Step Diagnostic Runbook

Run these in order when a user reports mail not arriving. Stop at the first failure.

```bash
echo "=== DIAGNOSTIC RUNBOOK: Mail Not Arriving ==="
echo "Sender: info@consultancy.accesswt.com (Zimbra)"
echo "Recipient: admin@school.accesswt.com (Axigen)"
echo ""

echo "--- STEP 1: Is Zimbra MTA running? ---"
sudo su - zimbra -c "zmcontrol status" | grep -E "mta|mailbox" | grep -v Running && \
  echo "PROBLEM: Service down" || echo "OK: Zimbra services running"

echo ""
echo "--- STEP 2: Did message leave Zimbra queue? ---"
sudo su - zimbra -c "postqueue -p" | grep -c "^[A-F0-9]" | \
  xargs -I{} bash -c '[ {} -gt 0 ] && echo "PROBLEM: {} messages stuck in queue" || echo "OK: Queue empty"'

echo ""
echo "--- STEP 3: Did Postfix connect to Axigen? ---"
sudo grep "school.accesswt.com\|axigen\|status=" /var/log/zimbra.log \
  | tail -5

echo ""
echo "--- STEP 4: Did Axigen receive the SMTP session? ---"
sudo grep "10.10.10.110\|zimbra" /var/log/axigen/smtp-in.log 2>/dev/null \
  | tail -5

echo ""
echo "--- STEP 5: Did Axigen accept and queue the message? ---"
sudo grep "queued\|250 Mail\|accepted" /var/log/axigen/smtp-in.log 2>/dev/null \
  | tail -5

echo ""
echo "--- STEP 6: Is the .eml file in the mailbox? ---"
sudo find /var/opt/axigen/domains/school.accesswt.com/accounts/admin/INBOX \
  -name "*.eml" 2>/dev/null | wc -l | xargs -I{} echo "{} message files in INBOX"

echo ""
echo "--- STEP 7: Does IMAP confirm the message? ---"
python3 -c "
import imaplib
try:
    m = imaplib.IMAP4_SSL('10.10.10.111', 993)
    m.login('admin@school.accesswt.com', 'your_password')
    m.select('INBOX', readonly=True)
    s, d = m.search(None, 'ALL')
    print(f'OK: IMAP INBOX contains {len(d[0].split())} messages')
    m.logout()
except Exception as e:
    print(f'PROBLEM: {e}')
"

echo ""
echo "--- STEP 8: Is webmail accessible? ---"
curl -sk https://10.10.10.111:443/webmail 2>/dev/null | grep -c "axigen" | \
  xargs -I{} bash -c '[ {} -gt 0 ] && echo "OK: Webmail responding" || echo "PROBLEM: Webmail not responding"'
```

---

### 14.5 Quick Reference Card

| Symptom | First Check | Command | Fix |
|---|---|---|---|
| Browser can't connect | VM running? | `ping 10.10.10.110` | Start VM |
| DNAT not working | Rule present? | `iptables -t nat -L PREROUTING -n` | Re-apply DNAT rule |
| Login fails | LDAP up? | `su - zimbra && ldap status` | `ldap start` |
| Account locked | Status check | `zmprov ga user@domain` | `zmprov ma user zimbraAccountStatus active` |
| Mail stuck in queue | Queue check | `postqueue -p` | `postqueue -f` or fix DNS |
| 451 greylisting | Normal first time | `postqueue -p` | Wait 5 min or `postqueue -f` |
| 550 user unknown | Account exists? | Check Axigen WebAdmin | Create account |
| Port 25 refused | Port listening? | `nc -zv 10.10.10.111 25` | `sudo ufw allow 25/tcp` |
| SMTP banner missing | Service up? | `systemctl status axigen` | `systemctl restart axigen` |
| IMAPS fails | Port listening? | `ss -tlnp \| grep 993` | Start IMAP service |
| Cert warning | Cert valid? | `openssl x509 -in cert -noout -dates` | Renew or regenerate cert |
| STARTTLS error | Cert+key combined? | `grep -c CERTIFICATE axigenmail.pem` | Recombine cert+key |
| Mail in spam | SPF correct? | `dig TXT consultancy.accesswt.com` | Fix SPF record |
| Body empty | Blank line? | Check RFC822 format | Add blank line after headers |
| DNS not resolving | BIND running? | `systemctl status bind9` | Restart BIND |
| Zone not reloading | Journal conflict? | `rndc freeze/reload/thaw` | Delete .jnl file |
| Mail not in webmail | File exists? | `find /var/opt/axigen/...INBOX -name *.eml` | Restart Axigen |
| Disk full | Disk usage | `df -h /var/opt/axigen` | Clean old mail/logs |

---

## Lab vs Production Comparison Summary

| Item | Lab | Production |
|---|---|---|
| User URL | `https://192.168.1.3:8443` | `https://mail.consultancy.accesswt.com` |
| DNS | Internal BIND only | Public DNS (Cloudflare/Route53) |
| Certificate | Self-signed (browser warns) | Let's Encrypt (trusted) |
| PTR record | Not needed | Required by Gmail/Outlook |
| DKIM | Optional/internal | Required for delivery to major providers |
| SPF | Internal DNS | Public DNS |
| DMARC | Internal DNS | Public DNS |
| RBL checks | Not applicable | Active (Spamhaus, SpamCop) |
| FCrDNS | Not needed | Required |
| Firewall | ufw + iptables DNAT | Hardware/cloud firewall |
| Port 25 access | Internal only | Open to internet |
| Backup MX | Not configured | Recommended |
| IP warming | Not needed | Required for new IPs |
| Monitoring | Manual CLI | Automated (Zabbix, Grafana, Nagios) |
| Blacklist check | Not needed | Daily via mxtoolbox.com |
| TTL | 604800 (1 week) | 300-3600 (5 min to 1 hour) |

---
*Mail Flow Complete Reference — Version 1.0*
*July 2026 | accesswt.com Lab Environment*
*Zimbra 10.1.16 (Rocky Linux 9.8) + Axigen 10.6.35 (Ubuntu 24.04)*
## 15. Axigen Mail Operations — Complete Script Reference

All scripts reside on the Axigen VM at `/tmp/`. They use Python3 IMAP/SMTP — no third-party libraries required.

### 14.1 Script 1 — Check ANY Mail in a Specific User

**Purpose:** Shows all folders and message counts for one Axigen account. Run this before any delete operation to confirm the mailbox state.

```bash
cat > /tmp/axigen_check_any_mail.py << 'EOF'
#!/usr/bin/env python3
"""
Check if ANY mail exists in a specific Axigen mailbox.
Shows folder list + message count per folder.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "principal@school.accesswt.com"
PASSWORD    = "your_password"

def check_any_mail(account, password):
    print(f"{'='*55}")
    print(f"Server  : {IMAP_SERVER}:{IMAP_PORT}")
    print(f"Account : {account}")
    print(f"{'='*55}")

    mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
    mail.login(account, password)
    print(f"✓ Login successful\n")

    # List all folders
    status, folders = mail.list()
    print("=== FOLDERS FOUND ===")
    folder_names = []
    for folder in folders:
        name = folder.decode().split('"/"')[-1].strip().strip('"')
        folder_names.append(name)
        print(f"  📁 {name}")

    # Check message count per folder
    print("\n=== MAIL COUNT PER FOLDER ===")
    total = 0
    for name in folder_names:
        try:
            status, data = mail.select(name, readonly=True)
            if status == "OK":
                count = int(data[0])
                total += count
                icon = "📬" if count > 0 else "📭"
                print(f"  {icon} {name}: {count} message(s)")
        except Exception:
            pass

    print(f"\n=== SUMMARY ===")
    print(f"Total messages across all folders: {total}")
    mail.logout()

check_any_mail(ACCOUNT, PASSWORD)
EOF

python3 /tmp/axigen_check_any_mail.py
```

**Expected output:**
```
=======================================================
Server  : 10.10.10.111:993
Account : principal@school.accesswt.com
=======================================================
✓ Login successful

=== FOLDERS FOUND ===
  📁 INBOX
  📁 Sent
  📁 Drafts
  📁 Trash
  📁 Spam

=== MAIL COUNT PER FOLDER ===
  📬 INBOX: 3 message(s)
  📬 Sent: 0 message(s)
  📭 Drafts: 0 message(s)
  📭 Trash: 0 message(s)
  📭 Spam: 0 message(s)

=== SUMMARY ===
Total messages across all folders: 3
```

*[INSERT SCREENSHOT: Terminal output of axigen_check_any_mail.py showing folder list and message counts for principal@school.accesswt.com]*

---

### 14.2 Script 2 — Check SPECIFIC Mail in a Specific User

**Purpose:** Search for a message by subject in INBOX and Trash. Shows full message headers if found.

```bash
cat > /tmp/axigen_check_specific_mail.py << 'EOF'
#!/usr/bin/env python3
"""
Check if a SPECIFIC mail (by subject) exists in a specific Axigen mailbox.
Checks both INBOX and Trash. Shows message headers if found.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "principal@school.accesswt.com"
PASSWORD    = "your_password"
SUBJECT     = "Confidential"

def check_specific_mail(account, password, subject):
    print(f"{'='*55}")
    print(f"Server  : {IMAP_SERVER}:{IMAP_PORT}")
    print(f"Account : {account}")
    print(f"Subject : '{subject}'")
    print(f"{'='*55}")

    mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
    mail.login(account, password)
    print(f"✓ Login successful\n")

    # Search INBOX
    mail.select("INBOX", readonly=True)
    status, data = mail.search(None, f'SUBJECT "{subject}"')
    msg_ids = data[0].split()

    print(f"=== INBOX ===")
    print(f"Messages matching '{subject}': {len(msg_ids)}")

    if msg_ids:
        print(f"\n=== MESSAGE DETAILS ===")
        for msg_id in msg_ids:
            _, msg_data = mail.fetch(
                msg_id,
                "(BODY[HEADER.FIELDS (SUBJECT FROM TO DATE MESSAGE-ID)])"
            )
            print(f"\n--- Message ID: {msg_id.decode()} ---")
            print(msg_data[0][1].decode().strip())
        print(f"\n✓ FOUND — mail EXISTS in {account}")
    else:
        print(f"\n✗ NOT FOUND — no mail with subject '{subject}' in INBOX")

    # Also check Trash
    print(f"\n=== TRASH ===")
    try:
        mail.select("Trash", readonly=True)
        status, data = mail.search(None, f'SUBJECT "{subject}"')
        trash_ids = data[0].split()
        print(f"Messages matching '{subject}' in Trash: {len(trash_ids)}")
    except Exception:
        print("Could not access Trash folder")

    mail.logout()

check_specific_mail(ACCOUNT, PASSWORD, SUBJECT)
EOF

python3 /tmp/axigen_check_specific_mail.py
```

**Expected output (when mail exists):**
```
=======================================================
Server  : 10.10.10.111:993
Account : principal@school.accesswt.com
Subject : 'Confidential'
=======================================================
✓ Login successful

=== INBOX ===
Messages matching 'Confidential': 1

=== MESSAGE DETAILS ===
--- Message ID: 5 ---
Subject: Confidential
From: info@consultancy.accesswt.com
To: principal@school.accesswt.com
Date: Tue, 01 Jul 2026 07:02:14 +0000

✓ FOUND — mail EXISTS in principal@school.accesswt.com

=== TRASH ===
Messages matching 'Confidential' in Trash: 0
```

*[INSERT SCREENSHOT: Terminal output of axigen_check_specific_mail.py showing FOUND with message headers]*

---

### 14.3 Script 3 — Bulk Move to Trash (All Accounts)

**Purpose:** Moves all matching messages from INBOX to Trash across ALL Axigen accounts. Equivalent to Zimbra's `deleteConversation` loop. Shows BEFORE/AFTER count per account.

```bash
cat > /tmp/axigen_move_to_trash.py << 'EOF'
#!/usr/bin/env python3
"""
Move ALL matching messages (by subject) to Trash
across ALL specified Axigen mailboxes.
Same behavior as Zimbra's bulk deleteConversation script.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
SUBJECT     = "Confidential"
TRASH       = "Trash"

# All Axigen accounts — fill in actual passwords
ACCOUNTS = [
    ("admin@school.accesswt.com",        "your_password"),
    ("info@school.accesswt.com",         "your_password"),
    ("principal@school.accesswt.com",    "your_password"),
    ("admin@fitness.accesswt.com",       "your_password"),
    ("info@fitness.accesswt.com",        "your_password"),
    ("membership@fitness.accesswt.com",  "your_password"),
]

def move_to_trash(account, password):
    try:
        mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
        mail.login(account, password)

        # BEFORE — count in INBOX
        mail.select("INBOX", readonly=True)
        status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
        before_ids = data[0].split()
        print(f"  BEFORE : {len(before_ids)} matching message(s) in INBOX")

        if not before_ids:
            print(f"  ✓ Nothing to move")
            mail.logout()
            return 0

        # ACTION — mark deleted + expunge (moves to Trash in Axigen)
        mail.select("INBOX")
        for msg_id in before_ids:
            mail.store(msg_id, "+FLAGS", "\\Deleted")
        mail.expunge()
        print(f"  ACTION : Marked \\Deleted + expunged from INBOX")

        # AFTER INBOX — verify gone from inbox
        mail.select("INBOX", readonly=True)
        status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
        after_inbox = data[0].split()
        print(f"  AFTER  : {len(after_inbox)} remaining in INBOX (should be 0)")

        # VERIFY IN TRASH
        try:
            mail.select(TRASH, readonly=True)
            status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
            trash_ids = data[0].split()
            print(f"  TRASH  : {len(trash_ids)} message(s) now in Trash ✓")
        except Exception as e:
            print(f"  TRASH  : Could not verify — {e}")

        mail.logout()
        return len(before_ids)

    except imaplib.IMAP4.error as e:
        print(f"  ✗ IMAP error: {e}")
        return 0
    except Exception as e:
        print(f"  ✗ Error: {e}")
        return 0

# MAIN
print(f"Axigen Bulk Move to Trash")
print(f"Subject : '{SUBJECT}'")
print(f"Server  : {IMAP_SERVER}:{IMAP_PORT}")
print(f"Accounts: {len(ACCOUNTS)}")
print("=" * 55)

total = 0
for account, password in ACCOUNTS:
    print(f"\n[{account}]")
    total += move_to_trash(account, password)

print(f"\n{'='*55}")
print(f"Complete — {total} message(s) moved to Trash across {len(ACCOUNTS)} mailboxes")
EOF

python3 /tmp/axigen_move_to_trash.py
```

**Expected output:**
```
Axigen Bulk Move to Trash
Subject : 'Confidential'
Server  : 10.10.10.111:993
Accounts: 6
=======================================================

[admin@school.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

[info@school.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

[principal@school.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

[admin@fitness.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

[info@fitness.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

[membership@fitness.accesswt.com]
  BEFORE : 1 matching message(s) in INBOX
  ACTION : Marked \Deleted + expunged from INBOX
  AFTER  : 0 remaining in INBOX (should be 0)
  TRASH  : 1 message(s) now in Trash ✓

=======================================================
Complete — 6 message(s) moved to Trash across 6 mailboxes
```

*[INSERT SCREENSHOT: axigen_move_to_trash.py output showing BEFORE/ACTION/AFTER/TRASH for each account with 0 remaining in INBOX]*

---

### 14.4 Script 4 — Empty Trash for Specific User

**Purpose:** Permanently deletes all items from Trash for a specific account. Shows item list before deletion and confirms 0 remaining after.

```bash
cat > /tmp/axigen_empty_trash.py << 'EOF'
#!/usr/bin/env python3
"""
Empty Trash folder for a specific Axigen user.
Shows contents before, empties permanently, verifies after.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "principal@school.accesswt.com"
PASSWORD    = "your_password"
TRASH       = "Trash"

def empty_trash(account, password):
    print(f"{'='*55}")
    print(f"Server  : {IMAP_SERVER}:{IMAP_PORT}")
    print(f"Account : {account}")
    print(f"Folder  : {TRASH}")
    print(f"{'='*55}")

    mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
    mail.login(account, password)
    print(f"✓ Login successful\n")

    # BEFORE
    mail.select(TRASH, readonly=True)
    status, data = mail.search(None, "ALL")
    before_ids = data[0].split()
    print(f"=== BEFORE ===")
    print(f"Items in Trash: {len(before_ids)}")

    if not before_ids:
        print(f"\n✓ Trash already empty — nothing to do")
        mail.logout()
        return

    # Show what's in Trash before deleting (max 10)
    print(f"\nItems that will be permanently deleted:")
    mail.select(TRASH, readonly=True)
    for msg_id in before_ids[:10]:
        _, msg_data = mail.fetch(
            msg_id,
            "(BODY[HEADER.FIELDS (SUBJECT FROM DATE)])"
        )
        print(f"  --- ID {msg_id.decode()} ---")
        print(f"  {msg_data[0][1].decode().strip()}")
    if len(before_ids) > 10:
        print(f"  ... and {len(before_ids) - 10} more")

    # ACTION — permanently delete
    print(f"\n=== ACTION: Emptying Trash ===")
    mail.select(TRASH)
    for msg_id in before_ids:
        mail.store(msg_id, "+FLAGS", "\\Deleted")
    mail.expunge()
    print(f"✓ All items permanently deleted (no recovery)")

    # AFTER
    mail.select(TRASH, readonly=True)
    status, data = mail.search(None, "ALL")
    after_ids = data[0].split()
    print(f"\n=== AFTER ===")
    print(f"Items in Trash: {len(after_ids)} (should be 0)")

    if len(after_ids) == 0:
        print(f"\n✓ Trash successfully emptied for {account}")
    else:
        print(f"\n⚠ Warning: {len(after_ids)} item(s) still remain")

    mail.logout()

empty_trash(ACCOUNT, PASSWORD)
EOF

python3 /tmp/axigen_empty_trash.py
```

**Expected output:**
```
=======================================================
Server  : 10.10.10.111:993
Account : principal@school.accesswt.com
Folder  : Trash
=======================================================
✓ Login successful

=== BEFORE ===
Items in Trash: 1

Items that will be permanently deleted:
  --- ID 1 ---
  Subject: Confidential
  From: info@consultancy.accesswt.com
  Date: Tue, 01 Jul 2026 07:02:14 +0000

=== ACTION: Emptying Trash ===
✓ All items permanently deleted (no recovery)

=== AFTER ===
Items in Trash: 0 (should be 0)

✓ Trash successfully emptied for principal@school.accesswt.com
```

*[INSERT SCREENSHOT: axigen_empty_trash.py output showing BEFORE with 1 item details, ACTION, and AFTER showing 0 items]*

---

### 14.5 Script 5 — Permanent Delete (No Trash, No Recovery)

**Purpose:** Alternative to the two-step move→empty approach. Marks messages deleted and immediately expunges — bypasses Trash entirely. Equivalent to Zimbra's hard delete. Always run with `DRY_RUN = True` first.

```bash
cat > /tmp/axigen_permanent_delete.py << 'EOF'
#!/usr/bin/env python3
"""
Permanent bulk delete — marks \Deleted and immediately expunges.
No Trash intermediary. No recovery possible after DRY_RUN=False.
Run with DRY_RUN=True first to preview matches.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
SUBJECT     = "Confidential"
DRY_RUN     = True   # ← set False only after confirming dry-run

ACCOUNTS = [
    ("admin@school.accesswt.com",        "your_password"),
    ("info@school.accesswt.com",         "your_password"),
    ("principal@school.accesswt.com",    "your_password"),
    ("admin@fitness.accesswt.com",       "your_password"),
    ("info@fitness.accesswt.com",        "your_password"),
    ("membership@fitness.accesswt.com",  "your_password"),
]

def permanent_delete(account, password):
    try:
        mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
        mail.login(account, password)
        mail.select("INBOX" if not DRY_RUN else "INBOX")

        mail.select("INBOX", readonly=DRY_RUN)
        status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
        msg_ids = data[0].split()

        if not msg_ids:
            print(f"  ✓ No match — nothing to delete")
            mail.logout()
            return

        print(f"  Found {len(msg_ids)} message(s)")

        if DRY_RUN:
            for msg_id in msg_ids:
                _, msg_data = mail.fetch(
                    msg_id,
                    "(BODY[HEADER.FIELDS (SUBJECT FROM DATE)])"
                )
                print(f"  [DRY RUN] Would permanently delete:")
                print(f"  {msg_data[0][1].decode().strip()}")
        else:
            mail.select("INBOX")
            for msg_id in msg_ids:
                mail.store(msg_id, "+FLAGS", "\\Deleted")
            mail.expunge()   # gone forever
            print(f"  ✓ Permanently deleted and expunged (no Trash)")

            # Verify
            status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
            remaining = data[0].split()
            print(f"  Verification: {len(remaining)} remaining (should be 0)")

        mail.logout()

    except Exception as e:
        print(f"  ✗ Error: {e}")

# MAIN
mode = "DRY RUN (preview only)" if DRY_RUN else "LIVE — PERMANENT DELETE"
print(f"Axigen Permanent Delete — {mode}")
print(f"Subject : '{SUBJECT}'")
print(f"{'⚠ No recovery possible' if not DRY_RUN else 'ℹ Nothing will be deleted'}")
print("=" * 55)

for account, password in ACCOUNTS:
    print(f"\n[{account}]")
    permanent_delete(account, password)

print("\n" + "=" * 55)
print("Done.")
EOF

# Step 1 — run dry run first
python3 /tmp/axigen_permanent_delete.py

# Step 2 — after confirming correct matches, switch to live
sed -i 's/DRY_RUN     = True/DRY_RUN     = False/' /tmp/axigen_permanent_delete.py
python3 /tmp/axigen_permanent_delete.py
```

*[INSERT SCREENSHOT: axigen_permanent_delete.py DRY RUN output showing messages that would be deleted]*
*[INSERT SCREENSHOT: axigen_permanent_delete.py LIVE output showing "Permanently deleted and expunged" with verification: 0 remaining]*

---

### 14.6 Script 6 — Bulk Push (Send to All Axigen Users)

**Purpose:** Sends a test message from `info@consultancy.accesswt.com` to all Axigen-hosted users via SMTP.

```bash
cat > /tmp/axigen_bulk_push.py << 'EOF'
#!/usr/bin/env python3
"""
Bulk push — send message to all Axigen-hosted users via SMTP.
Sender: info@consultancy.accesswt.com (Zimbra account)
Server: 10.10.10.110 (Zimbra MTA — sender's home server)
"""
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from datetime import datetime

SENDER      = "info@consultancy.accesswt.com"
SMTP_SERVER = "10.10.10.110"   # Zimbra MTA
SMTP_PORT   = 25
SUBJECT     = "Confidential - Bulk Push Test"

RECIPIENTS = [
    "admin@school.accesswt.com",
    "info@school.accesswt.com",
    "principal@school.accesswt.com",
    "admin@fitness.accesswt.com",
    "info@fitness.accesswt.com",
    "membership@fitness.accesswt.com",
]

BODY = f"""\
Dear All,

This is a BULK PUSH test message sent to all Axigen-hosted users.

CONFIDENTIAL: This message is for testing bulk mail operations.
Sent: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

Regards,
Info - Consultancy Team
"""

success, failed = [], []
print(f"Axigen Bulk Push")
print(f"Server  : {SMTP_SERVER}:{SMTP_PORT}")
print(f"Sender  : {SENDER}")
print(f"Subject : {SUBJECT}")
print(f"Recipients: {len(RECIPIENTS)}")
print("=" * 55)

with smtplib.SMTP(SMTP_SERVER, SMTP_PORT, timeout=10) as smtp:
    smtp.ehlo()
    for recipient in RECIPIENTS:
        try:
            msg = MIMEMultipart()
            msg["From"]    = SENDER
            msg["To"]      = recipient
            msg["Subject"] = SUBJECT
            msg.attach(MIMEText(BODY, "plain"))
            smtp.sendmail(SENDER, recipient, msg.as_string())
            print(f"  ✓ {recipient}")
            success.append(recipient)
        except Exception as e:
            print(f"  ✗ {recipient} — {e}")
            failed.append(recipient)

print(f"\n{'='*55}")
print(f"Done: {len(success)} sent, {len(failed)} failed")
EOF

python3 /tmp/axigen_bulk_push.py
```

*[INSERT SCREENSHOT: axigen_bulk_push.py output showing ✓ for all 6 Axigen recipients]*

---

### 14.7 Script 7 — Single Push (Targeted Send)

**Purpose:** Send a single test message to one specific Axigen user using swaks.

```bash
# Using swaks — shows full SMTP conversation
swaks \
  --to admin@school.accesswt.com \
  --from info@consultancy.accesswt.com \
  --server 10.10.10.111 \
  --port 25 \
  --header "Subject: Single Push Test" \
  --body "This is a single targeted test message to admin@school.accesswt.com only." \
  --verbose

# Or using Python for consistency
python3 << 'EOF'
import smtplib
from email.mime.text import MIMEText

msg = MIMEText("This is a single targeted test message.")
msg["Subject"] = "Single Push Test"
msg["From"]    = "info@consultancy.accesswt.com"
msg["To"]      = "admin@school.accesswt.com"

with smtplib.SMTP("10.10.10.111", 25) as smtp:
    smtp.sendmail(msg["From"], msg["To"], msg.as_string())
    print("✓ Single message sent successfully")
EOF
```

*[INSERT SCREENSHOT: swaks --verbose output showing full SMTP handshake with 250 OK confirmation]*

---

### 14.8 Script 8 — Single Delete (One Message, One User)

**Purpose:** Delete one specific message from one specific user's INBOX. Shows message details before deleting.

```bash
cat > /tmp/axigen_single_delete.py << 'EOF'
#!/usr/bin/env python3
"""
Delete a single specific message from one Axigen user's INBOX.
Shows message details before deletion, verifies after.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "admin@school.accesswt.com"
PASSWORD    = "your_password"
SUBJECT     = "Single Push Test"

print(f"{'='*55}")
print(f"Account : {ACCOUNT}")
print(f"Subject : '{SUBJECT}'")
print(f"{'='*55}")

mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
mail.login(ACCOUNT, PASSWORD)
print(f"✓ Login successful\n")

mail.select("INBOX", readonly=True)
status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
msg_ids = data[0].split()

print(f"=== BEFORE DELETE ===")
print(f"Messages matching '{SUBJECT}': {len(msg_ids)}")

if not msg_ids:
    print(f"\n✗ No messages found — nothing to delete")
    mail.logout()
    exit()

# Show message details before deleting
for msg_id in msg_ids:
    _, msg_data = mail.fetch(
        msg_id,
        "(BODY[HEADER.FIELDS (SUBJECT FROM DATE)])"
    )
    print(f"\n--- Will delete ID {msg_id.decode()} ---")
    print(msg_data[0][1].decode().strip())

# ACTION — delete
print(f"\n=== ACTION: Deleting ===")
mail.select("INBOX")
for msg_id in msg_ids:
    mail.store(msg_id, "+FLAGS", "\\Deleted")
mail.expunge()
print(f"✓ Message(s) deleted and expunged")

# AFTER — verify
mail.select("INBOX", readonly=True)
status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
remaining = data[0].split()
print(f"\n=== AFTER DELETE ===")
print(f"Messages remaining: {len(remaining)} (should be 0)")

mail.logout()
print(f"\n✓ Single delete complete for {ACCOUNT}")
EOF

python3 /tmp/axigen_single_delete.py
```

*[INSERT SCREENSHOT: axigen_single_delete.py output showing BEFORE with message details, ACTION, and AFTER with 0 remaining]*

---

### 14.9 Script 9 — Test Push (SMTP Verification)

**Purpose:** Send a test message and display the complete SMTP conversation for verification.

```bash
# Method 1 — swaks verbose (recommended for SMTP verification)
swaks \
  --to info@fitness.accesswt.com \
  --from test@accesswt.com \
  --server 10.10.10.111 \
  --port 25 \
  --header "Subject: Test Push Verification" \
  --body "This is a test push to verify SMTP delivery on Axigen." \
  --verbose

# Expected swaks output:
# -> EHLO ...
# <- 250-axigen.accesswt.com Axigen ESMTP hello
# -> MAIL FROM:<test@accesswt.com>
# <- 250 Sender accepted
# -> RCPT TO:<info@fitness.accesswt.com>
# <- 250 Recipient accepted
# -> DATA
# <- 354 Ready to receive data
# -> [message content]
# -> .
# <- 250 Mail queued for delivery  ← SUCCESS
# -> QUIT
# <- 221 Good bye
```

*[INSERT SCREENSHOT: swaks --verbose output showing complete SMTP conversation with 250 OK at DATA step]*

---

### 14.10 Script 10 — Test Delete (IMAP Verification)

**Purpose:** Full verified delete cycle for a single test message — confirms existence, shows details, deletes, verifies deletion.

```bash
cat > /tmp/axigen_test_delete.py << 'EOF'
#!/usr/bin/env python3
"""
Full verified delete cycle for one test message.
Shows every step with before/after confirmation.
"""
import imaplib

IMAP_SERVER = "10.10.10.111"
IMAP_PORT   = 993
ACCOUNT     = "info@fitness.accesswt.com"
PASSWORD    = "your_password"
SUBJECT     = "Test Push Verification"

print(f"Axigen Test Delete — Full Verification Cycle")
print(f"{'='*55}")
print(f"Account : {ACCOUNT}")
print(f"Subject : '{SUBJECT}'")
print(f"Server  : {IMAP_SERVER}:{IMAP_PORT}")
print(f"{'='*55}\n")

mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
mail.login(ACCOUNT, PASSWORD)
print(f"✓ Step 1: Login successful")

mail.select("INBOX", readonly=True)
status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
msg_ids = data[0].split()

print(f"\n✓ Step 2: INBOX search complete")
print(f"  Found: {len(msg_ids)} message(s) matching '{SUBJECT}'")

if not msg_ids:
    print(f"\n✗ No messages found — run test push first")
    mail.logout()
    exit()

# Show message details
print(f"\n✓ Step 3: Message details")
for msg_id in msg_ids:
    _, msg_data = mail.fetch(
        msg_id,
        "(BODY[HEADER.FIELDS (SUBJECT FROM DATE)])"
    )
    print(f"  {msg_data[0][1].decode().strip()}")

# Delete
print(f"\n✓ Step 4: Deleting...")
mail.select("INBOX")
for msg_id in msg_ids:
    mail.store(msg_id, "+FLAGS", "\\Deleted")
mail.expunge()
print(f"  \\Deleted flag set + expunged")

# Verify
mail.select("INBOX", readonly=True)
status, data = mail.search(None, f'SUBJECT "{SUBJECT}"')
remaining = data[0].split()
print(f"\n✓ Step 5: Verification")
print(f"  Messages remaining: {len(remaining)}")

if len(remaining) == 0:
    print(f"\n✅ TEST DELETE SUCCESS — message completely removed from {ACCOUNT}")
else:
    print(f"\n⚠ WARNING — {len(remaining)} message(s) still present")

mail.logout()
EOF

python3 /tmp/axigen_test_delete.py
```

*[INSERT SCREENSHOT: axigen_test_delete.py output showing all 5 steps with final "TEST DELETE SUCCESS" confirmation]*

---

### 14.11 Axigen Operations Quick Reference

| Script | Purpose | Key Setting |
|---|---|---|
| `axigen_check_any_mail.py` | List all folders + counts | Change `ACCOUNT` |
| `axigen_check_specific_mail.py` | Search by subject | Change `ACCOUNT`, `SUBJECT` |
| `axigen_move_to_trash.py` | Bulk move to Trash | Edit `ACCOUNTS` list + passwords |
| `axigen_empty_trash.py` | Empty Trash for one user | Change `ACCOUNT` |
| `axigen_permanent_delete.py` | No-recovery bulk delete | Set `DRY_RUN=False` after preview |
| `axigen_bulk_push.py` | Send to all Axigen users | Edit `RECIPIENTS`, `SUBJECT` |
| `axigen_single_delete.py` | Delete one message, one user | Change `ACCOUNT`, `SUBJECT` |
| `axigen_test_delete.py` | Full verified delete cycle | Change `ACCOUNT`, `SUBJECT` |

---

## 16. Bulk Operations — Full Scenario Walkthrough

### 15.1 Scenario: Accidental Confidential Mass Mail — Complete Resolution

**What happened:**
`info@consultancy.accesswt.com` (Zimbra) accidentally sent a message with subject "Confidential" to all users across both Zimbra and Axigen mail servers. As a high-privilege admin, the task is to delete this message from every mailbox on both servers — as if it never existed.

**Affected mailboxes:**
- Zimbra: 22 accounts (consultancy, microfinance, dairy)
- Axigen: 6 accounts (school, fitness)
- Total: 28 mailboxes

### 15.2 Step-by-Step Resolution

**Phase 1 — Confirm the mail exists (verify before acting)**

```bash
# On Zimbra — check one recipient
su - zimbra
zmmailbox -z -m info@microfinance.accesswt.com search "in:inbox subject:Confidential"
# Result: num: 1, more: false  ← confirmed exists
```

```bash
# On Axigen — check one recipient
python3 /tmp/axigen_check_specific_mail.py
# Result: ✓ FOUND — mail EXISTS in principal@school.accesswt.com
```

*[INSERT SCREENSHOT: zmmailbox search result showing num:1 before deletion]*
*[INSERT SCREENSHOT: axigen_check_specific_mail.py showing FOUND with message headers]*

**Phase 2 — Delete from Zimbra (bulk)**

```bash
chmod +x /tmp/bulk_delete_zimbra.sh
/tmp/bulk_delete_zimbra.sh
# Result: Found conversation ID -257 in each account → deleted
```

**Phase 3 — Empty Zimbra Trash (bulk)**

```bash
for ACCOUNT in $(zmprov -l gaa); do
  zmmailbox -z -m "$ACCOUNT" emptyFolder /Trash >/dev/null 2>&1
done
echo "All Trash folders successfully purged!"
```

**Phase 4 — Verify Zimbra deletion**

```bash
zmmailbox -z -m info@microfinance.accesswt.com search "in:inbox subject:Confidential"
# Result: num: 0, more: false  ← CONFIRMED DELETED
```

*[INSERT SCREENSHOT: zmmailbox search after deletion showing num:0]*

**Phase 5 — Delete from Axigen (bulk, two-step)**

```bash
python3 /tmp/axigen_move_to_trash.py
# Result: 6 messages moved to Trash across 6 mailboxes

python3 /tmp/axigen_empty_trash.py
# Run for each account, or loop through all
```

**Phase 6 — Final verification across both servers**

```bash
# Zimbra
zmmailbox -z -m user1@consultancy.accesswt.com search "in:inbox subject:Confidential"
# num: 0, more: false ✓

# Axigen
python3 /tmp/axigen_check_specific_mail.py
# ✗ NOT FOUND — no mail with subject 'Confidential' in INBOX ✓
```

*[INSERT SCREENSHOT: Final verification on Zimbra showing num:0]*
*[INSERT SCREENSHOT: Final verification on Axigen showing NOT FOUND]*

---

## 17. Security Hardening Summary

### 17.1 What Was Applied

| Component | Hardening Applied |
|---|---|
| Proxmox NAT | WebAdmin (9443) restricted to internal IPs only via DNAT targeting |
| Axigen | ufw firewall with specific port allowances |
| Axigen | fail2ban installed for brute-force protection |
| Axigen | Root SSH login disabled |
| Axigen | Greylisting enabled and confirmed working |
| Axigen | Self-signed TLS certificate (CN=axigen.accesswt.com, 2-year) |
| Zimbra | SELinux set to permissive (required for Zimbra operation) |
| Zimbra | Built-in Amavis antispam + ClamAV antivirus running |
| Zimbra | OpenDKIM running for outbound signing |
| DNS | DNSSEC validation enabled |
| DNS | Zone transfers restricted to `transfer-key` HMAC-SHA256 |
| DNS | DDNS updates restricted to `ddns-key` only |
| All | SPF records configured for each client domain |
| All | DMARC records with `p=quarantine` for each client domain |

### 17.2 What Still Needs Doing (Production Checklist)

```
[ ] Replace self-signed certs with Let's Encrypt (once domain is public)
[ ] Set DMARC to p=reject (currently p=quarantine)
[ ] Change SPF ~all to -all (hard fail) after confirming all senders
[ ] Add DKIM TXT records for each client domain in DNS
[ ] Set account lockout policy in Axigen (brute-force protection)
[ ] Configure per-domain quotas in Axigen and Zimbra
[ ] Set up regular backup cron jobs for both mail servers
[ ] Move WebAdmin (9443) behind VPN — remove from DNAT entirely
[ ] Review and close unused ports (21/FTP on Axigen if not using)
[ ] Enable audit logging for all admin actions
```

---

## 18. Environment Summary — Final State

### 18.1 All Running Services — Verified

**Zimbra (10.10.10.110):**
```
zmcontrol status output — all 18 services: Running
amavis, antispam, antivirus, ldap, logger, mailbox,
memcached, mta, opendkim, proxy, service webapp,
snmp, spell, stats, zimbra webapp, zimbraAdmin webapp,
zimlet webapp, zmconfigd
```

**Axigen (10.10.10.111):**
```
Services Management — running:
Queue Processing, SMTP Receiving, SMTP Sending,
IMAP, WebMail, WebAdmin, CLI, Log,
FTP Backup & Restore, Axigen TNEF Decoder
```

**DNS (ns1 10.10.10.106):**
```
BIND9 running — authoritative for:
accesswt.com (master)
10.10.10.in-addr.arpa (master)
Zone serial: 2026063002
```

### 18.2 All Accounts — Final State

**Zimbra domains and accounts:**
```
zimbra.accesswt.com      → admin + testuser + system accounts
consultancy.accesswt.com → admin, info, user1-8 (10 accounts)
microfinance.accesswt.com → admin, info, user1-8 (10 accounts)
dairy.accesswt.com       → admin, info (2 accounts)
Total: ~28 user accounts
```

**Axigen domains and accounts:**
```
accesswt.com             → postmaster (primary domain)
school.accesswt.com      → admin, info, principal (3 accounts)
fitness.accesswt.com     → admin, info, membership (3 accounts)
hospitalname.com.np      → created but DNS/delivery pending public IP
Total: 7+ user accounts
```

*[INSERT SCREENSHOT: Zimbra Admin Console showing all 4 domains in Configure → Domains]*
*[INSERT SCREENSHOT: Axigen Manage Accounts page showing all domains in left panel with account counts]*

---
*Textbook Version: 1.0*
*Document generated: July 01, 2026*
*Environment: accesswt.com internal lab*
*Proxmox PVE 9.2.2 | Zimbra 10.1.16 GA (Rocky Linux 9.8) | Axigen 10.6.35 (Ubuntu 24.04)*
*Total pages: ~100 equivalent | Total sections: 18*
