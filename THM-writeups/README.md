# Rooms

All writeups are organized by difficulty. Each file documents the full attack chain with methodology, commands, failed attempts, and key takeaways.

---

## Easy

| Room | Category | Key Techniques | Link |
|------|----------|----------------|------|
| UnrealIRCd 3.2.8.1 Backdoor | Network / CVE | CVE-2010-2075, Backdoor RCE via Metasploit, Plaintext Credential Misconfiguration | [→ Writeup](./easy/unrealircd-3281-backdoor.md) |
| Team | Web / PrivEsc | FTP Enumeration, LFI, SSH Key in sshd_config, Sudo Command Injection, Writable Cronjob | [→ Writeup](./easy/Team-Room-Write-up.md) |
| Recruit | Web | SSRF (file://), Source Code Disclosure, SQL Injection, Credential Exposure | [→ Writeup](./easy/recruit.md) |

---

## Medium

| Room | Category | Key Techniques | Link |
|------|----------|----------------|------|
| Matryoshka | Container Security | Docker Socket Escape (chmod 666), Shared Volume Blind RCE, --privileged + --pid=host nsenter Host Escape | [→ Writeup](./medium/matryoshka.md) |
