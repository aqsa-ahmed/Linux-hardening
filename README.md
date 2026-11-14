# Linux Host Hardening and Baseline Security

**Project:** DevSecOps Hardening and Pipeline Security for SkillUp Academy
**Task 1 of 3:** Linux Host Hardening and Baseline Security
**Intern:** Aqsa Zubair
**Date completed:** 14 Nov 2026

---

## 1. Overview

SkillUp Academy is an online training platform that stores student data and course materials. A default Ubuntu server is built for convenience, not security: it accepts password logins, allows direct root access, and has no protection against brute-force attacks.

In this task I hardened an Ubuntu Server virtual machine that hosts a sample web application (Nginx). The goal was to **reduce the attack surface**, which means removing every unnecessary way an attacker could get in. I did four things:

1. Updated all system packages to patch known vulnerabilities.
2. Locked down SSH so that only key-based logins are allowed and root login is disabled.
3. Enabled a UFW firewall that allows only SSH and web traffic.
4. Installed Fail2ban to automatically ban IP addresses that repeatedly fail to log in.

I then verified the final state using command output, log files, and the Lynis security auditing tool.

---

## 2. Environment

| Item | Value |
|---|---|
| Host OS | Windows |
| Hypervisor | Oracle VirtualBox |
| Guest OS | Ubuntu Server 24.04 LTS |
| VM resources | 2 CPU, 2 GB RAM, 20 GB disk |
| Network mode | NAT with port forwarding |
| Port forwarding | Host 2222 to Guest 22 (SSH), Host 8080 to Guest 80 (Web) |
| Sample application | Nginx web server |
| Tools used | SSH (OpenSSH), UFW, Fail2ban, Lynis |

---

## 3. Changes Made and Why

### 3.1 System update

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install nginx -y
```

**Why:** Outdated software contains publicly known vulnerabilities. Patching closes them before anything else is configured. Nginx was installed to act as the sample training application so that "web traffic" has a real service behind it.

### 3.2 SSH hardening

First I generated an SSH key pair on my host machine and copied the public key to the server:

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 aqsa@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

I confirmed that key login worked **before** disabling passwords, so I would not lock myself out. Then I created `/etc/ssh/sshd_config.d/00-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

Applied and verified with:

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
```

| Setting | Reason |
|---|---|
| `PermitRootLogin no` | The `root` account exists on every Linux system, so attackers always target it first. |
| `PasswordAuthentication no` | Passwords can be guessed by bots. SSH keys cannot realistically be brute-forced. |
| `PubkeyAuthentication yes` | Makes sure key-based login is the method in use. |
| `MaxAuthTries 3` | Limits the number of attempts per connection. |

The file is named `00-hardening.conf` so it is read first. SSH uses the first value it finds for a setting, and some Ubuntu files can otherwise re-enable password login.

### 3.3 Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

**Why:** Every open port is a possible entry point. The server only needs SSH (22) for administration and HTTP/HTTPS (80, 443) for web visitors. All other incoming traffic is denied by default. SSH was allowed **before** enabling the firewall to avoid cutting off my own session.

### 3.4 Fail2ban

```bash
sudo apt install fail2ban -y
sudo nano /etc/fail2ban/jail.local
```

Contents of `jail.local`:

```
[sshd]
enabled  = true
port     = ssh
maxretry = 3
findtime = 10m
bantime  = 1h
```

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

**Why:** Even with key-only SSH, bots keep hammering the login port, which fills logs and wastes resources. Fail2ban reads the authentication logs and bans any IP that fails 3 times within 10 minutes for 1 hour. Ubuntu 24.04 uses the systemd journal rather than `/var/log/auth.log`, and Fail2ban reads from it automatically.

---

## 4. Verification and Evidence

All evidence files are in the [`evidence/`](evidence/) folder.

| Check | Command | Evidence file |
|---|---|---|
| Root login disabled and password auth disabled | `sudo sshd -T \| grep -Ei 'permitrootlogin\|passwordauthentication'` | `evidence/ssh-evidence.txt` |
| Firewall active with only necessary ports | `sudo ufw status verbose` | `evidence/ufw-evidence.txt` |
| Fail2ban jail running | `sudo fail2ban-client status sshd` | `evidence/fail2ban-evidence.txt` |
| Fail2ban log entry | `sudo tail -n 30 /var/log/fail2ban.log` | `evidence/fail2ban-log.txt` |
| Service status | `sudo systemctl status ssh fail2ban ufw --no-pager` | `evidence/services.txt` |
| Security audit | `sudo lynis audit system` | `evidence/lynis-report.txt` |

### Expected results

- **SSH:** output shows `permitrootlogin no` and `passwordauthentication no`.
- **Password login test:** `ssh -p 2222 -o PubkeyAuthentication=no aqsa@127.0.0.1` returns `Permission denied (publickey)`, which proves password login is rejected.
- **UFW:** `Status: active`, default `deny (incoming)`, with rules only for 22/tcp, 80/tcp and 443/tcp.
- **Fail2ban:** the `sshd` jail is listed, and the log contains a `Ban` entry after repeated failed logins.

### Screenshots

![SSH configuration](evidence/screenshot-ssh.png)
![UFW status](evidence/screenshot-ufw.png)
![Fail2ban status and log](evidence/screenshot-fail2ban.png)
![Lynis result](evidence/screenshot-lynis.png)

### Lynis audit

- **Hardening index:** `XX / 100` *(replace XX with your score)*
- Lynis was run after all hardening steps to record a baseline. Its suggestions will be used for further improvement in later tasks.

---

## 5. Testing Fail2ban

To confirm Fail2ban actually works, I deliberately failed several SSH logins using a fake username:

```powershell
ssh -p 2222 fakeuser@127.0.0.1
```

After 3 failed attempts the source IP was banned, and the ban appeared in `/var/log/fail2ban.log`. I then removed the ban:

```bash
sudo fail2ban-client set sshd unbanip 10.0.2.2
```

---

## 6. Issues Faced and How I Solved Them

| Issue | Cause | Fix |
|---|---|---|
| Fail2ban banned my own machine during testing | With NAT networking the VM sees all connections as coming from `10.0.2.2` (the host) | Used the VirtualBox console to run `fail2ban-client set sshd unbanip 10.0.2.2` |
| `/var/log/auth.log` not found | Ubuntu 24.04 logs authentication events to the systemd journal | Fail2ban reads the journal automatically, so no change was needed |
| *(add any other problem you really hit)* | | |

---

## 7. Repository Structure

```
linux-hardening/
├── README.md
├── configs/
│   ├── 00-hardening.conf
│   └── jail.local
└── evidence/
    ├── ssh-evidence.txt
    ├── ufw-evidence.txt
    ├── fail2ban-evidence.txt
    ├── fail2ban-log.txt
    ├── services.txt
    ├── lynis-report.txt
    └── screenshot-*.png
```

---

## 8. Mentor Checklist

- [x] SSH access is restricted to key-based authentication only
- [x] UFW firewall is active with only necessary ports open
- [x] Fail2ban is installed and configured to block brute-force attempts

---

## 9. Key Takeaways

- Hardening is about reducing attack surface: fewer open doors means fewer ways in.
- Layered security matters. Keys protect login, the firewall limits exposure, and Fail2ban reacts to attacks.
- Always keep a working session open and test changes in a second window before closing the first, to avoid lockouts.
- Document every change so the configuration is repeatable and auditable.
