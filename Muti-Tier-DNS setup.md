# Multi-Tier DNS Architecture — Complete Setup Guide
### accesswt.com | BIND9 | 4-Server Implementation

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Server Summary](#2-server-summary)
3. [File Paths — What They Are and Why They Exist](#3-file-paths--what-they-are-and-why-they-exist)
4. [NS1 — Primary Master (10.10.10.106)](#4-ns1--primary-master-101010106)
5. [NS2 — Secondary Master (10.10.10.107)](#5-ns2--secondary-master-101010107)
6. [NS3 — Resolver (10.10.10.108)](#6-ns3--resolver-101010108)
7. [NS4 — Resolver (10.10.10.109)](#7-ns4--resolver-101010109)
8. [Client VM Configuration — Which NS to Add](#8-client-vm-configuration--which-ns-to-add)
9. [Adding DNS Records — Static vs Dynamic](#9-adding-dns-records--static-vs-dynamic)
10. [Dynamic DNS Update via nsupdate](#10-dynamic-dns-update-via-nsupdate)
11. [Adding DNS from a Resolver — Can We? How to Verify?](#11-adding-dns-from-a-resolver--can-we-how-to-verify)
12. [How Dynamic DNS Hits the Same File as Static Edits](#12-how-dynamic-dns-hits-the-same-file-as-static-edits)
13. [Adding and Removing Records](#13-adding-and-removing-records)
14. [Whitelisting and Blacklisting DNS](#14-whitelisting-and-blacklisting-dns)
15. [Full Verification and High Availability Tests](#15-full-verification-and-high-availability-tests)
16. [Zone Update Workflow](#16-zone-update-workflow)
17. [Troubleshooting Reference](#17-troubleshooting-reference)
18. [Quick Reference Summary](#18-quick-reference-summary)

---

## 1. Architecture Overview

```
                    ┌────────────────────────────────────────────┐
                    │         TIER 1 — AUTHORITATIVE             │
                    │                                            │
                    │  NS1 (Primary Master)                      │
                    │  10.10.10.106                              │
                    │  type: master                              │
                    │  recursion: no                             │
                    │  Generates TSIG keys                       │
                    │  Zone files: /etc/bind/zones/              │
                    │         │                                  │
                    │         │  AXFR + TSIG (transfer-key)      │
                    │         ▼                                  │
                    │  NS2 (Secondary Master)                    │
                    │  10.10.10.107                              │
                    │  type: slave ← pulls from NS1              │
                    │  recursion: no                             │
                    │  Cache: /var/cache/bind/                   │
                    └──────────────┬─────────────────────────────┘
                                   │
                    NOTIFY / Forward queries
                                   │
                    ┌──────────────▼─────────────────────────────┐
                    │         TIER 2 — RESOLVERS                 │
                    │                                            │
                    │  NS3                NS4                    │
                    │  10.10.10.108       10.10.10.109           │
                    │  type: forward      type: forward          │
                    │  recursion: yes     recursion: yes         │
                    │  dnssec-val: no     dnssec-val: no         │
                    │  forwarders → NS1, NS2                     │
                    └──────────────┬─────────────────────────────┘
                                   │
                    ┌──────────────▼─────────────────────────────┐
                    │    CLIENT VMs — 10.10.10.0/24              │
                    │    resolv.conf:                            │
                    │      nameserver 10.10.10.108               │
                    │      nameserver 10.10.10.109               │
                    └────────────────────────────────────────────┘
```

### Key Design Decisions

- NS1 and NS2 set `recursion no` — strictly authoritative, never serve client queries directly.
- NS3 and NS4 set `recursion yes` — serve client queries, forward `accesswt.com` upstream.
- Zone transfers from NS1 → NS2 are secured with TSIG (HMAC-SHA256).
- NS3 and NS4 use `dnssec-validation no` because `accesswt.com` is an internal unsigned zone.
- NS2 re-notifies NS4 after receiving its own transfer from NS1.
- Clients point only to NS3 and NS4 — never to NS1 or NS2.

---

## 2. Server Summary

| Server | IP | Role | Zone Type | Clients |
|--------|----|------|-----------|---------|
| ns1.accesswt.com | 10.10.10.106 | Primary Master | master | Internal only |
| ns2.accesswt.com | 10.10.10.107 | Secondary Master | slave | Internal only |
| ns3.accesswt.com | 10.10.10.108 | Resolver | forward | 10.10.10.0/24 |
| ns4.accesswt.com | 10.10.10.109 | Resolver | forward | 10.10.10.0/24 |

---

## 3. File Paths — What They Are and Why They Exist

*🔴 Screenshot: Show the directory listing `ls -la /etc/bind/` and `ls -la /etc/bind/zones/` on NS1 after setup is complete.*

### On NS1 and NS2 (Authoritative Servers)

| File Path | What It Is | Why We Need It |
|-----------|------------|----------------|
| `/etc/bind/named.conf` | Main BIND entry point | Includes all other config files via `include` directives. BIND reads this first on startup. |
| `/etc/bind/named.conf.options` | Global server behavior | Controls how BIND behaves: what IP it listens on, recursion setting, DNSSEC mode, cache location. Think of it as "how should this server act?" |
| `/etc/bind/named.conf.local` | Zone definitions + TSIG keys | Defines which zones this server is responsible for, whether master or slave, where zone files live, and who can transfer them. Think of it as "what domains does this server know about?" |
| `/etc/bind/named.conf.default-zones` | Built-in BIND zones | Defines standard zones: root hint (`.`), localhost, loopback reverse. These come pre-installed. Do not modify. |
| `/etc/bind/zones/db.accesswt.com` | Forward zone file | Contains all DNS records for `accesswt.com`: A, NS, MX, TXT, CNAME records. This is the actual DNS data. Only exists on NS1 (master). NS2 downloads a copy to `/var/cache/bind/`. |
| `/etc/bind/zones/db.10.10.10` | Reverse zone file | Contains PTR records mapping IPs back to hostnames. Enables reverse DNS lookups (`dig -x`). Only on NS1 as master. |
| `/etc/bind/rndc.key` | Administrative authentication key | Used by the `rndc` tool to send control commands to BIND (reload, flush, status). Auto-generated at install. |
| `/var/cache/bind/db.accesswt.com` | Slave zone cache (NS2) | Auto-downloaded from NS1 via AXFR. BIND writes this automatically — never manually create it on NS2. |
| `/var/log/named/` or `journalctl -u named` | BIND logs | Records zone transfers, query errors, DNSSEC failures, startup messages. Critical for troubleshooting. |

### On NS3 and NS4 (Resolvers)

| File Path | What It Is | Why We Need It |
|-----------|------------|----------------|
| `/etc/bind/named.conf.options` | Resolver behavior | Sets `recursion yes`, `allow-recursion`, trusted client ACL, `dnssec-validation no`, listen IP. |
| `/etc/bind/named.conf.local` | TSIG key + forward zones | Contains the shared TSIG key and forward zone blocks that point `accesswt.com` queries at NS1/NS2. |
| `/etc/resolv.conf` (on clients) | Client DNS config | Tells the OS which nameserver to query. Points to NS3 and NS4. |

### Why `/etc/bind/zones/` vs `/var/cache/bind/`

- `/etc/bind/zones/` — manually maintained zone files. Only on NS1 (master). You edit these directly when adding records.
- `/var/cache/bind/` — auto-managed by BIND. Slave servers write downloaded zone copies here. Never edit these manually; the next zone transfer will overwrite your changes.

---

## 4. NS1 — Primary Master (10.10.10.106)

### Install

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc -y
```

### Step 1: Generate TSIG Keys (Only on NS1)

```bash
sudo tsig-keygen -a HMAC-SHA256 transfer-key
sudo tsig-keygen ddns-key
```

> Copy both secret strings. They must be pasted identically into NS2, NS3, and NS4.

*🔴 Screenshot: Output of both tsig-keygen commands showing the key blocks with secret strings.*

### Step 2: Create Directories

```bash
sudo mkdir -p /etc/bind/zones
sudo mkdir -p /etc/bind/keys
```

### Step 3: `/etc/bind/named.conf.options`

```bash
sudo tee /etc/bind/named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";
    recursion no;
    allow-query { 10.10.10.0/24; };
    listen-on { 10.10.10.106; };
    dnssec-validation auto;
    listen-on-v6 { none; };
};
EOF
```

### Step 4: `/etc/bind/named.conf.local`

```bash
sudo tee /etc/bind/named.conf.local << 'EOF'
key "transfer-key" {
    algorithm hmac-sha256;
    secret "YOUR_TRANSFER_KEY_SECRET_HERE";
};

key "ddns-key" {
    algorithm hmac-sha256;
    secret "YOUR_DDNS_KEY_SECRET_HERE";
};

zone "accesswt.com" {
    type master;
    file "/etc/bind/zones/db.accesswt.com";
    allow-transfer { key "transfer-key"; };
    update-policy {
        grant ddns-key zonesub ANY;
    };
    also-notify { 10.10.10.107; 10.10.10.108; };
    notify yes;
};

zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.10.10";
    allow-transfer { key "transfer-key"; };
    update-policy {
        grant ddns-key zonesub ANY;
    };
    also-notify { 10.10.10.107; 10.10.10.108; };
    notify yes;
};
EOF
```

### Step 5: AppArmor Permissions

```bash
sudo tee -a /etc/apparmor.d/local/usr.sbin.named > /dev/null << 'EOF'
/etc/bind/zones/ r,
/etc/bind/zones/** rw,
EOF

sudo systemctl reload apparmor
sudo chown -R bind:bind /etc/bind/zones
sudo chmod -R 775 /etc/bind/zones
```

### Step 6: Forward Zone File `/etc/bind/zones/db.accesswt.com`

```bash
sudo tee /etc/bind/zones/db.accesswt.com << 'EOF'
$TTL 604800
@ IN SOA ns1.accesswt.com. admin.accesswt.com. (
        2024010101 ; Serial — increment on every change
        604800
        86400
        2419200
        604800 )

@       IN NS  ns1.accesswt.com.
@       IN NS  ns2.accesswt.com.
@       IN NS  ns3.accesswt.com.
@       IN NS  ns4.accesswt.com.

ns1     IN A   10.10.10.106
ns2     IN A   10.10.10.107
ns3     IN A   10.10.10.108
ns4     IN A   10.10.10.109

mail1   IN A   10.10.10.180
mail2   IN A   10.10.10.181

@       IN MX 10 mail1.accesswt.com.
@       IN MX 20 mail2.accesswt.com.

@       IN TXT "v=spf1 mx ip4:10.10.10.180 ip4:10.10.10.181 ~all"
_dmarc  IN TXT "v=DMARC1; p=quarantine"
EOF
```

### Step 7: Reverse Zone File `/etc/bind/zones/db.10.10.10`

```bash
sudo tee /etc/bind/zones/db.10.10.10 << 'EOF'
$TTL 604800
@ IN SOA ns1.accesswt.com. admin.accesswt.com. (
        2024010101 ; Must match forward zone serial
        604800
        86400
        2419200
        604800 )

@ IN NS ns1.accesswt.com.
@ IN NS ns2.accesswt.com.
@ IN NS ns3.accesswt.com.
@ IN NS ns4.accesswt.com.

106     IN PTR ns1.accesswt.com.
107     IN PTR ns2.accesswt.com.
108     IN PTR ns3.accesswt.com.
109     IN PTR ns4.accesswt.com.

180     IN PTR mail1.accesswt.com.
181     IN PTR mail2.accesswt.com.
EOF
```

### Step 8: Verify and Start

```bash
sudo chown -R bind:bind /etc/bind/zones

# Verify syntax — expected: no output
sudo named-checkconf

# Verify forward zone — expected: OK
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com

# Verify reverse zone — expected: OK
sudo named-checkzone 10.10.10.in-addr.arpa /etc/bind/zones/db.10.10.10

sudo systemctl restart bind9
sudo systemctl enable named
```

*🔴 Screenshot: named-checkzone outputs showing "OK" for both zones. Also show `sudo systemctl status bind9` showing active (running).*

### Step 9: Verify NS1 is Authoritative

```bash
dig @10.10.10.106 accesswt.com SOA
dig @10.10.10.106 accesswt.com NS
dig @10.10.10.106 ns1.accesswt.com A
```

> Look for `flags: qr aa` — the `aa` flag confirms NS1 is authoritative.

*🔴 Screenshot: dig SOA output from NS1 showing `flags: qr aa` and serial 2024010101.*

---

## 5. NS2 — Secondary Master (10.10.10.107)

> Do NOT generate keys on NS2. Paste the exact same secrets from NS1.

### Install

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc -y
```

### `/etc/bind/named.conf.options`

```bash
sudo tee /etc/bind/named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";
    recursion no;
    allow-query { 10.10.10.0/24; };
    listen-on { 10.10.10.107; };
    dnssec-validation auto;
    listen-on-v6 { none; };
};
EOF
```

### `/etc/bind/named.conf.local`

```bash
sudo tee /etc/bind/named.conf.local << 'EOF'
key "transfer-key" {
    algorithm hmac-sha256;
    secret "YOUR_TRANSFER_KEY_SECRET_HERE";   /* Same secret as NS1 */
};

zone "accesswt.com" {
    type slave;
    file "/var/cache/bind/db.accesswt.com";
    masters { 10.10.10.106 key "transfer-key"; };
    allow-transfer { key "transfer-key"; };
    also-notify { 10.10.10.109; };
    notify yes;
};

zone "10.10.10.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.10.10.10";
    masters { 10.10.10.106 key "transfer-key"; };
    allow-transfer { key "transfer-key"; };
    also-notify { 10.10.10.109; };
    notify yes;
};
EOF
```

### Verify and Start

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl enable named
```

### Verify Zone Transfer Succeeded

```bash
sudo journalctl -u named -n 30 --no-pager
```

Expected output:

```
zone accesswt.com/IN: Transfer started.
zone accesswt.com/IN: transferred serial 2024010101: TSIG 'transfer-key'
Transfer status: success
```

*🔴 Screenshot: journalctl output on NS2 showing successful zone transfer with TSIG 'transfer-key'.*

```bash
# Zone files should now exist
ls -l /var/cache/bind/db.accesswt.com
ls -l /var/cache/bind/db.10.10.10

# SOA serials must match NS1
dig @10.10.10.106 accesswt.com SOA
dig @10.10.10.107 accesswt.com SOA
```

*🔴 Screenshot: dig SOA from NS1 and NS2 side by side, showing matching serial numbers.*

---

## 6. NS3 — Resolver (10.10.10.108)

> Do NOT generate keys. Paste the same secret from NS1.
> Zone statements must be in `named.conf.local`, NOT inside `named.conf.options`.

### Install

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc -y
```

### `/etc/bind/named.conf.options`

```bash
sudo tee /etc/bind/named.conf.options << 'EOF'
acl trusted-clients {
    10.10.10.0/24;
    localhost;
};

options {
    directory "/var/cache/bind";

    allow-query     { trusted-clients; };
    recursion yes;
    allow-recursion { trusted-clients; };

    dnssec-validation no;
    listen-on { 10.10.10.108; };
    listen-on-v6 { none; };
};
EOF
```

**Why `dnssec-validation no`?**

With `dnssec-validation auto`, BIND tries to validate every response against the global DNSSEC chain. Since `accesswt.com` is an internal zone with no published DNSSEC records, validation fails:

```
broken trust chain resolving 'accesswt.com/SOA/IN'
validating accesswt.com/SOA: got insecure response; parent indicates it should be secure
```

Setting `dnssec-validation no` disables this check, allowing the resolver to accept unsigned internal responses.

### `/etc/bind/named.conf.local`

```bash
sudo tee /etc/bind/named.conf.local << 'EOF'
key "transfer-key" {
    algorithm hmac-sha256;
    secret "YOUR_TRANSFER_KEY_SECRET_HERE";
};

zone "accesswt.com" {
    type forward;
    forward only;
    forwarders {
        10.10.10.106;
        10.10.10.107;
    };
};

zone "10.10.10.in-addr.arpa" {
    type forward;
    forward only;
    forwarders {
        10.10.10.106;
        10.10.10.107;
    };
};
EOF
```

### Verify and Start

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl enable named
```

### Verify NS3

```bash
dig @10.10.10.108 accesswt.com SOA
dig @10.10.10.108 ns1.accesswt.com A
dig @10.10.10.108 accesswt.com NS
dig @10.10.10.108 accesswt.com MX
dig @10.10.10.108 -x 10.10.10.106
```

*🔴 Screenshot: All dig outputs from NS3 showing status: NOERROR with answers.*

---

## 7. NS4 — Resolver (10.10.10.109)

Identical to NS3 except `listen-on` IP.

### Install

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc -y
```

### `/etc/bind/named.conf.options`

```bash
sudo tee /etc/bind/named.conf.options << 'EOF'
acl trusted-clients {
    10.10.10.0/24;
    localhost;
};

options {
    directory "/var/cache/bind";

    allow-query     { trusted-clients; };
    recursion yes;
    allow-recursion { trusted-clients; };

    dnssec-validation no;
    listen-on { 10.10.10.109; };
    listen-on-v6 { none; };
};
EOF
```

### `/etc/bind/named.conf.local`

```bash
sudo tee /etc/bind/named.conf.local << 'EOF'
key "transfer-key" {
    algorithm hmac-sha256;
    secret "YOUR_TRANSFER_KEY_SECRET_HERE";
};

zone "accesswt.com" {
    type forward;
    forward only;
    forwarders {
        10.10.10.106;
        10.10.10.107;
    };
};

zone "10.10.10.in-addr.arpa" {
    type forward;
    forward only;
    forwarders {
        10.10.10.106;
        10.10.10.107;
    };
};
EOF
```

### Verify and Start

```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl enable named
```

### Verify NS4

```bash
dig @10.10.10.109 accesswt.com SOA
dig @10.10.10.109 accesswt.com NS
dig @10.10.10.109 ns1.accesswt.com A
dig @10.10.10.109 accesswt.com MX
dig @10.10.10.109 -x 10.10.10.106
```

*🔴 Screenshot: All dig outputs from NS4 showing NOERROR for all record types.*

---

## 8. Client VM Configuration — Which NS to Add

**Always point clients to NS3 and NS4 only. Never to NS1 or NS2.**

NS1 and NS2 are authoritative-only servers with `recursion no`. A client querying them directly for `google.com` would get `REFUSED`. Only NS3 and NS4 have `recursion yes` and can chase answers for external names.

### Static Configuration — `/etc/resolv.conf`

```bash
sudo tee /etc/resolv.conf << 'EOF'
search accesswt.com
nameserver 10.10.10.108
nameserver 10.10.10.109
EOF
```

### Persistent Configuration — Netplan (Ubuntu 20.04+)

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses: [10.10.10.X/24]
      gateway4: 10.10.10.1
      nameservers:
        addresses:
          - 10.10.10.108
          - 10.10.10.109
        search:
          - accesswt.com
```

```bash
sudo netplan apply
```

### Test from Client VM

```bash
nslookup ns1.accesswt.com
nslookup accesswt.com
nslookup google.com
dig accesswt.com SOA
```

*🔴 Screenshot: nslookup outputs from a client VM successfully resolving both internal (accesswt.com) and external (google.com) names.*

---

## 9. Adding DNS Records — Static vs Dynamic

### Method 1: Static — Direct Zone File Edit on NS1

This is the standard method. You edit the zone file on NS1 and reload BIND.

**Step 1: Edit the zone file**

```bash
sudo nano /etc/bind/zones/db.accesswt.com
```

Add records at the bottom:

```
; A Records
www     IN A   10.10.10.200
app     IN A   10.10.10.201
vpn     IN A   10.10.10.202

; CNAME Record
ftp     IN CNAME  www.accesswt.com.

; Additional MX
@       IN MX 30 mail3.accesswt.com.

; TXT Record
verify  IN TXT  "google-site-verification=abcdef123456"
```

**Step 2: Increment the serial**

```
2024010101  →  2024010102
```

> If you forget to increment the serial, slave servers will NOT pull the update.

**Step 3: Verify and reload**

```bash
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
sudo rndc reload accesswt.com
```

**Step 4: Verify propagation**

```bash
dig @10.10.10.106 www.accesswt.com A
dig @10.10.10.107 www.accesswt.com A
dig @10.10.10.108 www.accesswt.com A
dig @10.10.10.109 www.accesswt.com A
```

*🔴 Screenshot: dig A record for www.accesswt.com from all 4 servers showing the same answer.*

### Method 2: Dynamic DNS — nsupdate

Dynamic DNS lets you add/update/delete records without touching the zone file directly. It is used by automation, DHCP servers, or scripts.

**Prerequisite:** NS1 must have `update-policy` configured in `named.conf.local` (already done above with `ddns-key`).

```bash
# From any machine with access to NS1 port 53
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update add www.accesswt.com 3600 A 10.10.10.200
send
EOF
```

For reverse record:

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone 10.10.10.in-addr.arpa
update add 200.10.10.10.in-addr.arpa 3600 PTR www.accesswt.com.
send
EOF
```

**Verify**

```bash
dig @10.10.10.106 www.accesswt.com A
dig @10.10.10.106 -x 10.10.10.200
```

*🔴 Screenshot: nsupdate command and the resulting dig output confirming the record was added.*

---

## 10. Dynamic DNS Update via nsupdate

### How nsupdate Works

`nsupdate` sends RFC 2136 dynamic update packets to NS1. NS1 verifies the TSIG signature against the `ddns-key`, applies the update to the live zone in memory, and writes the change to the journal file (`.jnl`). The journal is periodically merged into the zone file.

### Adding Multiple Records at Once

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update add web1.accesswt.com 3600 A 10.10.10.210
update add web2.accesswt.com 3600 A 10.10.10.211
update add api.accesswt.com  3600 A 10.10.10.212
update add api.accesswt.com  3600 TXT "env=production"
send
EOF
```

### Deleting a Record with nsupdate

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update delete web1.accesswt.com A
send
EOF
```

### Updating an Existing Record

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update delete web1.accesswt.com A
update add web1.accesswt.com 3600 A 10.10.10.220
send
EOF
```

*🔴 Screenshot: nsupdate session adding a record, then dig confirming the record appears on NS1.*

---

## 11. Adding DNS from a Resolver — Can We? How to Verify?

### Can We Add Records from NS3 or NS4?

**No, not directly.** NS3 and NS4 are `type forward` servers. They hold no zone data and have no write access to any zone. You can only update records on the authoritative master (NS1) using `nsupdate`.

However, you can **run `nsupdate` targeting NS1 from NS3's shell**:

```bash
# On NS3 (10.10.10.108) — pointing update at NS1
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update add test.accesswt.com 3600 A 10.10.10.250
send
EOF
```

### Verifying the Update Reached NS1 (Primary Zone File)

After running `nsupdate`, verify the update propagated through the entire chain:

**Step 1: Check NS1 zone file is updated**

```bash
# On NS1 — check journal file was created
ls -la /etc/bind/zones/db.accesswt.com.jnl

# Force journal to flush into zone file
sudo rndc sync accesswt.com

# Now check the zone file
grep "test" /etc/bind/zones/db.accesswt.com
```

**Step 2: Check all 4 servers answer the query**

```bash
dig @10.10.10.106 test.accesswt.com A   # NS1 — must answer
dig @10.10.10.107 test.accesswt.com A   # NS2 — must have transferred
dig @10.10.10.108 test.accesswt.com A   # NS3 — must forward correctly
dig @10.10.10.109 test.accesswt.com A   # NS4 — must forward correctly
```

All four must return the same answer.

**Step 3: Check zone serial incremented on NS1**

```bash
dig @10.10.10.106 accesswt.com SOA
dig @10.10.10.107 accesswt.com SOA
```

NS1 serial will have auto-incremented after the dynamic update. NS2 must match.

*🔴 Screenshot: nsupdate from NS3 terminal, followed by dig confirming answer on all 4 servers.*

---

## 12. How Dynamic DNS Hits the Same File as Static Edits

This is a critical concept. When you run `nsupdate`, BIND does NOT write directly to `db.accesswt.com`. Instead:

```
nsupdate sends update packet
        │
        ▼
NS1 BIND verifies TSIG with ddns-key
        │
        ▼
Update applied to in-memory zone
        │
        ▼
Written to journal file: db.accesswt.com.jnl
        │
        ▼ (periodically, or on rndc sync)
Journal merged into zone file: db.accesswt.com
        │
        ▼
NOTIFY sent to NS2, NS3 (who then forward to NS4)
        │
        ▼
NS2 pulls updated zone via AXFR
```

### Forcing the Journal to Merge

```bash
# Force journal to merge into zone file NOW
sudo rndc sync accesswt.com

# Or sync all zones
sudo rndc sync -clean
```

After this, `db.accesswt.com` will contain the dynamically added record exactly as if you had added it with a text editor. The two methods produce the same end result in the zone file.

### Important Warning

If you edit `db.accesswt.com` with a text editor while BIND has an unsynced journal, BIND may reject or overwrite your edits. Always run `sudo rndc sync` before manually editing the file when dynamic updates have been used.

*🔴 Screenshot: Contents of db.accesswt.com before and after `rndc sync`, showing the dynamically added record now present in the file.*

---

## 13. Adding and Removing Records

### Adding Records — Static Method

Edit `/etc/bind/zones/db.accesswt.com` on NS1:

```bash
sudo nano /etc/bind/zones/db.accesswt.com
```

**Add an A record:**

```
webapp  IN A   10.10.10.205
```

**Add a CNAME:**

```
portal  IN CNAME  webapp.accesswt.com.
```

**Add an MX record:**

```
@       IN MX 30 mail3.accesswt.com.
mail3   IN A   10.10.10.182
```

**Add a TXT record:**

```
_acme-challenge  IN TXT  "validation-token-here"
```

Then increment serial and reload:

```bash
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
sudo rndc reload accesswt.com
```

### Removing Records — Static Method

Open the zone file, delete the line(s), increment serial:

```bash
sudo nano /etc/bind/zones/db.accesswt.com
# Remove the record line
# Increment serial
sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
sudo rndc reload accesswt.com
```

### Adding Records — Dynamic Method (nsupdate)

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update add newhost.accesswt.com 3600 A 10.10.10.230
send
EOF
```

### Removing Records — Dynamic Method (nsupdate)

```bash
nsupdate -k /etc/bind/keys/ddns.key << 'EOF'
server 10.10.10.106
zone accesswt.com
update delete newhost.accesswt.com A
send
EOF
```

### Verify Addition or Removal

```bash
dig @10.10.10.108 newhost.accesswt.com A
```

*🔴 Screenshot: Zone file showing added record. Then dig output confirming it resolves. Then zone file after deletion and dig confirming NXDOMAIN.*

---

## 14. Whitelisting and Blacklisting DNS

### Whitelisting — Allow Only Specific Clients

Whitelist means only specific IPs or subnets can query the server.

**On NS3 and NS4 (Resolvers)** — already implemented via ACL:

```conf
acl trusted-clients {
    10.10.10.0/24;    # Allow entire client subnet
    localhost;         # Allow local queries
};

options {
    allow-query     { trusted-clients; };
    allow-recursion { trusted-clients; };
};
```

**To whitelist individual IPs:**

```conf
acl trusted-clients {
    10.10.10.50;    # Specific host
    10.10.10.51;
    10.10.10.0/24;
    localhost;
};
```

**On NS1 and NS2 (Authoritative)** — already whitelist-only:

```conf
options {
    allow-query { 10.10.10.0/24; };
    allow-transfer { key "transfer-key"; };  # TSIG key is the whitelist
};
```

### Blacklisting — Block Specific Clients

**Block a specific IP from querying:**

```conf
acl blocked-clients {
    10.10.10.99;      # Bad actor
    192.168.5.0/24;   # Untrusted network
};

options {
    allow-query { !blocked-clients; 10.10.10.0/24; };
    # The ! means NOT — blocked-clients is evaluated first
};
```

**Block all queries except whitelist:**

```conf
options {
    allow-query { 10.10.10.0/24; localhost; };
    # Anything not in the list is implicitly denied
};
```

### Blacklisting Specific Domains (Response Policy Zone — RPZ)

RPZ allows you to block resolution of specific external domains — useful for blocking malware domains or ad networks.

**Add to `named.conf.local` on NS3 and NS4:**

```conf
zone "rpz.local" {
    type master;
    file "/etc/bind/zones/db.rpz";
};

options {
    response-policy { zone "rpz.local"; };
};
```

**Create `/etc/bind/zones/db.rpz`:**

```
$TTL 300
@ IN SOA localhost. root.localhost. (
        1 3600 900 86400 300 )
@ IN NS localhost.

; Block these domains — return NXDOMAIN
malware-site.com        IN CNAME .
ads.tracking-net.com    IN CNAME .
badactor.org            IN CNAME .
```

```bash
sudo named-checkconf
sudo systemctl restart bind9
```

**Test that a blocked domain returns NXDOMAIN:**

```bash
dig @10.10.10.108 malware-site.com A
# Expected: status: NXDOMAIN
```

*🔴 Screenshot: RPZ zone file, then dig showing NXDOMAIN for a blocked domain and NOERROR for a non-blocked domain.*

### Blacklisting Zone Transfers (Already Hardened)

```conf
zone "accesswt.com" {
    allow-transfer { key "transfer-key"; };  # Only TSIG key holders
};
```

Test that AXFR is refused from unauthorized clients:

```bash
dig @10.10.10.108 accesswt.com AXFR  # Must return REFUSED
dig @10.10.10.109 accesswt.com AXFR  # Must return REFUSED
```

*🔴 Screenshot: AXFR attempt returning REFUSED from NS3 and NS4.*

---

## 15. Full Verification and High Availability Tests

### Verification Matrix

| Test | Command | Expected Result |
|------|---------|-----------------|
| NS1 authoritative | `dig @10.10.10.106 accesswt.com SOA` | `flags: qr aa` — serial 2024010101 |
| NS2 authoritative | `dig @10.10.10.107 accesswt.com SOA` | `flags: qr aa` — same serial |
| NS3 resolves | `dig @10.10.10.108 accesswt.com SOA` | `status: NOERROR` with answer |
| NS4 resolves | `dig @10.10.10.109 accesswt.com SOA` | `status: NOERROR` with answer |
| NS records | `dig @10.10.10.107 accesswt.com NS` | 4 NS records returned |
| MX records | `dig @10.10.10.107 accesswt.com MX` | mail1 (10), mail2 (20) |
| TXT/SPF | `dig @10.10.10.107 accesswt.com TXT` | SPF and DMARC records |
| Reverse lookup | `dig @10.10.10.106 -x 10.10.10.106` | ns1.accesswt.com. |
| AXFR blocked | `dig @10.10.10.108 accesswt.com AXFR` | `status: REFUSED` |
| External recursion | `dig @10.10.10.108 google.com A` | valid answer via NS3 |

### Complete Verification Run

```bash
# 1. SOA serial — all 4 must match
dig @10.10.10.106 accesswt.com SOA
dig @10.10.10.107 accesswt.com SOA
dig @10.10.10.108 accesswt.com SOA
dig @10.10.10.109 accesswt.com SOA

# 2. NS records
dig @10.10.10.106 accesswt.com NS

# 3. MX records
dig @10.10.10.107 accesswt.com MX
dig @10.10.10.107 mail1.accesswt.com A
dig @10.10.10.107 mail2.accesswt.com A

# 4. TXT records
dig @10.10.10.107 accesswt.com TXT

# 5. Reverse lookups
dig @10.10.10.106 -x 10.10.10.106
dig @10.10.10.106 -x 10.10.10.107
dig @10.10.10.106 -x 10.10.10.108
dig @10.10.10.106 -x 10.10.10.109

# 6. Zone transfer blocked
dig @10.10.10.108 accesswt.com AXFR
dig @10.10.10.109 accesswt.com AXFR

# 7. External resolution via resolvers
dig @10.10.10.108 google.com A
dig @10.10.10.109 google.com A
```

*🔴 Screenshot: All SOA queries in one terminal showing matching serials. Separate screenshots for MX, TXT, PTR, and AXFR REFUSED.*

### High Availability Tests

#### Test 1: NS1 Goes Down — NS2 Must Still Answer

```bash
# Simulate NS1 failure
sudo systemctl stop bind9     # Run this on NS1

# From another machine, NS2 must still answer authoritatively
dig @10.10.10.107 accesswt.com SOA    # Must return answer with aa flag
dig @10.10.10.108 accesswt.com SOA    # NS3 forwards to NS2, must work
dig @10.10.10.109 accesswt.com SOA    # NS4 forwards to NS2, must work

# Restore NS1
sudo systemctl start bind9     # Run this on NS1
```

*🔴 Screenshot: dig SOA responses from NS2, NS3, NS4 while NS1 is stopped, all returning NOERROR.*

#### Test 2: NS3 Goes Down — NS4 Must Serve Clients

```bash
# Simulate NS3 failure
sudo systemctl stop bind9     # Run this on NS3

# Client should fall over to NS4 automatically (if resolv.conf has both)
# From a client VM:
dig accesswt.com SOA          # Should still resolve via NS4
nslookup accesswt.com         # Should show Server: 10.10.10.109

# Restore NS3
sudo systemctl start bind9    # Run this on NS3
```

*🔴 Screenshot: Client dig output while NS3 is down, showing resolution via NS4 (10.10.10.109).*

#### Test 3: Zone Transfer Failover

```bash
# If NS1 is unreachable, NS2 must use its cached zone
# Confirm NS2 has zone files
ls -l /var/cache/bind/db.accesswt.com
ls -l /var/cache/bind/db.10.10.10

# Check zone status on NS2
sudo rndc zonestatus accesswt.com
```

*🔴 Screenshot: `rndc zonestatus` on NS2 showing loaded zone with correct serial.*

#### Test 4: BIND Service Recovery

```bash
# Crash recovery test — kill BIND hard
sudo kill -9 $(pidof named)

# systemd should restart it automatically (if service is enabled)
sleep 5
sudo systemctl status bind9

# Verify zone loaded correctly after restart
dig @10.10.10.107 accesswt.com SOA
```

*🔴 Screenshot: systemctl status showing BIND restarted automatically after kill.*

#### Test 5: Query Response Time

```bash
# Measure query time
dig @10.10.10.108 accesswt.com A | grep "Query time"
dig @10.10.10.109 accesswt.com A | grep "Query time"

# External query via resolver
dig @10.10.10.108 google.com A | grep "Query time"
```

*🔴 Screenshot: Query time outputs — internal queries should be under 5ms, external may vary.*

---

## 16. Zone Update Workflow

```
Edit zone file on NS1 ONLY
        │
Increment serial number
        │
sudo named-checkzone (verify)
        │
sudo rndc reload accesswt.com
        │
NS1 sends NOTIFY → NS2, NS3
        │
NS2 pulls AXFR (TSIG authenticated)
        │
NS2 sends NOTIFY → NS4
        │
All 4 servers serve new record
        │
Verify: dig @all-4-ips new-record
```

```bash
# Full update workflow
sudo nano /etc/bind/zones/db.accesswt.com
# (edit records, increment serial)

sudo named-checkzone accesswt.com /etc/bind/zones/db.accesswt.com
sudo rndc reload accesswt.com

# Verify serial propagated everywhere
dig @10.10.10.106 accesswt.com SOA | grep serial
dig @10.10.10.107 accesswt.com SOA | grep serial
dig @10.10.10.108 accesswt.com SOA | grep serial
dig @10.10.10.109 accesswt.com SOA | grep serial
```

---

## 17. Troubleshooting Reference

### SERVFAIL on NS3 or NS4

```
status: SERVFAIL
```

- **Cause 1:** `dnssec-validation auto` — resolver rejecting unsigned internal zone with "broken trust chain".
- **Fix:** Set `dnssec-validation no` in `named.conf.options` on NS3 and NS4.
- **Cause 2:** Zone statements placed inside `options {}` block instead of `named.conf.local`.
- **Fix:** Move all `zone` blocks out of `named.conf.options` into `named.conf.local`.

### Zone Transfer Fails on NS2

```
TSIG 'transfer-key': tsig verify failure
```

- **Cause:** Secret string mismatch between NS1 and NS2.
- **Fix:** Copy the exact secret string from NS1's `tsig-keygen` output into NS2's config.

### bind9 enable fails

```
Failed to enable unit: Refusing to operate on alias name or linked unit file: bind9.service
```

- **Cause:** `bind9` is an alias for `named.service` on this system.
- **Fix:** Use `sudo systemctl enable named` instead. `restart bind9` still works.

### IPv6 noise in logs

```
network unreachable resolving './DNSKEY/IN': 2001:500:...
```

- **Cause:** No IPv6 network, but BIND tries IPv6 root servers.
- **Fix:** Add `listen-on-v6 { none };` to all servers' `named.conf.options`.

### No entries in `journalctl -u bind9`

- **Cause:** Service is registered as `named`, not `bind9`.
- **Fix:** Use `sudo journalctl -u named -n 50 --no-pager`.

### Dynamic update fails (nsupdate)

```
update failed: REFUSED
```

- **Cause:** `ddns-key` secret not matching, or `update-policy` not set in NS1's zone config.
- **Fix:** Verify ddns-key secret is identical, and NS1 zone has `update-policy { grant ddns-key zonesub ANY; };`.

---

## 18. Quick Reference Summary

### Key Differences Between Server Roles

| Setting | NS1 | NS2 | NS3 | NS4 |
|---------|-----|-----|-----|-----|
| `recursion` | no | no | yes | yes |
| Zone type | master | slave | forward | forward |
| `dnssec-validation` | auto | auto | no | no |
| `allow-transfer` | key only | key only | none | none |
| Generates TSIG key | YES | no | no | no |
| Serves clients | no | no | YES | YES |
| Zone files | /etc/bind/zones/ | /var/cache/bind/ | none | none |

### Setup Order Checklist

- [ ] Generate TSIG keys on NS1 — copy secrets to all other servers
- [ ] Configure NS1 — options, local, both zone files, AppArmor
- [ ] Verify NS1 zone files with `named-checkzone`
- [ ] Start NS1 — verify `aa` flag in dig response
- [ ] Configure NS2 — paste TSIG secret, set type slave pointing to NS1
- [ ] Start NS2 — verify zone transfer success in `journalctl -u named`
- [ ] Configure NS3 — `recursion yes`, `dnssec-validation no`, forward zones in `named.conf.local`
- [ ] Start NS3 — verify NOERROR for all record types
- [ ] Configure NS4 — identical to NS3 with different listen-on IP
- [ ] Start NS4 — verify NOERROR for all record types
- [ ] Configure client VMs — point to NS3 and NS4 only
- [ ] Run full verification matrix — all checks must pass
- [ ] Run HA tests — verify failover with each server stopped individually

### Diagnostic Commands

| Command | Purpose |
|---------|---------|
| `sudo named-checkconf` | Verify all config files for syntax errors |
| `sudo named-checkzone <zone> <file>` | Verify zone file syntax |
| `sudo journalctl -u named -n 50` | View BIND logs |
| `sudo rndc status` | Show BIND server status |
| `sudo rndc zonestatus accesswt.com` | Show zone transfer status |
| `sudo rndc reload accesswt.com` | Reload zone and trigger NOTIFY |
| `sudo rndc sync accesswt.com` | Flush DDNS journal to zone file |
| `dig @<IP> <zone> SOA` | Check SOA serial on specific server |
| `dig @<IP> <zone> AXFR` | Test if AXFR is blocked (should be REFUSED) |
| `dig @<IP> -x <IP>` | Reverse DNS lookup |
| `nsupdate -k <keyfile>` | Send dynamic DNS update |

---

*Guide compiled from live implementation on accesswt.com — all steps verified and all errors encountered documented in Section 17.*

---
---
---

# Understanding the BIND Configuration Files and Why They Are Configured

Those paths are configuration and zone database files used by **BIND9** (the DNS server). Here's what each one is for.

| Path | Purpose |
|-------|---------|
| `/etc/bind/named.conf.options` | Global BIND server options such as recursion, listening IPs, DNSSEC, and cache settings. |
| `/etc/bind/named.conf.local` | Local DNS configuration where you define your own zones, TSIG keys, and custom settings. |
| `/var/cache/bind` | BIND's working/cache directory. It stores cached DNS data and dynamic zone files if configured. |
| `/etc/bind/zones/db.isumanbhandari.com.np` | The forward DNS zone file for `isumanbhandari.com.np`. It contains A, AAAA, MX, CNAME, TXT, NS, etc. records. |
| `/etc/bind/zones/db.172.18.1` | The reverse DNS zone file for the `172.18.1.0/24` network. It contains PTR records that map IP addresses back to hostnames. |

---

# 1. `/etc/bind/named.conf.options`

This file configures the DNS server itself.

```conf
options {
    directory "/var/cache/bind";
    recursion no;
    allow-query { any; };
    listen-on { 172.18.1.171; };
    dnssec-validation auto;
    listen-on-v6 { none; };
};
```

### Meaning

- `directory "/var/cache/bind";`
  - Working directory used by BIND.

- `recursion no;`
  - Your server is **authoritative only**.
  - It won't perform recursive lookups like Google DNS (8.8.8.8).

- `allow-query { any; };`
  - Anyone can query your DNS server.

- `listen-on { 172.18.1.171; };`
  - Only listen on that IPv4 address.

- `dnssec-validation auto;`
  - Enables DNSSEC validation for recursive lookups.
  - With `recursion no`, this setting has little practical effect.

- `listen-on-v6 { none; };`
  - Disable IPv6.

---

# 2. `/etc/bind/named.conf.local`

This is where you define **your DNS zones**.

It contains:

- TSIG keys
- forward zones
- reverse zones
- slave/master definitions

Example:

```conf
zone "isumanbhandari.com.np" {
    type master;
    file "/etc/bind/zones/db.isumanbhandari.com.np";
};
```

This tells BIND:

> "I am the authoritative master for this domain, and the DNS records are stored in `/etc/bind/zones/db.isumanbhandari.com.np`."

---

# 3. `/etc/bind/zones/db.isumanbhandari.com.np`

This file does **not exist automatically**.

You must create it.

Example:

```dns
$TTL 86400

@   IN SOA ns1.isumanbhandari.com.np. admin.isumanbhandari.com.np. (
        2026062901
        3600
        1800
        604800
        86400 )

@       IN NS ns1.isumanbhandari.com.np.
@       IN NS ns2.isumanbhandari.com.np.

ns1     IN A 172.18.1.171
ns2     IN A 172.18.1.172
www     IN A 172.18.1.50
mail    IN A 172.18.1.60

@       IN MX 10 mail
```

This is your **forward lookup** database.

For example:

```text
www.isumanbhandari.com.np
        ↓
172.18.1.50
```

---

# 4. `/etc/bind/zones/db.172.18.1`

This is the **reverse zone**.

Example:

```dns
$TTL 86400

@   IN SOA ns1.isumanbhandari.com.np. admin.isumanbhandari.com.np. (
        2026062901
        3600
        1800
        604800
        86400 )

@   IN NS ns1.isumanbhandari.com.np.
@   IN NS ns2.isumanbhandari.com.np.

171 IN PTR ns1.isumanbhandari.com.np.
172 IN PTR ns2.isumanbhandari.com.np.
50  IN PTR www.isumanbhandari.com.np.
60  IN PTR mail.isumanbhandari.com.np.
```

This maps:

```text
172.18.1.50
      ↓
www.isumanbhandari.com.np
```

---

# 5. `/var/cache/bind`

This directory is created when BIND is installed.

Typical contents:

```text
/var/cache/bind/
├── managed-keys.bind
├── managed-keys.bind.jnl
├── tmp-*
└── ...
```

If you configure **dynamic DNS** with zone files stored there, you'll also see:

```text
db.example.com.jnl
```

These `.jnl` files are journals that track dynamic updates.

---

# Your TSIG Keys

You also define two TSIG keys.

## `transfer-key`

```conf
key "transfer-key"
```

Used for:

- AXFR
- IXFR
- secure zone replication

Only servers with this key can copy your zones.

---

## `ddns-key`

```conf
key "ddns-key"
```

Used for:

- Dynamic DNS updates
- DHCP integration
- Automation (for example, `nsupdate`)

Only clients that possess this key can modify DNS records in the zone according to the `update-policy`.

---

# Typical Directory Structure

A common layout on a Debian/Ubuntu system running BIND looks like:

```text
/etc/bind/
├── named.conf
├── named.conf.options
├── named.conf.local
├── named.conf.default-zones
├── rndc.key
└── zones/
    ├── db.isumanbhandari.com.np
    └── db.172.18.1

/var/cache/bind/
    managed-keys.bind
    managed-keys.bind.jnl
    ...
```

One thing to note: the `zones` subdirectory (`/etc/bind/zones/`) is **not created automatically** by BIND on most installations.

Before writing your zone files there, you'll typically need to create it:

```bash
sudo mkdir -p /etc/bind/zones
sudo chown root:bind /etc/bind/zones
sudo chmod 775 /etc/bind/zones
```

After that, you can place `db.isumanbhandari.com.np` and `db.172.18.1` into that directory, and BIND will load them according to the `named.conf.local` configuration.

---

# Why Do We Configure Each of These Files?

Each file has a different responsibility.

BIND separates configuration into multiple files so that server settings, zone definitions, and DNS records are managed independently.

This makes the configuration easier to understand, maintain, and troubleshoot.

| File | Why it's configured |
|------|---------------------|
| `/etc/bind/named.conf.options` | Configure how the DNS server behaves. |
| `/etc/bind/named.conf.local` | Tell BIND which DNS zones it is responsible for and how to manage them. |
| `/etc/bind/zones/db.isumanbhandari.com.np` | Store the actual DNS records for your domain (forward lookups). |
| `/etc/bind/zones/db.172.18.1` | Store reverse DNS (IP → hostname) records. |
| `/var/cache/bind` | BIND's working directory for cached data and dynamic update files. |

---

# 1. `/etc/bind/named.conf.options`

## Why do we configure it?

This file controls **how the DNS server operates**, regardless of the domains it hosts.

Think of it as configuring the DNS server itself.

For example:

```conf
options {
    recursion no;
    listen-on { 172.18.1.171; };
    allow-query { any; };
};
```

These settings answer questions like:

- Should this server perform recursive lookups?
- Which network interfaces should it listen on?
- Who is allowed to query it?
- Should IPv6 be enabled?
- Where should it keep its cache?

Without this file, BIND wouldn't know its general operating behavior.

---

# 2. `/etc/bind/named.conf.local`

## Why do we configure it?

This file tells BIND:

> "Which domains am I authoritative for?"

For example:

```conf
zone "isumanbhandari.com.np" {
    type master;
    file "/etc/bind/zones/db.isumanbhandari.com.np";
};
```

This means:

> "If someone asks about `isumanbhandari.com.np`, use the records in this zone file."

It also defines:

- master/slave relationships
- TSIG keys
- zone transfers
- dynamic DNS permissions
- notification settings

Without this file, BIND wouldn't know which zones to serve.

---

# 3. `/etc/bind/zones/db.isumanbhandari.com.np`

## Why do we configure it?

This file contains the **actual DNS records** for your domain.

Example:

```dns
www     IN A      172.18.1.50
mail    IN A      172.18.1.60
@        IN MX 10 mail
```

When a client asks:

> "What is the IP address of `www.isumanbhandari.com.np`?"

BIND reads this file and answers:

```text
172.18.1.50
```

Without this file, BIND knows it serves the zone but has no records to return.

---

# 4. `/etc/bind/zones/db.172.18.1`

## Why do we configure it?

DNS also supports reverse lookups.

Instead of:

```text
Hostname → IP
```

it answers:

```text
IP → Hostname
```

Example:

```dns
50 IN PTR www.isumanbhandari.com.np.
```

Now when someone asks:

```text
Who owns 172.18.1.50?
```

the server replies:

```text
www.isumanbhandari.com.np
```

Reverse DNS is commonly used by:

- mail servers
- logging systems
- monitoring tools
- network troubleshooting

---

# 5. `/var/cache/bind`

## Why do we configure it?

This isn't a file you typically edit.

Instead, the `directory` setting tells BIND where to store its working data.

For example:

```conf
options {
    directory "/var/cache/bind";
};
```

BIND uses this directory for:

- cached DNS responses (if recursion is enabled)
- DNSSEC managed keys
- journal (`.jnl`) files created by dynamic DNS updates
- temporary working files

Even if your server is authoritative-only (`recursion no;`), BIND may still use this directory for internal state such as DNSSEC keys or dynamic update journals.

---

# How These Pieces Work Together

Imagine a user requests the IP address for `www.isumanbhandari.com.np`.

1. **`named.conf.options`** tells BIND **how to behave** (for example, listen on `172.18.1.171` and allow queries).

2. **`named.conf.local`** tells BIND **which zone** contains that domain (`isumanbhandari.com.np`).

3. BIND opens **`db.isumanbhandari.com.np`** to find the requested record.

4. If the user instead performs a reverse lookup (for example, `172.18.1.50` → hostname), BIND consults **`db.172.18.1`**.

5. If dynamic updates or other runtime data are involved, BIND stores the necessary working files in **`/var/cache/bind`**.

---

# Simple Way to Remember

- **`named.conf.options`**
  - **How should the DNS server operate?**

- **`named.conf.local`**
  - **What zones should the server manage?**

- **Zone files (`db.*`)**
  - **What DNS records exist within those zones?**

- **`/var/cache/bind`**
  - **Where should the server keep its runtime data?**

This separation keeps global server behavior, zone definitions, DNS data, and runtime files independent, making the configuration easier to maintain and less error-prone.


# Static DNS Record Update on a Dynamic BIND9 DNS Server

## Scenario

Add a static DNS record:

| Record     | Value                 |
| ---------- | --------------------- |
| Hostname   | `zimbra.accesswt.com` |
| IP Address | `10.10.10.110`        |

The BIND server is configured as a **dynamic primary zone** using:

```conf
update-policy {
    grant ddns-key zonesub ANY;
};
```

Because the zone is dynamic, **editing the zone file alone is not enough**. The correct workflow is to **freeze the zone**, edit the files, then **thaw the zone** so BIND reloads the updated records.

---

## Step 1 — Verify the Zone Configuration

Check the zone configuration:

```bash
cat /etc/bind/named.conf.local
```

Example:

```conf
zone "accesswt.com" {
    type master;
    file "/etc/bind/zones/db.accesswt.com";

    update-policy {
        grant ddns-key zonesub ANY;
    };

    allow-transfer { key "transfer-key"; };
    also-notify { 10.10.10.107; 10.10.10.108; };
    notify yes;
};

zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.10.10";

    update-policy {
        grant ddns-key zonesub ANY;
    };
};
```

> **Important**
>
> The presence of `update-policy` means the zone is **dynamic**.
>
> Manual edits require the zone to be **frozen** before changes are made.

---

## Step 2 — Backup the Zone Files

```bash
sudo cp /etc/bind/zones/db.accesswt.com \
    /etc/bind/zones/db.accesswt.com.bak

sudo cp /etc/bind/zones/db.10.10.10 \
    /etc/bind/zones/db.10.10.10.bak
```

---

## Step 3 — Freeze the Dynamic Zones

Freeze both the forward and reverse zones.

```bash
sudo rndc freeze accesswt.com
sudo rndc freeze 10.10.10.in-addr.arpa
```

Verify:

```bash
sudo rndc zonestatus accesswt.com
```

Expected:

```text
dynamic: yes
frozen: yes
```

---

## Step 4 — Edit the Forward Zone

Open the zone file:

```bash
sudo nano /etc/bind/zones/db.accesswt.com
```

Add:

```dns
zimbra      IN      A       10.10.10.110
```

Example:

```dns
mail1       IN      A       10.10.10.180
mail2       IN      A       10.10.10.181

zimbra      IN      A       10.10.10.110
```

---

## Step 5 — Edit the Reverse Zone

Open:

```bash
sudo nano /etc/bind/zones/db.10.10.10
```

Add:

```dns
110     IN      PTR     zimbra.accesswt.com.
```

Example:

```dns
106     IN PTR ns1.accesswt.com.
107     IN PTR ns2.accesswt.com.
108     IN PTR ns3.accesswt.com.
109     IN PTR ns4.accesswt.com.

110     IN PTR zimbra.accesswt.com.

180     IN PTR mail1.accesswt.com.
181     IN PTR mail2.accesswt.com.
```

> **Note:** The trailing period (`.`) after the FQDN is required.

---

## Step 6 — Increment the SOA Serial

Increase the serial number in **both** zone files.

Old:

```text
2024010101
```

New:

```text
2024010102
```

Example:

```dns
@ IN SOA ns1.accesswt.com. admin.accesswt.com. (
        2024010102
        604800
        86400
        2419200
        604800
)
```

---

## Step 7 — Validate the Zone Files

Validate the forward zone:

```bash
sudo named-checkzone accesswt.com \
    /etc/bind/zones/db.accesswt.com
```

Expected:

```text
zone accesswt.com/IN: loaded serial 2024010102
OK
```

Validate the reverse zone:

```bash
sudo named-checkzone \
    10.10.10.in-addr.arpa \
    /etc/bind/zones/db.10.10.10
```

Expected:

```text
zone 10.10.10.in-addr.arpa/IN: loaded serial 2024010102
OK
```

Validate the BIND configuration:

```bash
sudo named-checkconf
```

No output indicates the configuration is valid.

---

## Step 8 — Thaw the Zones

Reload the updated zones:

```bash
sudo rndc thaw accesswt.com
sudo rndc thaw 10.10.10.in-addr.arpa
```

Expected:

```text
A zone reload and thaw was started.
Check the logs to see the result.
```

---

## Step 9 — Verify the Zone Loaded

Check:

```bash
sudo rndc zonestatus accesswt.com
```

Expected:

```text
serial: 2024010102
dynamic: yes
frozen: no
```

The important part is:

```text
frozen: no
```

The zone is now active again.

---

## Step 10 — Test DNS Resolution

### Forward Lookup

```bash
dig @10.10.10.106 zimbra.accesswt.com
```

Expected:

```text
;; ANSWER SECTION:

zimbra.accesswt.com.    604800    IN    A    10.10.10.110
```

### Reverse Lookup

```bash
dig @10.10.10.106 -x 10.10.10.110
```

Expected:

```text
110.10.10.10.in-addr.arpa.    IN    PTR    zimbra.accesswt.com.
```

### Verify with `host`

```bash
host zimbra.accesswt.com 10.10.10.106
```

Expected:

```text
zimbra.accesswt.com has address 10.10.10.110
```

### Verify with `nslookup`

```bash
nslookup zimbra.accesswt.com 10.10.10.106
```

Expected:

```text
Name: zimbra.accesswt.com
Address: 10.10.10.110
```

---

## Step 11 — Verify Slave Replication

The master configuration includes:

```conf
also-notify {
    10.10.10.107;
    10.10.10.108;
};
```

Verify that the slave servers have received the update:

```bash
dig @10.10.10.107 zimbra.accesswt.com
dig @10.10.10.108 zimbra.accesswt.com

dig @10.10.10.107 -x 10.10.10.110
dig @10.10.10.108 -x 10.10.10.110
```

Expected:

```text
zimbra.accesswt.com.    IN    A    10.10.10.110

110.10.10.10.in-addr.arpa.    IN    PTR    zimbra.accesswt.com.
```

---

## Why Freeze and Thaw?

Initially, the zone file was edited and the serial incremented, but DNS still returned the old serial:

```text
serial: 2024010101
```

This happened because the zone was configured as a **dynamic zone** (`update-policy` enabled).

For dynamic zones:

* Editing the zone file alone does **not** reload the in-memory zone.
* Attempting to reload a dynamic zone directly returns:

```text
rndc: 'reload' failed: dynamic zone
```

The correct workflow is:

```text
Freeze
    ↓
Edit zone file
    ↓
Increment SOA serial
    ↓
Validate zone
    ↓
Thaw
    ↓
BIND reloads the zone
```

After thawing, the server correctly loaded:

```text
serial: 2024010102
nodes: 9
dynamic: yes
frozen: no
```

Both forward and reverse lookups returned the expected records.

---

# Summary

For BIND zones configured with `update-policy`:

1. Back up the zone files.
2. Freeze the zone (`rndc freeze`).
3. Edit the forward and reverse zone files.
4. Increment the SOA serial number.
5. Validate with `named-checkzone`.
6. Validate the configuration with `named-checkconf`.
7. Thaw the zone (`rndc thaw`).
8. Confirm the updated serial using `rndc zonestatus`.
9. Test forward and reverse DNS resolution.
10. Verify that the slave DNS servers received the updated records.

This workflow ensures that manual changes are correctly loaded into BIND while preserving support for dynamic DNS updates.

