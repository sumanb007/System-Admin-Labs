# Enterprise AAA Architecture: Complete Setup Guide
## TACACS+ HA · LDAP HA · PAM Integration · Zero Local User Provisioning

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Deployment Plan](#2-deployment-plan)
3. [Why use keepalived,VIP](#3-why-use-keepalived,vip)
4. [Network & Node Reference](#2-network--node-reference)
5. [Phase 1: HA TACACS+ Servers (tac1 & tac2)](#3-phase-1-ha-tacacs-servers)
6. [Phase 2: TACACS+ High Availability via Keepalived](#4-phase-2-tacacs-high-availability)
7. [Phase 3: HA LDAP Servers (ldap1 & ldap2)](#5-phase-3-ha-ldap-servers)
8. [Phase 4: LDAP High Availability via Keepalived](#6-phase-4-ldap-high-availability)
9. [Phase 5: LDAP Data Seeding](#7-phase-5-ldap-data-seeding)
10. [Phase 6: TACACS & LDAP Server Nodes as LDAP Clients](#8-phase-6-tacacs--ldap-server-nodes-as-ldap-clients)
11. [Phase 7: Ubuntu Client VM Provisioning](#9-phase-7-ubuntu-client-vm-provisioning)
12. [Phase 8: PAM Stack Configuration](#10-phase-8-pam-stack-configuration)
13. [Phase 9: Verification & Testing](#11-phase-9-verification--testing)
14. [Key Lessons & Syntax Reference](#12-key-lessons--syntax-reference)
15. [Authentication Flow Diagram](#13-authentication-flow-diagram)

---

## 1. Architecture Overview

This guide documents the construction of a fully centralized AAA (Authentication, Authorization, Accounting) infrastructure. The design goals are:

- **No local user creation** required on any client VM
- **No manual home directory setup** — auto-created on first login
- **High availability** for both TACACS+ and LDAP via virtual IPs
- **Layered authentication**: TACACS+ first, LDAP fallback, local last
- **Centralized identity**: all user data lives in LDAP, consumed by all nodes

<img width="900" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/proxmox-hypervisor.png">

---

## 2. Deployment Plan

---

## 3. Why use keepalived,VIP

---

## 4. Network & Node Reference

| Role | Hostname | OS | IP | VIP |
|---|---|---|---|---|
| TACACS+ Master | tac1 | CentOS Stream 9 | 10.10.10.125 | 10.10.10.120 |
| TACACS+ Backup | tac2 | CentOS Stream 9 | 10.10.10.126 | 10.10.10.120 |
| LDAP Master | ldap1 | CentOS Stream 9 | 10.10.10.101 | 10.10.10.110 |
| LDAP Backup | ldap2 | Ubuntu 22.04.5 | 10.10.10.102 | 10.10.10.110 |
| Client VM | cpanel | Ubuntu 24.04 LTS | 10.10.10.202 | — |
| Subnet | — | — | 10.10.10.0/24 | — |

---

## 5. Phase 1: HA TACACS+ Servers

Execute on **both tac1 and tac2**.

### 5.1 Install Build Tools

```bash
sudo yum groupinstall "Development Tools" -y
sudo yum install zlib-devel openssl-devel pcre2-devel -y
```

### 5.2 Build tac_plus-ng from Source

```bash
git clone https://github.com/MarcJHuber/event-driven-servers.git
cd event-driven-servers
./configure
make
sudo make install
```

### 5.3 Create TACACS+ Configuration

> **Critical syntax rules** for `tac_plus-ng` (learned through iterative debugging):
> - `rule { }` not `rule-1 { }` — named labels are invalid
> - `script` inside a profile expects an ACL name, not a brace block
> - `service` is not valid inside `group`, `user`, or `profile` blocks
> - `profile` block accepts: `enable`, `script` (ACL name only), `acl`, `skip`, `hushlogin`
> - `user` block accepts: `password`, `profile`, `member`, `enable`, `ssh-key` etc.
> - `group` block accepts: `group`, `parent` only
> - `password pap = clear` is required in addition to `password login = clear` for PAP clients
> - Remove `ruleset` blocks entirely if causing parse errors — auth works without them

```bash
sudo tee /etc/tacplusng.cfg > /dev/null <<'EOF'
id = spawnd {
    listen {
        address = 0.0.0.0
        port = 49
    }
}

id = tac_plus-ng {
    host = vm-subnet {
        address = 10.10.10.0/24
        key = "labKey"
    }

    group = administrators {
    }

    profile = admin-profile {
        enable = clear "Admin!1783Cms10B"
    }

    user = admin {
        member = administrators
        password login = clear "Admin!1783Cms10B"
        password pap = clear "Admin!1783Cms10B"
        profile = admin-profile
    }
}
EOF
```

> **Why both `password login` and `password pap`?**
> `pam_tacplus` sends PAP authentication by default. Without `password pap`, the server
> receives the request but logs `pap login failed` even if `password login` is set.

### 5.4 Validate Configuration

```bash
/usr/local/sbin/tac_plus-ng -P /etc/tacplusng.cfg
# No output = valid config
```

### 5.5 Create Systemd Service

```bash
sudo tee /etc/systemd/system/tac_plus-ng.service > /dev/null <<'EOF'
[Unit]
Description=TACACS+ Next Generation Daemon Engine
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/tac_plus-ng -f /etc/tacplusng.cfg
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tac_plus-ng
sudo systemctl status tac_plus-ng
```

### 5.6 Open Firewall

```bash
sudo firewall-cmd --permanent --add-port=49/tcp
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

### 5.7 Verify Binding

```bash
sudo ss -tlnp | grep 49
# Expected: LISTEN 0.0.0.0:49
```

---

## 6. Phase 2: TACACS+ High Availability

### 6.1 Install Keepalived (Both Nodes)

```bash
sudo yum install keepalived -y
```

### 6.2 Configure tac1 as MASTER

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_instance VI_1 {
    state MASTER
    interface ens18
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass TacHaSecret
    }
    virtual_ipaddress {
        10.10.10.120/24
    }
}
EOF

sudo systemctl enable --now keepalived
```

### 6.3 Configure tac2 as BACKUP

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass TacHaSecret
    }
    virtual_ipaddress {
        10.10.10.120/24
    }
}
EOF

sudo systemctl enable --now keepalived
```

### 6.4 Verify VIP on tac1

```bash
ip -br address show | grep 10.10.10.120
# Expected: ens18 UP 10.10.10.125/24 10.10.10.120/24
```

<img width="1100" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/tac-keepalived.png">

---

## 7. Phase 3: HA LDAP Servers

### 7.1 ldap1 — Build OpenLDAP from Source (CentOS 9)

`openldap-servers` is not available in CentOS 9 Stream default repos. Build from source.

```bash
# Install build dependencies
sudo dnf install -y gcc make libtool openssl-devel \
  cyrus-sasl-devel groff libtool-ltdl libdb-devel

# Download and build OpenLDAP 2.6.8
cd ~
curl -O https://www.openldap.org/software/download/OpenLDAP/openldap-release/openldap-2.6.8.tgz
tar -xzf openldap-2.6.8.tgz
cd openldap-2.6.8
./configure --prefix=/usr/local --enable-slapd --enable-mdb
make depend
make
sudo make install

# Register libraries
sudo bash -c 'echo /usr/local/lib > /etc/ld.so.conf.d/openldap.conf'
sudo ldconfig
```

### 7.2 Create ldap System User and Data Directory

```bash
sudo useradd -r -s /sbin/nologin ldap
sudo mkdir -p /usr/local/var/openldap-data
sudo mkdir -p /usr/local/var/run
sudo chown -R ldap:ldap /usr/local/var/openldap-data
sudo chown ldap:ldap /usr/local/var/run
sudo chown -R ldap:ldap /usr/local/etc/openldap
```

### 7.3 Generate Root Password Hash

```bash
/usr/local/sbin/slappasswd -s 'LdapAdmin!123'
# Copy the {SSHA}... output
```

### 7.4 Configure slapd.conf

> **Critical placement rule**: `overlay syncprov` must be declared **after** the `database`
> block it applies to — not before. Placing it before causes the error:
> `overlay "syncprov" already in list` and parser failures.

```bash
sudo tee /usr/local/etc/openldap/slapd.conf > /dev/null <<'EOF'
include         /usr/local/etc/openldap/schema/core.schema
include         /usr/local/etc/openldap/schema/cosine.schema
include         /usr/local/etc/openldap/schema/nis.schema
include         /usr/local/etc/openldap/schema/inetorgperson.schema

pidfile         /usr/local/var/run/slapd.pid
argsfile        /usr/local/var/run/slapd.args

database        mdb
maxsize         1073741824
suffix          "dc=example,dc=com"
rootdn          "cn=admin,dc=example,dc=com"
rootpw          {SSHA}REPLACE_WITH_YOUR_HASH
directory       /usr/local/var/openldap-data
index           objectClass     eq

overlay syncprov
syncprov-checkpoint 100 10
syncprov-sessionlog 100
EOF
```

### 7.5 Validate and Start slapd

```bash
sudo /usr/local/sbin/slaptest -f /usr/local/etc/openldap/slapd.conf -u
# Expected: config file testing succeeded
```

```bash
sudo tee /etc/systemd/system/slapd.service > /dev/null <<'EOF'
[Unit]
Description=OpenLDAP Server
After=network.target

[Service]
Type=forking
User=ldap
Group=ldap
AmbientCapabilities=CAP_NET_BIND_SERVICE
ExecStart=/usr/local/libexec/slapd -u ldap -g ldap \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389/"
PIDFile=/usr/local/var/run/slapd.pid
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now slapd
sudo systemctl status slapd
```

> **Why `AmbientCapabilities=CAP_NET_BIND_SERVICE`?**
> Port 389 is a privileged port (below 1024). Running slapd as the `ldap` user requires
> this capability to bind to it. Without it: `bind failed errno=13 (Permission denied)`.

### 7.6 Open Firewall on ldap1

```bash
sudo firewall-cmd --permanent --add-service=ldap
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
```

### 7.7 ldap2 — Install slapd (Ubuntu 22.04.5)

```bash
sudo apt update
sudo apt install slapd ldap-utils -y

sudo dpkg-reconfigure slapd
```

During `dpkg-reconfigure slapd`:
- Omit OpenLDAP server configuration: **No**
- DNS domain name: **example.com**
- Organization name: **Example Lab**
- Admin password: **LdapAdmin!123**
- Remove database when slapd is purged: **No**
- Move old database: **Yes**

```bash
sudo systemctl status slapd
```

### 7.8 Fix Hostname Resolution on ldap2

```bash
sudo tee -a /etc/hosts > /dev/null <<'EOF'
10.10.10.102    ldap2.example.com ldap2
10.10.10.101    ldap1.example.com ldap1
EOF
```

---

## 8. Phase 4: LDAP High Availability

### 8.1 Install Keepalived

```bash
# ldap1 (CentOS 9)
sudo dnf install keepalived -y

# ldap2 (Ubuntu 22.04)
sudo apt install keepalived -y
```

### 8.2 Configure ldap1 as MASTER

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_instance LDAP_VIP {
    state MASTER
    interface ens18
    virtual_router_id 52
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LdapHaSecret
    }
    virtual_ipaddress {
        10.10.10.110/24
    }
}
EOF

sudo systemctl enable --now keepalived
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
```

### 8.3 Configure ldap2 as BACKUP

```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOF'
vrrp_instance LDAP_VIP {
    state BACKUP
    interface ens18
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LdapHaSecret
    }
    virtual_ipaddress {
        10.10.10.110/24
    }
}
EOF

sudo systemctl enable --now keepalived

# ufw does not support 'vrrp' protocol by name — allow by subnet instead
sudo ufw allow from 10.10.10.0/24
sudo ufw enable
```

### 8.4 Verify VIP

```bash
# On ldap1
ip -br address show | grep 10.10.10.110
# Expected: ens18 UP 10.10.10.101/24 10.10.10.110/24

# Test query via VIP from any node
ldapsearch -x -H ldap://10.10.10.110 \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -b "dc=example,dc=com" "(uid=admin)"
```
<img width="1100" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/ldap-HAkeepalived.png">

---

## 9. Phase 5: LDAP Data Seeding

### 9.1 Create Base Structure on ldap1

```bash
sudo tee /tmp/base.ldif > /dev/null <<'EOF'
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Lab
dc: example

dn: ou=people,dc=example,dc=com
objectClass: top
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=com
objectClass: top
objectClass: organizationalUnit
ou: groups
EOF

/usr/local/bin/ldapadd -x \
  -H ldap://localhost \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -f /tmp/base.ldif
```

### 9.2 Generate User Password Hash

```bash
/usr/local/sbin/slappasswd -s 'Admin!1783Cms10B'
# Copy the {SSHA}... output
```

### 9.3 Add Admin User and Group

```bash
sudo tee /tmp/admin-user.ldif > /dev/null <<'EOF'
dn: uid=admin,ou=people,dc=example,dc=com
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: Admin User
sn: User
uid: admin
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/admin
loginShell: /bin/bash
userPassword: {SSHA}REPLACE_WITH_YOUR_HASH

dn: cn=administrators,ou=groups,dc=example,dc=com
objectClass: top
objectClass: posixGroup
cn: administrators
gidNumber: 10001
memberUid: admin
EOF

/usr/local/bin/ldapadd -x \
  -H ldap://localhost \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -f /tmp/admin-user.ldif
```

### 9.4 Manually Seed ldap2

> **Why no automatic replication?**
> `ldap1` runs OpenLDAP 2.6.8 (source build) and `ldap2` runs 2.5.20 (Ubuntu package).
> The syncrepl control OID `1.3.6.1.4.1.4203.1.9.1.1` is not recognized cross-version,
> causing `got search entry without Sync State control`. Manual seeding is used instead.
> In production, both nodes should run identical OpenLDAP versions.

```bash
# On ldap2 — remove syncrepl if previously configured
sudo tee /tmp/del-syncrepl.ldif > /dev/null <<'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
delete: olcSyncRepl
EOF

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/del-syncrepl.ldif
sudo systemctl restart slapd

# Import data
ldapadd -x \
  -H ldap://localhost \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -f /tmp/import.ldif
```

### 9.5 Verify Both Nodes

```bash
# On ldap1
/usr/local/bin/ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -b "dc=example,dc=com" "(uid=admin)"

# On ldap2
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -b "dc=example,dc=com" "(uid=admin)"
```

<img width="1100" alt="dockerNetwork" src="https://github.com/sumanb007/System-Admin-Labs/blob/main/img/ldap-syncsuccess.png">

Both nodes now show the identical base64 value e1NTSEF9R09SYk5UdlpubmVuU3JrWDY2MlZsS2VuTXl2QmlZSnM= which decodes to {SSHA}GORbNTvZnnenSrkX662VlKenMyvBiYJs — the correct real password hash. Both LDAP nodes are in sync with the correct credentials.

What confirms they are correct:

Check ldap1 & ldap2 admin entry: exists, userPassword base64e1NTSEF9R09SYk5Udlpu...e1NTSEF9R09SYk5Udlpu...Hash matches same on both same, on bothHash is real (not placeholder).


---

## 10. Phase 6: TACACS & LDAP Server Nodes as LDAP Clients

All server nodes (tac1, tac2, ldap1, ldap2) must also authenticate via LDAP so that
centralized user management applies to them as well.

### 10.1 CentOS 9 Nodes (tac1, tac2, ldap1)

```bash
sudo dnf install -y openldap-clients nss-pam-ldapd authselect \
  sssd sssd-tools oddjob oddjob-mkhomedir

sudo tee /etc/sssd/sssd.conf > /dev/null <<'EOF'
[sssd]
services = nss, pam
config_file_version = 2
domains = example.com

[domain/example.com]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://10.10.10.110
ldap_search_base = dc=example,dc=com
ldap_default_bind_dn = cn=admin,dc=example,dc=com
ldap_default_authtok_type = password
ldap_default_authtok = LdapAdmin!123
ldap_tls_reqcert = never
ldap_id_use_start_tls = false
ldap_auth_disable_tls_never_use_in_production = true
enumerate = true
cache_credentials = true
EOF

sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl enable --now oddjobd
sudo authselect select sssd with-mkhomedir --force
sudo sss_cache -E
sudo systemctl enable --now sssd
sudo systemctl restart sssd

# Enable SSH password authentication
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Grant sudo access
echo "admin ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/ldap-admin

# Verify
sudo sssctl domain-status example.com
getent passwd admin
```

> **Why `ldap_auth_disable_tls_never_use_in_production = true`?**
> SSSD's LDAP provider defaults to requiring TLS even for plaintext `ldap://` URIs.
> Without this flag, SSSD goes `Offline` even though `ldapsearch` and `nc` can reach
> the server on port 389. This flag disables that TLS requirement for lab use.

> **Why `ldap_default_authtok_type = password`?**
> SSSD requires explicit declaration of the authtok type when using a plaintext password
> in the config. Without it, the bind credential is not correctly passed.

### 10.2 Ubuntu 22.04 Node (ldap2)

```bash
sudo apt install -y libnss-ldap libpam-ldap nscd ldap-utils

sudo tee /etc/ldap.conf > /dev/null <<'EOF'
base dc=example,dc=com
uri ldap://10.10.10.110
ldap_version 3
binddn cn=admin,dc=example,dc=com
bindpw LdapAdmin!123
pam_password md5
EOF

sudo sed -i 's/^passwd:.*/passwd:     files ldap/' /etc/nsswitch.conf
sudo sed -i 's/^group:.*/group:      files ldap/' /etc/nsswitch.conf
sudo sed -i 's/^shadow:.*/shadow:     files ldap/' /etc/nsswitch.conf

sudo systemctl restart nscd
getent passwd admin
```

---

When prompted:

LDAP server URI: ldap://10.10.10.110
Search base: dc=example,dc=com
LDAP version: 3
Make local root database admin: yes
Does LDAP database require login: no
LDAP account for root: cn=admin,dc=example,dc=com
LDAP root account password: LdapAdmin!123

## 11. Phase 7: Ubuntu Client VM Provisioning

Execute on **every new Ubuntu 24.04 client VM**.

### 11.1 Fix Hosts File

```bash
sudo tee -a /etc/hosts > /dev/null <<'EOF'
10.10.10.120    tac1.example.com tac1
10.10.10.110    ldap1.example.com ldap1
EOF
```

### 11.2 Install Build Dependencies

```bash
sudo apt update
sudo apt install -y git build-essential libpam0g-dev autoconf automake libtool \
  libcrypt-dev libc-ares-dev libssl-dev libcurl4-openssl-dev \
  libldap2-dev libpcre2-dev libsctp-dev zlib1g-dev
```

### 11.3 Build gnulib (Required by pam_tacplus)

```bash
git clone https://git.savannah.gnu.org/git/gnulib.git $HOME/gnulib
export PATH=$PATH:$HOME/gnulib
```

### 11.4 Build pam_tacplus

```bash
git clone https://github.com/kravietz/pam_tacplus.git
cd pam_tacplus
gnulib-tool --makefile-name=Makefile.gnulib --libtool --import \
  fcntl crypto/md5 array-list list xlist getrandom \
  realloc-posix explicit_bzero xalloc getopt-gnu
autoreconf -i
./configure
make
sudo make install
sudo ldconfig

# Link to Ubuntu security path
sudo mkdir -p /lib/x86_64-linux-gnu/security/
sudo ln -sf /usr/local/lib/security/pam_tacplus.so \
  /lib/x86_64-linux-gnu/security/pam_tacplus.so
```

### 11.5 Install LDAP Client Packages

```bash
sudo apt install -y libnss-ldap libpam-ldap nscd ldap-utils
```

During installation prompts:
- LDAP server URI: `ldap://10.10.10.110`
- Search base: `dc=example,dc=com`
- LDAP version: `3`
- Admin DN: `cn=admin,dc=example,dc=com`
- Admin password: `LdapAdmin!123`

### 11.6 Configure TACACS+ Client

```bash
sudo tee /etc/tacplus.conf > /dev/null <<'EOF'
secret=labKey
server=10.10.10.120
timeout=5
login=pap
EOF
```

### 11.7 Configure NSS and LDAP

```bash
sudo tee /etc/ldap.conf > /dev/null <<'EOF'
base dc=example,dc=com
uri ldap://10.10.10.110
ldap_version 3
binddn cn=admin,dc=example,dc=com
bindpw LdapAdmin!123
pam_password md5
EOF

sudo sed -i 's/^passwd:.*/passwd:     files ldap/' /etc/nsswitch.conf
sudo sed -i 's/^group:.*/group:      files ldap/' /etc/nsswitch.conf
sudo sed -i 's/^shadow:.*/shadow:     files ldap/' /etc/nsswitch.conf

sudo systemctl restart nscd
```

### 11.8 Grant Sudo Access

```bash
echo "admin ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/ldap-admin
```

---

## 12. Phase 8: PAM Stack Configuration

### 12.1 /etc/pam.d/common-auth

```
auth sufficient pam_tacplus.so config=/etc/tacplus.conf login=pap template_user=bsuman

auth [success=2 default=ignore] pam_unix.so nullok
auth [success=1 default=ignore] pam_ldap.so use_first_pass
auth requisite pam_deny.so
auth required pam_permit.so
auth optional pam_cap.so
```

### 12.2 /etc/pam.d/common-account

```
account sufficient pam_tacplus.so config=/etc/tacplus.conf

account [success=2 new_authtok_reqd=done default=ignore] pam_unix.so
account [success=1 default=ignore] pam_ldap.so
account requisite pam_deny.so
account required pam_permit.so
```

### 12.3 /etc/pam.d/common-session

Add at the bottom:
```
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

> **Why `pam_mkhomedir.so`?**
> LDAP users have no local home directory. This PAM module auto-creates `/home/<username>`
> from `/etc/skel` on first login, eliminating the need to run `useradd -m` on every VM.

> **Why `template_user=bsuman`?**
> `pam_tacplus` uses this local user as a profile template for UID/GID when the TACACS+
> user has no local entry. Ensures shell and environment are correctly inherited.

---

## 13. Phase 9: Verification & Testing

### 13.1 Test TACACS+ Authentication

```bash
tacc --authenticate \
  --username admin \
  --password 'Admin!1783Cms10B' \
  --server 10.10.10.120 \
  --secret labKey \
  --remote $(hostname -I | awk '{print $1}') \
  --service login \
  --protocol ssh \
  --tty tty1 \
  --login pap
# Expected: Authentication OK
```

### 13.2 Test LDAP User Resolution

```bash
getent passwd admin
# Expected: admin:x:10001:10001:Admin User:/home/admin:/bin/bash
```

### 13.3 Test SSH Login

```bash
ssh admin@<vm-ip>
# Password: Admin!1783Cms10B
# Home directory auto-created on first login
```

### 13.4 Verify Identity Source

```bash
id
# Expected: uid=10001(admin) gid=10001(administrators) groups=10001(administrators)
# uid=10001 confirms LDAP — local users start at 1000
```

### 13.5 Test TACACS+ HA Failover

```bash
# Stop TACACS+ master
sudo systemctl stop keepalived   # on tac1

# VIP should move to tac2
ip -br address show | grep 10.10.10.120   # on tac2

# Auth should still work
tacc --authenticate --username admin --password 'Admin!1783Cms10B' \
  --server 10.10.10.120 --secret labKey \
  --remote 10.10.10.202 --service login --protocol ssh --tty tty1 --login pap

# Restore
sudo systemctl start keepalived   # on tac1
```

### 13.6 Test LDAP HA Failover

```bash
# Stop LDAP master keepalived
sudo systemctl stop keepalived   # on ldap1

# VIP should move to ldap2
ip -br address show | grep 10.10.10.110   # on ldap2

# Query should still work
ldapsearch -x -H ldap://10.10.10.110 \
  -D "cn=admin,dc=example,dc=com" \
  -w 'LdapAdmin!123' \
  -b "dc=example,dc=com" "(uid=admin)"

# Restore
sudo systemctl start keepalived   # on ldap1
```

### 13.7 Monitor Logs

```bash
# TACACS+ auth log (on tac1)
sudo journalctl -u tac_plus-ng -f
# Expected: pap login for 'admin' ... succeeded (profile=admin-profile)

# Auth log on client VM
sudo tail -f /var/log/auth.log

# SSSD log on CentOS nodes
sudo tail -f /var/log/sssd/sssd_example.com.log
```

---

## 14. Key Lessons & Syntax Reference

### tac_plus-ng Config Rules

| Problem | Root Cause | Fix |
|---|---|---|
| `rule-1 { }` parse error | Named labels invalid in ruleset | Use `rule { }` only |
| `script = { if ... }` fails | Script expects ACL name not block | Use `script = permit` or remove |
| `service` invalid in `group` | Group only allows `group`, `parent` | Move to `profile` block |
| `service` invalid in `user` | User block has fixed valid keywords | Use `profile =` reference |
| `service` invalid in `profile` | Profile allows `enable`, `script`, `acl` etc | Use `enable = clear` |
| `script = permit` fails | Script expects ACL name not keyword | Remove ruleset entirely |
| PAP auth fails | Only `password login` defined | Add `password pap = clear` |
| Port 4949 unreachable | `tacc` defaults to port 49 | Set `port = 49` in spawnd |

### OpenLDAP Source Build Notes

| Issue | Fix |
|---|---|
| `openldap-servers` not in CentOS 9 repos | Build from source (openldap.org) |
| `bind errno=13` on port 389 | Add `AmbientCapabilities=CAP_NET_BIND_SERVICE` to systemd unit |
| `overlay syncprov already in list` | Overlay was added twice — rewrite config with single overlay block |
| Overlay not sending sync controls | `overlay syncprov` must appear **after** the `database` block |
| Cross-version syncrepl fails | OpenLDAP 2.5 and 2.6 sync controls are incompatible — use matching versions |
| `shadow context; no update referral` | `olcSyncRepl` still active — remove it before importing data |

### SSSD on CentOS Notes

| Issue | Fix |
|---|---|
| SSSD shows `Offline` despite LDAP reachable | Add `ldap_auth_disable_tls_never_use_in_production = true` |
| `su - admin` fails with auth error | SSSD was not yet online — restart after config fix |
| `oddjobd.service does not exist` | Install `oddjob oddjob-mkhomedir` packages first |
| `sssctl: command not found` | Install `sssd-tools` package |

### pam_tacplus Build Notes

| Issue | Fix |
|---|---|
| `libpam-tacplus` not in Ubuntu 24.04 repos | Build from source: github.com/kravietz/pam_tacplus |
| `autoreconf` fails with missing gnulib | Clone gnulib and run `gnulib-tool --import ...` first |
| `tacc` has no `--port` flag | Change `tac_plus-ng` to listen on port 49 |
| `error: remote address is required` | Add `--remote <client-ip>` flag |
| `error: service is required` | Add `--service login` flag |
| `error: protocol is required` | Add `--protocol ssh` flag |
| `error: tty name is required` | Add `--tty tty1` flag |
| `Transport endpoint is not connected` | TACACS+ port blocked by firewall — open port 49 |

---

## 15. Authentication Flow Diagram

```
SSH login attempt for 'admin' on any client VM
                |
                v
      PAM: pam_tacplus.so
      (config=/etc/tacplus.conf, login=pap)
      Contacts VIP: 10.10.10.120
                |
    +-----------+-----------+
    |                       |
  SUCCESS               FAILURE
    |                       |
    v                       v
 Granted           PAM: pam_ldap.so
 Home dir          (uri=ldap://10.10.10.110)
 auto-created      Contacts VIP: 10.10.10.110
                           |
               +-----------+-----------+
               |                       |
             SUCCESS               FAILURE
               |                       |
               v                       v
            Granted           PAM: pam_unix.so
            Home dir          (local /etc/shadow)
            auto-created               |
                           +-----------+-----------+
                           |                       |
                         SUCCESS               FAILURE
                           |                       |
                           v                       v
                        Granted               Denied
                        (local only)


TACACS+ HA:                    LDAP HA:
tac1 (MASTER) ─┐               ldap1 (MASTER) ─┐
               ├─► VIP 10.10.10.120             ├─► VIP 10.10.10.110
tac2 (BACKUP) ─┘               ldap2 (BACKUP) ─┘

NSS Resolution (for UID/GID/home):
  /etc/nsswitch.conf: passwd: files ldap
  getent passwd admin → uid=10001 from LDAP
  Home dir: /home/admin (auto-created by pam_mkhomedir.so)
```

---

## Secrets & Credentials Reference

| Item | Value |
|---|---|
| TACACS+ shared secret | `labKey` |
| TACACS+ HA password | `TacHaSecret` |
| Admin TACACS+ password | `Admin!1783Cms10B` |
| LDAP admin password | `LdapAdmin!123` |
| LDAP HA password | `LdapHaSecret` |
| LDAP base DN | `dc=example,dc=com` |
| LDAP admin DN | `cn=admin,dc=example,dc=com` |
| Admin UID | `10001` |
| Admin GID | `10001` |
| Admin group | `administrators` |

---

*Guide compiled from live lab session — June 2026*
