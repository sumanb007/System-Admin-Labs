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

