# Squid Proxy Server — Complete Knowledge Reference

> A comprehensive guide covering architecture, installation (RHEL/Ubuntu), configuration, ACLs, caching, logging, authentication, and more.

---

## Table of Contents

1. [What is Squid Proxy Server?](#what-is-squid-proxy-server)
2. [Core Concepts](#core-concepts)
3. [How Squid Works — Step by Step](#how-squid-works--step-by-step)
4. [Architecture Overview](#architecture-overview)
5. [Installation](#installation)
   - [RHEL / CentOS / Rocky Linux](#rhel--centos--rocky-linux)
   - [Ubuntu / Debian](#ubuntu--debian)
6. [Important Files and Directories](#important-files-and-directories)
7. [The Configuration File: squid.conf](#the-configuration-file-squidconf)
8. [ACL (Access Control Lists) — Deep Dive](#acl-access-control-lists--deep-dive)
9. [Squid Proxy Modes](#squid-proxy-modes)
10. [Caching — How It Actually Works](#caching--how-it-actually-works)
11. [Logging and Monitoring](#logging-and-monitoring)
12. [Authentication in Squid](#authentication-in-squid)
13. [SELinux Considerations (RHEL/CentOS)](#selinux-considerations-rhelcentos)
14. [AppArmor Considerations (Ubuntu/Debian)](#apparmor-considerations-ubuntudebian)
15. [Firewall Configuration](#firewall-configuration)
16. [Squid vs Other Proxy Solutions](#squid-vs-other-proxy-solutions)
17. [Key Terms Reference](#key-terms-reference)
18. [Useful Commands Cheatsheet](#useful-commands-cheatsheet)

---

## What is Squid Proxy Server?

Squid is a **free, open-source caching and forwarding HTTP/HTTPS proxy server** for Linux/Unix systems. It was originally developed in the 1990s, is licensed under the GPL, and remains one of the most widely deployed proxy servers in the world — used by ISPs, enterprises, governments, universities, and open-source projects.

Squid sits between your internal clients (users or systems) and the internet. Instead of every client going directly to a website, all traffic routes through Squid first. This gives you:

- **Centralized caching** — store web content locally, save bandwidth
- **Access control** — block or allow sites, users, times, file types
- **Logging and visibility** — see exactly what every client is accessing
- **Anonymity** — the internet sees Squid's IP, not the client's
- **Bandwidth savings** — cached content does not re-use internet bandwidth

---

## Core Concepts

### What is a Proxy?

A proxy is a middleman server. The traffic flow looks like this:

```
Client → Squid Proxy → Internet → Web Server
                ↑
         (cache check happens here)
```

Squid is primarily a **forward proxy** — it serves clients who want to reach the internet. This is different from a **reverse proxy** (like Nginx), which sits in front of your own servers.

### The Three Main Jobs of Squid

| Job | Description |
|-----|-------------|
| **Caching** | Stores copies of web content locally. Subsequent requests for the same content are served from disk — no internet trip needed. |
| **Access Control (ACL)** | A powerful rules engine. Block/allow by IP, domain, time, URL pattern, file type, user, and more. |
| **Logging** | Every single request passes through Squid, so everything is logged with timestamps, client IP, bytes, response codes, and cache status. |

---

## How Squid Works — Step by Step

When a client makes an HTTP/HTTPS request:

1. The client's browser or system is configured to use Squid's IP and port (default `3128`) as its proxy.
2. The request arrives at Squid.
3. Squid checks its **ACL rules** — is this request allowed or denied?
4. **If denied** → Squid immediately returns a `403 Forbidden` to the client.
5. **If allowed** → Squid checks its **cache** — does it already have this content?
6. **Cache HIT** → Squid serves the content directly from its local disk. Fast, no internet used.
7. **Cache MISS** → Squid forwards the request to the origin server on the internet, receives the response, stores a copy in cache, and delivers it to the client.

```
Client Request
     │
     ▼
 ACL Check ──── DENY ──→ 403 Forbidden
     │
    ALLOW
     │
     ▼
Cache Check ──── HIT ───→ Serve from disk (fast)
     │
    MISS
     │
     ▼
Fetch from Internet
     │
     ▼
Store in Cache → Serve to Client
```

---

## Architecture Overview

```
┌─────────────┐          ┌────────────────────────────────┐          ┌──────────────┐
│  Client 1   │          │                                │          │ Web Server A │
│  Client 2   │──────────▶      SQUID PROXY SERVER        │──────────▶ Web Server B │
│  Client 3   │          │                                │          │ Web Server C │
└─────────────┘          │  ┌──────────┐  ┌───────────┐  │          └──────────────┘
   Internal LAN          │  │ACL Engine│  │Cache Store│  │
   (port 3128)           │  └──────────┘  └───────────┘  │
                         │  ┌─────────────────────────┐   │
                         │  │  Logger (access.log)    │   │
                         │  └─────────────────────────┘   │
                         └────────────────────────────────┘
                                    │
                              ACL DENY
                                    │
                                    ▼
                             403 Forbidden
                           (blocked request)
```

**Cache HIT flow** (content already stored):
```
Client → Squid → [Cache HIT] → Client  (internet NOT contacted)
```

**Cache MISS flow** (content not yet stored):
```
Client → Squid → Internet → Web Server → Squid stores copy → Client
```

---

## Installation

### RHEL / CentOS / Rocky Linux

```bash
# RHEL 8, RHEL 9, CentOS Stream, Rocky Linux, AlmaLinux
sudo dnf install squid -y

# Start the service and enable it on boot
sudo systemctl enable --now squid

# Verify it is running
sudo systemctl status squid

# Check which port Squid is listening on
sudo ss -tlnp | grep squid
```

### Ubuntu / Debian

```bash
# Ubuntu 20.04, 22.04, 24.04 / Debian 11, 12
sudo apt update
sudo apt install squid -y

# Start the service and enable it on boot
sudo systemctl enable --now squid

# Verify it is running
sudo systemctl status squid

# Check which port Squid is listening on
sudo ss -tlnp | grep squid
```

> **Note:** On Ubuntu, the Squid service is sometimes named `squid` and sometimes `squid3` on older versions. Use `systemctl status squid` first; if that fails, try `squid3`.

---

## Important Files and Directories

### RHEL / CentOS / Rocky Linux

| Path | Purpose |
|------|---------|
| `/etc/squid/squid.conf` | Main configuration file |
| `/var/log/squid/access.log` | Every client request is logged here |
| `/var/log/squid/cache.log` | Squid daemon events and errors |
| `/var/spool/squid/` | On-disk cache storage location |
| `/usr/lib64/squid/` | Squid helper programs (auth, redirectors) |
| `/etc/squid/mime.conf` | MIME type definitions |

### Ubuntu / Debian

| Path | Purpose |
|------|---------|
| `/etc/squid/squid.conf` | Main configuration file |
| `/var/log/squid/access.log` | Every client request is logged here |
| `/var/log/squid/cache.log` | Squid daemon events and errors |
| `/var/spool/squid/` | On-disk cache storage location |
| `/usr/lib/squid/` | Squid helper programs (auth, redirectors) |
| `/etc/squid/conf.d/` | Drop-in config directory (Ubuntu splits config here) |

> **Ubuntu tip:** On Ubuntu, Squid's helper binaries live in `/usr/lib/squid/` not `/usr/lib64/squid/`. This matters when writing `auth_param` lines in your config — use the correct path or authentication will silently fail.

---

## The Configuration File: squid.conf

This is the heart of Squid. A well-commented baseline configuration:

```squid
# ============================================================
# STEP 1: Define your networks and ports using ACLs
# ============================================================

# Your local network (clients who are allowed to use this proxy)
acl localnet src 192.168.1.0/24
acl localnet src 10.0.0.0/8
acl localnet src 172.16.0.0/12

# Define safe ports (block everything else)
acl SSL_ports  port 443
acl Safe_ports port 80    # HTTP
acl Safe_ports port 443   # HTTPS
acl Safe_ports port 21    # FTP
acl Safe_ports port 70    # Gopher
acl Safe_ports port 210   # WAIS
acl Safe_ports port 1025-65535  # Unregistered ports
acl CONNECT method CONNECT

# ============================================================
# STEP 2: Access rules (ORDER MATTERS — first match wins)
# ============================================================

# Deny requests to unknown ports
http_access deny !Safe_ports

# Deny CONNECT (tunneling) to non-HTTPS ports
http_access deny CONNECT !SSL_ports

# Allow your local network
http_access allow localnet

# Allow localhost (for testing)
http_access allow localhost

# Deny everything else
http_access deny all

# ============================================================
# STEP 3: Network settings
# ============================================================

# Port Squid listens on
http_port 3128

# ============================================================
# STEP 4: Cache settings
# ============================================================

# RAM cache size
cache_mem 256 MB

# Disk cache: type=ufs, path, size=1GB, 16 L1 dirs, 256 L2 dirs
cache_dir ufs /var/spool/squid 1024 16 256

# Maximum object size to cache (in KB)
maximum_object_size 4096 KB

# Cache replacement policy
cache_replacement_policy heap LFUDA

# ============================================================
# STEP 5: Logging
# ============================================================

access_log /var/log/squid/access.log squid
cache_log  /var/log/squid/cache.log

# ============================================================
# STEP 6: Other tuning
# ============================================================

# Hide your server's identity from origin servers
via off
forwarded_for off

# DNS servers to use
dns_nameservers 8.8.8.8 1.1.1.1
```

**After any change to squid.conf:**

```bash
# Check configuration syntax before reloading
sudo squid -k parse

# Reload config without dropping connections (RHEL & Ubuntu)
sudo squid -k reconfigure

# Or use systemctl
sudo systemctl reload squid
```

> **The golden rule of ACL:** Rules are processed **top to bottom** and Squid stops at the **first match**. Order is everything. Always end with `http_access deny all`.

---

## ACL (Access Control Lists) — Deep Dive

ACLs are the most powerful part of Squid. They work in two parts:

**Part 1 — Define what you are matching:**
```
acl <name> <type> <value>
```

**Part 2 — Allow or deny based on that match:**
```
http_access allow|deny <acl_name>
```

### ACL Types Reference

| ACL Type | Matches | Example |
|----------|---------|---------|
| `src` | Source IP / network | `acl mynet src 10.0.0.0/8` |
| `dst` | Destination IP | `acl blocked_ip dst 1.2.3.4` |
| `dstdomain` | Destination domain | `acl social dstdomain .facebook.com` |
| `url_regex` | Full URL (regex) | `acl porn url_regex -i porn adult` |
| `urlpath_regex` | URL path only | `acl exe urlpath_regex -i \.exe$` |
| `port` | Destination port | `acl ssl port 443` |
| `time` | Day and time | `acl workhours time MTWHF 09:00-17:00` |
| `method` | HTTP method | `acl gets method GET` |
| `browser` | User-Agent string | `acl chrome browser Chrome` |
| `proxy_auth` | Authenticated username | `acl auth proxy_auth REQUIRED` |
| `maxconn` | Max connections per IP | `acl limit maxconn 10` |

> **Time ACL day codes:** `M`=Monday `T`=Tuesday `W`=Wednesday `H`=Thursday `F`=Friday `A`=Saturday `S`=Sunday

### Practical ACL Examples

**Block social media during work hours:**
```squid
acl workhours time MTWHF 08:00-18:00
acl social dstdomain .facebook.com .twitter.com .instagram.com .tiktok.com
http_access deny social workhours
```

**Block downloading executable files:**
```squid
acl exefiles urlpath_regex -i \.exe$ \.msi$ \.bat$ \.cmd$ \.ps1$
http_access deny exefiles
```

**Block a specific website for everyone:**
```squid
acl badsite dstdomain .gambling-site.com
http_access deny badsite
```

**Allow only specific IPs to bypass restrictions:**
```squid
acl admin_ip src 192.168.1.100 192.168.1.101
http_access allow admin_ip
```

**Block by URL keyword:**
```squid
acl keywords url_regex -i casino poker gambling torrent warez
http_access deny keywords
```

**Rate-limit connections per client:**
```squid
acl too_many_connections maxconn 20
http_access deny too_many_connections
```

**Use an external blocklist file (one domain per line):**
```squid
acl blocklist dstdomain "/etc/squid/blocked_domains.txt"
http_access deny blocklist
```

---

## Squid Proxy Modes

### 1. Standard / Explicit Proxy (Most Common)

Clients are manually configured — or auto-configured via DHCP option, Group Policy, or a PAC file — to use Squid's IP and port `3128`.

```
Browser settings → Manual proxy → 192.168.1.10 : 3128
```

No special network configuration needed. The client knowingly uses the proxy.

### 2. Transparent Proxy

Clients are unaware that a proxy exists. Network traffic on port 80 is intercepted by the router/firewall and silently redirected to Squid. Requires no client configuration.

```squid
# In squid.conf — tell Squid to operate in intercept mode
http_port 3128 intercept
```

**RHEL — redirect with firewalld:**
```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" protocol value="tcp" destination port protocol="tcp" port="80" redirect port="3128"'
sudo firewall-cmd --reload
```

**Ubuntu — redirect with iptables:**
```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128
# Make permanent
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

> **Limitation:** Transparent proxy does not work for HTTPS (port 443) without SSL Bump, because Squid cannot intercept an encrypted TLS handshake it is not part of.

### 3. SSL Bump (HTTPS Inspection)

Squid performs a controlled man-in-the-middle on HTTPS traffic. It terminates the client's TLS connection, inspects/logs/filters the content, then opens its own TLS connection to the origin server. Requires deploying a CA certificate to all client devices so they trust Squid's re-signed certificates.

```squid
# Requires Squid compiled with --enable-ssl-crtd
http_port 3128 ssl-bump \
    cert=/etc/squid/ssl_cert/squid-ca.pem

ssl_bump stare all
ssl_bump bump all

# Generate the CA
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 \
    -keyout /etc/squid/ssl_cert/squid-ca.pem \
    -out /etc/squid/ssl_cert/squid-ca.pem
```

### 4. Reverse Proxy (Less Common for Squid)

Squid sits in front of your own web servers, caching responses to accelerate and protect them. More commonly done with Nginx or Varnish, but Squid supports it.

```squid
http_port 80 accel defaultsite=your-backend-server.com
cache_peer your-backend-server.com parent 80 0 no-query originserver name=myAccel
http_access allow all
```

---

## Caching — How It Actually Works

### Cache Storage

Squid stores cached objects in `/var/spool/squid/` in a hierarchical directory structure. The `cache_dir` directive controls this:

```squid
# cache_dir <storage_type> <path> <size_MB> <L1_dirs> <L2_dirs>
cache_dir ufs /var/spool/squid 10000 16 256
```

- `ufs` — the standard Unix File System storage type
- `10000` — 10 GB of disk space allocated to cache
- `16` — 16 top-level directories
- `256` — 256 subdirectories inside each top-level directory

Squid hashes object URLs into this directory tree for fast lookup. You can have multiple `cache_dir` lines pointing to different disks for more capacity.

### Initialize the Cache Directory

```bash
# Must run this after first install or after changing cache_dir
sudo squid -z

# RHEL: may need to run as squid user
sudo -u squid squid -z
```

### Cache Replacement Policies

When the cache is full, Squid must evict old objects. The policy controls which objects get removed:

| Policy | Description | Best For |
|--------|-------------|---------|
| `heap LFUDA` | Least Frequently Used with Dynamic Aging | General purpose — recommended |
| `heap GDSF` | Greedy Dual-Size Frequency | Maximizing hit rate |
| `heap LRU` | Least Recently Used | Simple environments |

```squid
cache_replacement_policy heap LFUDA
```

### What Squid Will NOT Cache by Default

- HTTPS responses (without SSL Bump)
- Responses with `Cache-Control: no-store` or `Cache-Control: private`
- Responses to authenticated requests
- POST request responses
- Objects larger than `maximum_object_size`
- Dynamic content with certain cookie configurations

### Memory Cache

Squid also uses RAM for frequently accessed objects, which is faster than disk:

```squid
cache_mem 256 MB
maximum_object_size_in_memory 512 KB
```

---

## Logging and Monitoring

### The access.log Format

Every request through Squid is logged. A typical line looks like:

```
1234567890.123  342 192.168.1.10 TCP_MISS/200 4321 GET http://example.com/ - DIRECT/93.184.216.34 text/html
```

| Field | Value | Meaning |
|-------|-------|---------|
| Timestamp | `1234567890.123` | Unix epoch time |
| Duration | `342` | Response time in milliseconds |
| Client IP | `192.168.1.10` | Who made the request |
| Result | `TCP_MISS/200` | Cache result / HTTP status |
| Size | `4321` | Bytes transferred |
| Method | `GET` | HTTP method used |
| URL | `http://example.com/` | URL requested |
| User | `-` | Username (dash = unauthenticated) |
| Peer | `DIRECT/93.184.216.34` | How Squid fetched it |
| Type | `text/html` | Content MIME type |

### Cache Result Codes

| Code | Meaning |
|------|---------|
| `TCP_HIT` | Served fresh from disk cache |
| `TCP_MEM_HIT` | Served from memory cache (fastest) |
| `TCP_MISS` | Not cached, fetched from internet |
| `TCP_DENIED` | Blocked by ACL rule |
| `TCP_REFRESH_HIT` | Was cached but Squid revalidated with origin |
| `TCP_REFRESH_MISS` | Revalidation showed content had changed |
| `TCP_TUNNEL` | HTTPS tunnel (CONNECT method) |
| `NONE` | Error before any response |

### Useful Log Analysis Commands

```bash
# Real-time log monitoring
sudo tail -f /var/log/squid/access.log

# Top 10 most visited domains
awk '{print $7}' /var/log/squid/access.log \
  | cut -d/ -f3 | sort | uniq -c | sort -rn | head 10

# All denied requests
grep TCP_DENIED /var/log/squid/access.log

# Bandwidth used per client IP (in bytes)
awk '{print $3, $5}' /var/log/squid/access.log \
  | awk '{b[$1]+=$2} END {for(i in b) print b[i], i}' | sort -rn

# Count cache HITs vs MISSes
grep -o 'TCP_[A-Z_]*' /var/log/squid/access.log | sort | uniq -c | sort -rn

# Requests from a specific client
grep '192.168.1.50' /var/log/squid/access.log

# All HTTPS tunnel requests
grep TCP_TUNNEL /var/log/squid/access.log
```

### Log Rotation

On both RHEL and Ubuntu, Squid log rotation is handled by `logrotate`:

```bash
# Default logrotate config for Squid
cat /etc/logrotate.d/squid
```

Squid can also rotate its own logs:

```squid
# In squid.conf — rotate logs weekly, keep 10 weeks
logfile_rotate 10
```

---

## Authentication in Squid

Squid supports multiple authentication methods through external helper programs.

### Basic Authentication (Username + Password)

This uses a flat password file managed by `htpasswd`.

**On RHEL:**
```bash
# Install htpasswd tool
sudo dnf install httpd-tools -y

# Create the password file with first user
sudo htpasswd -c /etc/squid/squid_passwd alice

# Add more users (no -c flag, that would overwrite)
sudo htpasswd /etc/squid/squid_passwd bob
sudo htpasswd /etc/squid/squid_passwd carol
```

**On Ubuntu:**
```bash
# Install htpasswd tool
sudo apt install apache2-utils -y

# Create the password file with first user
sudo htpasswd -c /etc/squid/squid_passwd alice

# Add more users
sudo htpasswd /etc/squid/squid_passwd bob
```

**squid.conf for basic auth (note the different helper paths):**

```squid
# RHEL path
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_passwd

# Ubuntu path
# auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/squid_passwd

auth_param basic realm "Squid Proxy - Please Authenticate"
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive off

acl authenticated proxy_auth REQUIRED
http_access allow authenticated
http_access deny all
```

### LDAP Authentication (Active Directory Integration)

```squid
# RHEL path
auth_param basic program /usr/lib64/squid/basic_ldap_auth \
    -b "dc=company,dc=com" \
    -D "cn=squid-service,ou=ServiceAccounts,dc=company,dc=com" \
    -w "service-account-password" \
    -f "sAMAccountName=%s" \
    -h ldap.company.com

# Ubuntu path
# auth_param basic program /usr/lib/squid/basic_ldap_auth \
```

### Other Supported Authentication Methods

| Method | Use Case |
|--------|---------|
| `basic_ncsa_auth` | Simple flat file, good for small teams |
| `basic_ldap_auth` | Active Directory / OpenLDAP integration |
| `basic_pam_auth` | Linux PAM (integrates with local users) |
| `negotiate_kerberos_auth` | Kerberos / SSO — transparent to Windows clients |
| `digest_file_auth` | More secure than basic, still file-based |

---

## SELinux Considerations (RHEL/CentOS)

SELinux is enforcing by default on RHEL-family systems and is the number one reason Squid fails silently after configuration changes.

### Common SELinux Issues and Fixes

```bash
# Check for Squid-related SELinux denials in real time
sudo ausearch -m avc -ts recent | grep squid

# Or watch the audit log
sudo tail -f /var/log/audit/audit.log | grep squid

# If Squid is running on a non-default port, add the port context
sudo semanage port -a -t squid_port_t -p tcp 8080

# List all ports Squid is allowed to use
sudo semanage port -l | grep squid

# If you change the cache directory from /var/spool/squid:
sudo semanage fcontext -a -t squid_cache_t "/new/cache/path(/.*)?"
sudo restorecon -Rv /new/cache/path

# If you change the log directory:
sudo semanage fcontext -a -t squid_log_t "/new/log/path(/.*)?"
sudo restorecon -Rv /new/log/path

# Generate a custom policy from denials (use with caution)
sudo ausearch -m avc -ts recent | audit2allow -M squid_custom
sudo semodule -i squid_custom.pp
```

### SELinux Booleans for Squid

```bash
# List all Squid-related booleans
sudo getsebool -a | grep squid

# Allow Squid to connect to any port (needed for some upstream peers)
sudo setsebool -P squid_connect_any 1

# Allow Squid to use cron
sudo setsebool -P squid_use_pam_auth 1
```

---

## AppArmor Considerations (Ubuntu/Debian)

Ubuntu uses AppArmor instead of SELinux. Squid ships with an AppArmor profile that may block operations when you change default paths.

```bash
# Check if AppArmor is restricting Squid
sudo aa-status | grep squid

# View the current Squid AppArmor profile
sudo cat /etc/apparmor.d/usr.sbin.squid

# If you change cache or log paths, edit the profile
sudo nano /etc/apparmor.d/usr.sbin.squid

# Then reload the profile
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.squid

# Put Squid's AppArmor profile in complain mode (logs but doesn't block)
# Useful for debugging
sudo aa-complain /usr/sbin/squid

# Put it back in enforce mode
sudo aa-enforce /usr/sbin/squid

# Check AppArmor denials in syslog
sudo grep apparmor /var/log/syslog | grep squid
```

> **Ubuntu tip:** If Squid is silently failing to write to a custom cache or log directory, AppArmor is almost always the reason. Check `aa-status` and the syslog before spending time on squid.conf.

---

## Firewall Configuration

### RHEL / CentOS (firewalld)

```bash
# Allow Squid's default port
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports

# If Squid is used as a transparent proxy — redirect port 80
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  protocol value="tcp"
  destination port protocol="tcp" port="80"
  redirect port="3128"'
sudo firewall-cmd --reload
```

### Ubuntu / Debian (ufw)

```bash
# Allow Squid's default port
sudo ufw allow 3128/tcp

# Verify
sudo ufw status

# If you want to restrict who can reach the proxy
sudo ufw allow from 192.168.1.0/24 to any port 3128
```

### Ubuntu / Debian (iptables — transparent proxy)

```bash
# Redirect HTTP traffic to Squid (transparent mode)
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128

# Make persistent across reboots
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

## Squid vs Other Proxy Solutions

| Feature | Squid | Nginx | HAProxy | Varnish |
|---------|-------|-------|---------|---------|
| Forward proxy | ✅ Primary purpose | ⚠️ Limited | ❌ No | ❌ No |
| Reverse proxy | ✅ Supported | ✅ Excellent | ✅ Excellent | ✅ Excellent |
| HTTP caching | ✅ Excellent | ⚠️ Basic | ❌ No | ✅ Excellent |
| ACL / filtering | ✅ Very powerful | ⚠️ Basic | ⚠️ Basic | ⚠️ Basic |
| HTTPS inspection | ✅ SSL Bump | ❌ No | ❌ No | ❌ No |
| Load balancing | ⚠️ Basic | ✅ Excellent | ✅ Excellent | ⚠️ Basic |
| Authentication | ✅ Many methods | ⚠️ Basic | ⚠️ Basic | ❌ No |
| Open source | ✅ GPL | ✅ BSD | ✅ GPL | ✅ BSD |

**Use Squid when you need:** a forward proxy, content filtering, bandwidth caching, detailed access logging, or HTTPS inspection.

**Use Nginx/HAProxy when you need:** load balancing, reverse proxying, or high-performance SSL termination for your own services.

---

## Key Terms Reference

| Term | Definition |
|------|-----------|
| **Forward proxy** | Proxy that handles client requests going outbound to the internet |
| **Reverse proxy** | Proxy that sits in front of your own servers, handling inbound requests |
| **Transparent proxy** | Proxy where clients are unaware they are being proxied |
| **Cache HIT** | Requested content found in Squid's local cache — served without internet |
| **Cache MISS** | Requested content not in cache — Squid fetches from origin server |
| **ACL** | Access Control List — a named set of match criteria used in rules |
| **SSL Bump** | Squid intercepts and inspects HTTPS by acting as a trusted intermediary |
| **ICP** | Internet Cache Protocol — used by Squid instances to communicate with each other |
| **Peer** | Another Squid or cache server that this Squid instance can query or forward to |
| **Parent** | An upstream proxy that Squid forwards cache misses to |
| **Sibling** | A peer Squid that is consulted for cache HITs but misses are fetched directly |
| **Hierarchy** | A network of Squid caches working together |
| **WCCP** | Web Cache Communication Protocol — Cisco protocol for transparent proxy redirection |
| **htpasswd** | Apache utility used to create/manage basic auth password files |
| **PAC file** | Proxy Auto-Config — a JavaScript file browsers use to auto-discover proxy settings |
| **WPAD** | Web Proxy Auto-Discovery — protocol to auto-find PAC files via DNS/DHCP |

---

## Useful Commands Cheatsheet

### Service Management

```bash
# Start / stop / restart / reload
sudo systemctl start squid
sudo systemctl stop squid
sudo systemctl restart squid
sudo systemctl reload squid       # Reload config without dropping connections

# Check status
sudo systemctl status squid

# Enable / disable on boot
sudo systemctl enable squid
sudo systemctl disable squid
```

### Configuration Management

```bash
# Test configuration syntax (always do this before reload)
sudo squid -k parse

# Reload configuration gracefully
sudo squid -k reconfigure

# Rotate logs
sudo squid -k rotate

# Initialize/rebuild cache directory (run after install or cache_dir change)
sudo squid -z

# Show Squid version and build options
squid -v
```

### Cache Management

```bash
# Show cache manager statistics (requires cache manager access)
squidclient mgr:info
squidclient mgr:stats
squidclient mgr:objects

# Purge a specific URL from cache
squidclient -m PURGE http://example.com/page.html

# Show cache disk usage
sudo du -sh /var/spool/squid/
```

### Log Analysis

```bash
# Live log watch
sudo tail -f /var/log/squid/access.log

# Top accessed domains
awk '{print $7}' /var/log/squid/access.log | cut -d/ -f3 | sort | uniq -c | sort -rn | head 20

# All blocked requests
grep TCP_DENIED /var/log/squid/access.log

# Requests from one IP
grep '192.168.1.100' /var/log/squid/access.log

# Cache HIT ratio (count HITs and MISSes)
grep -c TCP_HIT /var/log/squid/access.log
grep -c TCP_MISS /var/log/squid/access.log

# Bandwidth per client
awk '{print $3, $5}' /var/log/squid/access.log \
  | awk '{b[$1]+=$2} END {for(i in b) print b[i]/1024/1024 " MB\t" i}' \
  | sort -rn | head 10
```

### Debugging

```bash
# Run Squid in the foreground with debug output (verbose)
sudo squid -NCd1

# Check what ports Squid is listening on
sudo ss -tlnp | grep squid

# Check which files Squid has open
sudo lsof -p $(pgrep squid | head -1)

# RHEL: check SELinux denials
sudo ausearch -m avc -ts recent | grep squid

# Ubuntu: check AppArmor denials
sudo grep apparmor /var/log/syslog | grep squid
```

---

## Further Reading

- [Official Squid Documentation](http://www.squid-cache.org/Doc/)
- [Squid Configuration Reference](http://www.squid-cache.org/Doc/config/)
- [Squid Wiki](https://wiki.squid-cache.org/)
- [SquidGuard](http://www.squidguard.org/) — URL redirect and blacklist filter for Squid
- [SARG](https://sourceforge.net/projects/sarg/) — Squid Analysis Report Generator (web-based log reports)

---

*Document maintained as part of the open-source infrastructure knowledge base.*
