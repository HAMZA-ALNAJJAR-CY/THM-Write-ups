# TryHackMe Writeups — Z3R0

Personal collection of TryHackMe room writeups documenting full attack chains — from recon to root. Written while building toward CPTS certification and real-world offensive security engagements.

Each writeup covers methodology, commands, reasoning behind decisions, failed attempts, and key takeaways — not just flags.

---

## Writeups

| Room | Difficulty | Category | Key Techniques |
|------|-----------|----------|----------------|
| [UnrealIRCd 3.2.8.1 Backdoor](https://github.com/HAMZA-ALNAJJAR-CY/THM-Write-ups/blob/main/THM-writeups/rooms/easy/unrealircd-3281-backdoor.md) | Easy | Network / CVE | CVE-2010-2075, Backdoor RCE, Plaintext Credential Misconfiguration |
| [Team](https://github.com/HAMZA-ALNAJJAR-CY/THM-Write-ups/blob/main/THM-writeups/rooms/easy/Team-Room-Write-up.md) | Easy | Web / PrivEsc | LFI, SSH Key Extraction, Sudo Command Injection, Writable Cronjob |
| [Recruit](https://github.com/HAMZA-ALNAJJAR-CY/THM-Write-ups/blob/main/THM-writeups/rooms/easy/recruit.md) | Easy | Web | SSRF (file://), SQL Injection, Credential Exposure |
| [Matryoshka](https://github.com/HAMZA-ALNAJJAR-CY/THM-Write-ups/blob/main/THM-writeups/rooms/medium/matryoshka.md) | Medium | Container Security | Docker Socket Escape, Shared Volume RCE, --privileged + --pid=host Host Escape |

---

## Structure

```
THM-writeups/
├── rooms/
│   ├── easy/
│   │   ├── unrealircd-3281-backdoor.md
│   │   ├── Team-Room-Write-up.md
│   │   └── recruit.md
│   └── medium/
│       └── matryoshka.md
└── README.md
```

---

**Author:** Hamza Al-Najjar (Z3R0) — Offensive Security Practitioner  
**Portfolio:** [hamza-alnajjar-cy.github.io](https://hamza-alnajjar-cy.github.io)  
**TryHackMe:** [Top 1% Globally](https://tryhackme.com/p/HAMZASAMIALNAJJAR)
