# 🔴 Toppo VM — Penetration Testing Report

> **Author:** Greg Ochieng | **Date:** January 23, 2026 | **Platform:** VulnHub  
> **Severity:** ![Critical](https://img.shields.io/badge/Severity-Critical-red) **CVSS v3.1 Score:** ![9.8](https://img.shields.io/badge/CVSS-9.8-darkred)

---

## 📋 Overview

A full penetration test performed against the **Toppo** VulnHub virtual machine, documenting a complete exploitation chain from unauthenticated web enumeration to **root-level system compromise** via plaintext credential disclosure and a sudo misconfiguration.

| Field | Details |
|---|---|
| Target IP | `192.168.8.113` |
| OS | Debian GNU/Linux (kernel 3.x–4.x) |
| Services | Apache 2.4.10 · OpenSSH · RPCbind |
| Open Ports | TCP 22 (SSH) · TCP 80 (HTTP) · TCP 111 (RPCbind) |
| Tools Used | Nmap · Nikto · SSH · GTFOBins |
| CVEs | Multiple (Apache 2.4.10, Directory Indexing, Sudo Misconfiguration) |

---

## 🗺️ Attack Chain

```
[Unauthenticated]
       │
       ▼
[1] Nmap Service Scan
       │  Apache 2.4.10 on port 80 identified
       ▼
[2] Nikto Web Enumeration
       │  /admin/ directory indexing flagged
       ▼
[3] Information Disclosure → notes.txt
       │  Plaintext credentials exposed: ted:12345ted123
       ▼
[4] SSH Login as ted
       │  Password auth accepted on first attempt
       ▼
[5] Sudo Misconfiguration → AWK (GTFOBins)
       │  ted ALL=(ALL) NOPASSWD: /usr/bin/awk
       ▼
[ROOT] uid=0(root) — Flag: 0wnedlab{p4ssi0n_c0me_with_practice} ✓
```

---

## 🔍 Phase 1 — Reconnaissance & Enumeration

### Network Discovery

Target identified at `192.168.8.113` running on VirtualBox, presenting a login banner:  
*"created for the Ownedlab's event"*

![Toppo VM VirtualBox login banner showing IP address](screenshots/vm-banner.png)

### Nmap Service Scan

```bash
sudo nmap -A 192.168.8.113
```

![Nmap full scan results showing ports 22, 80, 111](screenshots/nmap-scan.png)

**Results:**
- `22/tcp` — OpenSSH 6.7p1 Debian 5+deb8u4
- `80/tcp` — Apache httpd 2.4.10 (Debian) · *"Clean Blog – Start Bootstrap Theme"*
- `111/tcp` — RPCbind (versions 2, 3, 4)

![Nmap port 80 detail showing Apache version](screenshots/nmap-port80.png)

### Web Application Enumeration (Nikto)

```bash
nikto -h http://192.168.8.113
```

![Nikto scan output highlighting /admin/ directory finding](screenshots/nikto-scan.png)

**Key Findings:**
- Apache 2.4.10 is outdated (EOL for the 2.x branch)
- Server leaks inodes via ETags headers
- `/admin/` — **Directory indexing enabled** ⚠️
- `/css/`, `/img/`, `/mail/` — Directory indexing enabled

---

## 🌐 Phase 2 — Web Application Analysis & Information Disclosure

### Step 1 — Initial Web Reconnaissance

Browsing to `http://192.168.8.113` revealed a *"Clean Blog"* themed site using the Start Bootstrap template.

![Clean Blog homepage on Toppo VM](screenshots/web-homepage.png)

### Step 2 — Administrative Directory Enumeration

Following the Nikto finding, navigating to `http://192.168.8.113/admin/` confirmed directory indexing, exposing the full directory listing.

![/admin/ directory index page showing notes.txt file](screenshots/admin-directory.png)

**Discovered file:** `notes.txt` (154 bytes, Last modified: 2018-04-15 11:16)

### Step 3 — Critical Information Disclosure

```
URL: http://192.168.8.113/admin/notes.txt
```

![notes.txt file contents in browser showing plaintext password](screenshots/notes-txt.png)

> ⚠️ **Plaintext credentials exposed in a publicly accessible file**

![Mousepad text editor showing extracted credentials](screenshots/creds-extracted.png)

**Extracted Credentials:**
- **Username:** `ted` *(inferred from password pattern)*
- **Password:** `12345ted123`

---

## 🔑 Phase 3 — Initial Access: SSH Authentication

### SSH Login

```bash
ssh ted@192.168.8.113
```

![SSH connection attempt with host key fingerprint prompt](screenshots/ssh-connect.png)

Authentication succeeded on the **first attempt**.

![Successful SSH login as ted@Toppo](screenshots/ssh-success.png)

> ✅ **Shell obtained as `ted@Toppo`**

**System Info:**
- Hostname: `Toppo`
- OS: Debian GNU/Linux
- Last login: Fri Jan 23 06:28:40 2026 from 192.168.8.112
- Working Directory: `/home/ted`

---

## ⬆️ Phase 4 — Privilege Escalation: Sudo AWK (GTFOBins)

### Step 1 — Sudo Privilege Enumeration

```bash
cat /etc/sudoers
```

![/etc/sudoers output showing ted NOPASSWD awk entry](screenshots/sudoers.png)

**Discovered entry:**
```
ted ALL=(ALL) NOPASSWD: /usr/bin/awk
```

> ⚠️ `awk` is a text processor capable of executing arbitrary shell commands — a critical misconfiguration.

### Step 2 — GTFOBins: AWK Shell Escape

Reference: [https://gtfobins.github.io/gtfobins/awk/#shell](https://gtfobins.github.io/gtfobins/awk/#shell)

![GTFOBins awk shell escape page](screenshots/gtfobins-awk.png)

AWK's `BEGIN` block executes code before processing input. Using `system()` within it allows arbitrary command execution with the privileges of the AWK process.

### Step 3 — Exploitation

```bash
mawk 'BEGIN {system("/bin/sh")}'
```

![Terminal showing mawk command spawning root shell with whoami output](screenshots/awk-privesc.png)

> ✅ **ROOT shell obtained — `uid=0(root) gid=0(root)`**

---

## 👑 Phase 5 — Confirmation of Complete System Compromise

### Root Directory Access

```bash
cd /root
ls
cat flag.txt
```

![Root directory listing and flag.txt contents](screenshots/root-access.png)

### Flag Captured

![CTF flag displayed in terminal: 0wnedlab{p4ssi0n_c0me_with_practice}](screenshots/root-flag.png)

```
0wnedlab{p4ssi0n_c0me_with_practice}
```

**Evidence of Root Access:**
- `whoami` → `root`
- Full read/write access to `/root` and all protected system files
- Privileged command execution without restriction

---

## 🛡️ Vulnerabilities Summary

| # | Vulnerability | CWE | CVSS | Severity |
|---|---|---|---|---|
| 1 | Directory Indexing Enabled (`/admin/`) | CWE-200 | 5.3 | Medium |
| 2 | Plaintext Credential Disclosure (`notes.txt`) | CWE-200 | 9.8 | Critical |
| 3 | Weak SSH Password Authentication | CWE-287 | 9.8 | Critical |
| 4 | Sudo NOPASSWD Misconfiguration (`awk`) | CWE-269 | 8.8 | High |

---

## 🔧 Remediation Recommendations

### Immediate Actions
- Delete `/admin/notes.txt` and audit all web directories for exposed sensitive files
- Disable Apache directory indexing: add `Options -Indexes` to config or `.htaccess`
- Remove the NOPASSWD sudo entry for `ted` → `/usr/bin/awk`
- Update Apache from 2.4.10 to the latest stable release (2.4.54+)
- Apply all available Debian security patches: `apt-get update && apt-get upgrade`

### Authentication Hardening
- Force immediate password reset for `ted` and all system accounts
- Enforce strong password policy (16+ chars, mixed case, numbers, symbols)
- Disable SSH password authentication: set `PasswordAuthentication no` in `/etc/ssh/sshd_config`
- Implement SSH key-based authentication for all remote access
- Disable root SSH login: `PermitRootLogin no`
- Deploy **Fail2Ban** to block IPs after repeated failed login attempts

### System-Level Controls
- Audit all `/etc/sudoers` and `/etc/sudoers.d/` entries — apply **least privilege**
- Never grant sudo access to interpreters or utilities with shell escape capability (`awk`, `sed`, `vim`, `find`, etc.)
- Hide Apache version info: set `ServerTokens Prod` and `ServerSignature Off`
- Implement HTTPS with valid SSL/TLS certificates
- Deploy a Web Application Firewall (WAF)
- Enable centralized logging and alerting for privilege escalation attempts

---

## 📚 References

| # | Resource |
|---|---|
| [1] | [Apache HTTP Server Security Tips](https://httpd.apache.org/docs/2.4/misc/security_tips.html) |
| [2] | [GTFOBins — AWK](https://gtfobins.github.io/gtfobins/awk/) |
| [3] | [OpenSSH Security Best Practices](https://www.ssh.com/academy/ssh/security) |
| [4] | [Fail2Ban Documentation](https://www.fail2ban.org/) |
| [5] | [OWASP Top 10](https://owasp.org/www-project-top-ten/) |
| [6] | [CWE-200: Exposure of Sensitive Information](https://cwe.mitre.org/data/definitions/200.html) |
| [7] | [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html) |
| [8] | [CWE-269: Improper Privilege Management](https://cwe.mitre.org/data/definitions/269.html) |
| [9] | [Nikto Web Scanner](https://cirt.net/Nikto2) |
| [10] | [NIST National Vulnerability Database](https://nvd.nist.gov/) |
| [11] | [Debian Security Information](https://www.debian.org/security/) |

---

## 📁 Repository Structure

```
toppo-vm-report/
├── README.md                              ← This file
├── Toppo_Vulnerability_Assessment_Report.pdf
└── screenshots/
    ├── vm-banner.png
    ├── nmap-scan.png
    ├── nmap-port80.png
    ├── nikto-scan.png
    ├── web-homepage.png
    ├── admin-directory.png
    ├── notes-txt.png
    ├── creds-extracted.png
    ├── ssh-connect.png
    ├── ssh-success.png
    ├── sudoers.png
    ├── gtfobins-awk.png
    ├── awk-privesc.png
    ├── root-access.png
    └── root-flag.png
```

---

> *This report was produced in a controlled lab environment for educational purposes. All testing was performed on a VulnHub machine with no real systems affected.*  
> **Author:** Greg Ochieng · Cybersecurity Student · Nex-Experience · Nairobi, Kenya
