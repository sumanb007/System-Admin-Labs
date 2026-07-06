
# Mail Server Setup Locally on PC
## Zimbra & Axigen Mail Server Deployment on accesswt.com
### Complete Setup, Configuration, Verification & Operations Guide

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Minimum Required Resources](#2-minimum-required-resources)
3. [Network & NAT Configuration (Proxmox Host)](#3-network--nat-configuration-proxmox-host)
4. [Update DNS of mail servers](#4-update-dns-of-mail-servers)
5. [Axigen Mail Server Setup (Ubuntu 24)](#5-axigen-mail-server-setup-ubuntu-24)
6. [Zimbra Mail Server Setup (Rocky Linux 9)](#6-zimbra-mail-server-setup-rocky-linux-9)
7. [SSL/TLS Configuration](#7-ssltls-configuration)
8. [Axigen Hardening](#8-axigen-hardening)
9. [Multi-Tenant Client Provisioning](#9-multi-tenant-client-provisioning)
10. [Mail Flow — Detailed Chain Sequence](#10-mail-flow--detailed-chain-sequence)
11. [Test Cases — All Tried Scenarios](#11-test-cases--all-tried-scenarios)
12. [Bulk Mail Operations (Push & Delete)](#12-bulk-mail-operations-push--delete)
13. [Monitoring Commands](#13-monitoring-commands)

---

## 1. Infrastructure Overview

<img width="600" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/lab_mail_architecture.png">

```
Internet
    │
    ▼
ISP Router (wlo1 DHCP: 192.168.1.0/27 or 172.18.0.0/23)
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

## 2. Minimum Required Resources

### Zimbra VM (zimbra.accesswt.com)
| Resource | Minimum | Recommended | Actual Used |
|---|---|---|---|
| vCPU | 2 cores | 4 cores | 4 cores |
| RAM | 4 GB | 8 GB | 3.6 GB + 2 GB swap |
| Storage | 40 GB | 100 GB | 17 GB root |
| OS | RHEL 8/9, Rocky 9 | Rocky Linux 9.8 | Rocky Linux 9.8 |
| Zimbra Version | 10.x | 10.1.16/10.1.15 | 10.1.16/10.1.15 GA |

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
- `wlo1` — WiFi uplink to LAN (192.168.1.0/27,172.18.0./23), provides internet access
- `vmbr1` — Internal bridge (10.10.10.1/24), connects all VMs
- NAT MASQUERADE — allows VMs to reach the internet through wlo1
- DNAT rules — port-forward external access into specific VMs

### 3.2 Complete interfaces file

```bash
auto lo
iface lo inet loopback

auto nic1
iface nic1 inet manual

iface nic0 inet manual

auto wlo1
iface wlo1 inet dhcp
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
        metric 100

auto vmbr0
iface vmbr0 inet manual
        bridge-ports nic1
        bridge-stp off
        bridge-fd 0
        metric 10

auto vmbr1
iface vmbr1 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o wlo1 -j MASQUERADE
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 3389 -j DNAT --to-destination 10.10.10.201:3389
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2222 -j DNAT --to-destination 10.10.10.202:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2102 -j DNAT --to-destination 10.10.10.102:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2101 -j DNAT --to-destination 10.10.10.101:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2225 -j DNAT --to-destination 10.10.10.125:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2226 -j DNAT --to-destination 10.10.10.126:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2106 -j DNAT --to-destination 10.10.10.106:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2107 -j DNAT --to-destination 10.10.10.107:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2108 -j DNAT --to-destination 10.10.10.108:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2109 -j DNAT --to-destination 10.10.10.109:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2110 -j DNAT --to-destination 10.10.10.110:22
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 2111 -j DNAT --to-destination 10.10.10.111:22


#Axigen webadmin HTTP
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9000 -j DNAT --to-destination 10.10.10.111:9000
#Axigen webadmin HTTPS
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 9443 -j DNAT --to-destination 10.10.10.111:9443
#Axigen Webmail HTTPS
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 443 -j DNAT --to-destination 10.10.10.111:443
#Axigen Webmail SMTP
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 25 -j DNAT --to-destination 10.10.10.111:25
#Axigen SMTP Sbumission
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 587 -j DNAT --to-destination 10.10.10.111:587
#Axigen IMAPS (secure IMAP Mail Retrival)
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 993 -j DNAT --to-destination 10.10.10.111:993

#Zimbra Webmail HTTPS
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 8443 -j DNAT --to-destination 10.10.10.110:8443
#Zimbra Admin Console
        post-up iptables -t nat -A PREROUTING -i wlo1 -p tcp --dport 7071 -j DNAT --to-destination 10.10.10.110:7071
source /etc/network/interfaces.d/*
root@awtserver:~# 
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

## 4. Update DNS of mail servers

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

Since the zone has `update-policy` (DDNS), you cannot use `rndc reload` directly. Use freeze/thaw:

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
# Thank you for installing AXIGEN Mail Server
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
Password: (set your password — minimum 6 characters)
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

### 10.1 Complete Mail Flow: Internal Sender to Internal Receiver

**Scenario:** `info@consultancy.accesswt.com` (Zimbra) sends to `admin@school.accesswt.com` (Axigen)

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1 — User composes mail in browser                          │
│                                                                 │
│  Browser (192.168.1.x MacBook)                                  │
│  URL: https://192.168.1.3:8443/zimbra                          │
│  Action: Compose → To: admin@school.accesswt.com → Send        │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS POST (port 8443)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2 — Proxmox NAT (awtserver)                                │
│                                                                 │
│  Incoming: 192.168.1.x:random → 192.168.1.3:8443              │
│  DNAT rule: --dport 8443 → 10.10.10.110:8443                  │
│  Translated: 192.168.1.x → 10.10.10.110:8443                  │
│  Firewall: MASQUERADE (source NAT for return traffic)           │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS (port 8443)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3 — Zimbra Proxy (nginx) receives HTTPS request            │
│                                                                 │
│  10.10.10.110:8443                                             │
│  nginx proxy → Zimbra mailbox webapp (jetty, port 8080 internal)│
│  Session authenticated → mail submitted to local MTA queue      │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Internal LMTP/Postfix handoff
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 4 — Zimbra MTA (Postfix) processes outgoing message        │
│                                                                 │
│  From: info@consultancy.accesswt.com                           │
│  To:   admin@school.accesswt.com                               │
│                                                                 │
│  Postfix checks: is school.accesswt.com local? NO              │
│  → Performs DNS MX lookup for school.accesswt.com              │
└──────────────────────────────┬──────────────────────────────────┘
                               │ DNS query (UDP port 53)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 5 — DNS Resolution                                         │
│                                                                 │
│  Query → ns3/ns4 (10.10.10.108/109) resolver                  │
│  ns3/ns4 → forwards to ns1 (10.10.10.106) authoritative       │
│                                                                 │
│  MX query: school.accesswt.com → axigen.accesswt.com (prio 10) │
│  A query:  axigen.accesswt.com → 10.10.10.111                 │
│                                                                 │
│  Answer returned to Zimbra Postfix                              │
└──────────────────────────────┬──────────────────────────────────┘
                               │ SMTP (port 25)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 6 — Zimbra MTA delivers to Axigen SMTP                     │
│                                                                 │
│  Zimbra Postfix connects: 10.10.10.111:25                      │
│  EHLO zimbra.accesswt.com                                      │
│  STARTTLS negotiated (TLS 1.2+)                                │
│  MAIL FROM: <info@consultancy.accesswt.com>                    │
│  RCPT TO: <admin@school.accesswt.com>                          │
│  DATA → message body transmitted                                │
│  250 Mail queued for delivery                                   │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Axigen internal delivery
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 7 — Axigen receives and delivers to local mailbox          │
│                                                                 │
│  Axigen SMTP Receiving service                                  │
│  Checks: is admin@school.accesswt.com a local account? YES     │
│  Checks: SPF pass (ip4:10.10.10.110 allowed)                   │
│  Checks: Greylisting (first attempt → 451 temp reject)         │
│          (retry → 250 accepted)                                 │
│  Delivers to: /var/opt/axigen/domains/school.accesswt.com/     │
│               accounts/admin/INBOX/                             │
└──────────────────────────────┬──────────────────────────────────┘
                               │ User opens webmail
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 8 — Recipient reads mail in browser                        │
│                                                                 │
│  Browser: https://192.168.1.3:443/webmail                      │
│  DNAT: 192.168.1.3:443 → 10.10.10.111:443                     │
│  Axigen Webmail HTTPS → authenticates admin@school.accesswt.com│
│  Renders INBOX → message "Confidential" visible                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Port Reference Table

| Port | Protocol | Service | Direction |
|---|---|---|---|
| 25 | SMTP | Mail transfer between servers | Inbound/Outbound |
| 587 | SMTP Submission | Client authenticated send | Outbound |
| 465 | SMTPS | Implicit TLS mail submit | Outbound |
| 993 | IMAPS | Mail retrieval (SSL) | Inbound client |
| 995 | POP3S | Mail retrieval (SSL) | Inbound client |
| 443 | HTTPS | Axigen Webmail | Inbound client |
| 8443 | HTTPS | Zimbra Webmail | Inbound client |
| 7071 | HTTPS | Zimbra Admin Console | Admin only |
| 9443 | HTTPS | Axigen WebAdmin | Admin only |
| 53 | DNS UDP/TCP | Name resolution | Internal |

### 10.3 DNS Resolution Chain

```
Mail Client
    │
    ├─► asks: what is MX for school.accesswt.com?
    │
    ▼
ns3/ns4 (10.10.10.108/109) — Resolvers
    │
    ├─► forwards authoritative query to ns1 (10.10.10.106)
    │
    ▼
ns1 (10.10.10.106) — Authoritative Master
    │
    ├─► answers: MX 10 axigen.accesswt.com
    ├─► answers: A  axigen.accesswt.com = 10.10.10.111
    │
    ▼
Answer returned: deliver to 10.10.10.111:25
```

---

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

---

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

## 14. Axigen Mail Operations — Complete Script Reference

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

## 15. Bulk Operations — Full Scenario Walkthrough

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

## 16. Troubleshooting Reference

### 16.1 Common Issues and Fixes

| Issue | Symptom | Fix |
|---|---|---|
| Zimbra install interrupted | SSH drops during `Starting servers...` | Use `screen -S zimbra`; reconnect with `screen -r zimbra` |
| Zimbra CPU exhaustion | Install freezes, load >8 | Add swap (`fallocate -l 4G /swapfile`), bump to 4 vCPU |
| `rndc reload` fails on dynamic zone | `'reload' failed: dynamic zone` | Use `rndc freeze` → `reload` → `thaw` or delete `.jnl` file |
| Axigen SMTP 421 timeout | Connection drops before EHLO | Type commands fast; use `nc` heredoc or swaks instead |
| Mail body missing in webmail | Email shows subject only | Missing blank line between RFC822 headers and body text |
| `mail` command fails | `Cannot open mailer` | Use `swaks` or Python smtplib — no local MTA on Axigen |
| Axigen WebAdmin not reachable from LAN | `curl: (7) Failed to connect` | DNAT rules not applied — run `iptables -t nat -A PREROUTING...` manually |
| `sudo cd` fails | `sudo: cd: command not found` | `cd` is a shell built-in — use `sudo -i` then `cd`, or `sudo ls /path` |
| Zimbra `zmprov gaa` fails | `getAllAccounts can only be used with -l` | Use `zmprov -l gaa` (LDAP mode) |
| Axigen greylisting rejection | `451 Temporary rejected` | Normal behavior — retry once; greylisting passes on second attempt |

### 16.2 Zimbra Service Recovery

```bash
su - zimbra

# Full restart
zmcontrol stop
zmcontrol start
zmcontrol status

# Restart individual service
zmcontrol restart mailbox
zmcontrol restart mta
zmcontrol restart proxy

# Check for stuck mail queue
postqueue -p       # list queue
postqueue -f       # force flush
postsuper -d ALL deferred   # delete all deferred (use carefully)

# Check logs for errors
sudo grep -i "error\|fatal\|panic" /var/log/zimbra.log | tail -20
```

### 16.3 Axigen Service Recovery

```bash
# Restart Axigen
sudo systemctl stop axigen
sudo systemctl start axigen
sudo systemctl status axigen

# Check if process is actually running
ps aux | grep axigen | grep -v grep

# If ports not listening after restart
sudo ss -tlnp | grep axigen

# Check config integrity
sudo ls -la /var/opt/axigen/run/

# Full log check
sudo find /var/log/axigen -name "*.log" -exec tail -5 {} \; 2>/dev/null
```

### 16.4 DNS Troubleshooting

```bash
# Check if BIND is running
sudo systemctl status bind9

# Test specific resolution
dig @10.10.10.106 school.accesswt.com MX +short
dig @10.10.10.106 school.accesswt.com A +short

# Check zone loaded correctly
sudo rndc status | grep -i zone
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com

# Force zone reload (dynamic zone safe method)
sudo rndc freeze accesswt.com && sudo rndc reload accesswt.com && sudo rndc thaw accesswt.com

# Check journal conflict
sudo ls -la /etc/bind/zones/*.jnl
# If stale journal causing issues:
sudo systemctl stop bind9
sudo rm /etc/bind/zones/db.accesswt.com.jnl
sudo systemctl start bind9
```

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
*Total pages: ~80 equivalent | Total sections: 18*
