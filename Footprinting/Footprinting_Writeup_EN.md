# Footprinting — HTB Academy (CPTS)

**Platform:** Hack The Box Academy
**Module:** Footprinting
**Type:** Skills Assessment (3 independent labs: Easy, Medium, Hard)
**Certification:** CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Language:** English — [🇪🇸 Versión en español](./Footprinting_Writeup_ES.md)

---

## Exercise context

Fictional company **Inlanefreight Ltd** has hired us to test three different servers on their internal network. The company runs many different services, and the IT security department decided a penetration test was needed to better understand their overall security posture.

The goal is the same across all three labs: thoroughly enumerate each service, identify what information can be extracted from it, and figure out how that information can be chained together to gain access — **without actively exploiting vulnerabilities** (the services are in production). Teammates provide one shared initial hint across all three labs: the credentials `ceil:qwer1234`, and a rumor that some employees have been discussing SSH keys on a forum.

## Table of contents
1. [Easy lab — DNS/FTP/SSH server](#1-easy-lab--dnsftpssh-server)
2. [Medium lab — NFS/RDP/MSSQL server](#2-medium-lab--nfsrdpmssql-server)
3. [Hard lab — SNMP/IMAP/MySQL server](#3-hard-lab--snmpimapmysql-server)
4. [Lessons learned](#4-lessons-learned)
5. [Tool & command glossary](#5-tool--command-glossary)

---

## 1. Easy lab — DNS/FTP/SSH server

> **Scenario:** the first server is an **internal DNS server**. The client wants to know what information can be gathered from it and how it could be used against them. Administrators left a `flag.txt` file on the server to prove compromise.

### Official Skills Assessment question

| Question | Answer |
|----------|--------|
| *Enumerate the server carefully and find the flag.txt file. Submit the contents of this file as the answer.* | 🔒 Answer intentionally omitted — the full path to `cat flag.txt` is documented in §1.4; the exact flag content isn't published here so it doesn't hand the lab's answer to anyone still working through it |

### 1.1 Reconnaissance

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.177 -oN AllPorts
```

> 🛠️ **`nmap`** — the reference port/service scanner in any pentest. `-sS` (SYN scan, never completes the TCP handshake, stealthier), `-Pn` (don't skip the host even if it doesn't answer ping — very common in HTB labs, where ICMP is often filtered), `--min-rate 5000` (force a fast packet rate), `--open` (only show open ports, cuts noise), `-p-` (scan all 65535 ports), `-oN` (save output to a normal-format file, handy for attaching as evidence).

![](Imagenes/01-facil-nmap-todos-los-puertos.png)

Four open ports: **21** (ftp), **22** (ssh), **53** (domain), and **2121** (a second FTP on a non-standard port). Version and default-script scan:

```bash
nmap -sS -Pn -sCV -T5 -n -p21,22,53,2121 10.129.49.177 -oN Ports
```

> 🛠️ **`-sCV`** — combines `-sC` (runs Nmap's default NSE scripts: banners, basic config checks) with `-sV` (exact service version detection). **`-T5`** sets the most aggressive timing template, reasonable in a lab with no real IDS to worry about. **`-n`** disables reverse DNS lookups, gaining some speed.

![](Imagenes/02-facil-nmap-version-servicios.png)

| Port | Service | Detail |
|------|---------|--------|
| 21 | ProFTPD | Banner `ftp.int.inlanefreight.htb` |
| 22 | OpenSSH 8.2p1 (Ubuntu) | — |
| 53 | ISC BIND 9.16.1 (Ubuntu) | — |
| 2121 | ProFTPD | Banner **"Ceil's FTP"** |

> 💡 The second FTP service on port 2121 explicitly identifies itself as **Ceil's** FTP in its banner — a direct match for the `ceil:qwer1234` credentials the team had already shared.

### 1.2 Initial access — FTP with leaked credentials

We connect to the FTP on the non-standard port (2121) using the known credentials:

```bash
ftp 10.129.49.177 2121
Name: ceil
Password: qwer1234
```

> 🛠️ **`ftp`** — a command-line client for the **FTP** (File Transfer Protocol) protocol. After connecting, it interactively prompts for a username and password; once authenticated, it accepts protocol-specific commands (`ls`, `cd`, `get`, `put`...), distinct from the operating system's own shell commands.

![](Imagenes/03-facil-ftp-2121-login-ceil.png)

Inside `ceil`'s home there's an `.ssh` folder. We list it and download the private key:

```
ftp> cd .ssh
ftp> ls -la
ftp> get id_rsa
```

> 🛠️ **`get <file>`** (an FTP internal command) — downloads a file from the remote server to the local machine. This is how we pull out the SSH private key without needing another protocol.

![](Imagenes/04-facil-ftp-ssh-carpeta-descarga-id-rsa.png)

> 💡 This confirms the team's tip about "SSH keys": the `ceil` account has its own private key exposed via FTP, with `authorized_keys` also sitting in the same folder — it's genuinely their own access key.

### 1.3 Getting a shell

With the private key downloaded, we fix its permissions and connect via SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa ceil@10.129.49.177
```

> 🛠️ **`chmod 600`** — restricts a file's permissions to read/write for the owner only. **Mandatory** for SSH private keys: OpenSSH refuses to use a key with looser permissions ("UNPROTECTED PRIVATE KEY FILE"), precisely to stop other local users from reading it.
>
> 🛠️ **`ssh -i <key>`** — the SSH client, explicitly told which private key to use for authentication (`-i` for *identity file*), instead of relying on a password or the default keys in `~/.ssh/`.

![](Imagenes/05-facil-ssh-login-ceil-clave-privada.png)

Access confirmed as `ceil` on host **NIXEASY**.

### 1.4 Post-exploitation and the flag

```bash
cd /home
ls
```

> 🛠️ **`cd` / `ls`** — basic navigation commands: `cd` changes directory, `ls` lists its contents. The logical first move after landing a shell on any host is checking what other users/directories exist under `/home`.

![](Imagenes/06-facil-home-flag-cat-flag-txt.png)

Besides `ceil`, there are two other directories: **`cry0llt3`** (another system user) and **`flag`**. We go into the latter and read the requested file:

```bash
cd flag
cat flag.txt
```

> 🛠️ **`cat <file>`** — prints a text file's contents to the terminal. The standard command for reading flags, config files, logs, and so on.

> 🔒 As noted in this section's question table, `flag.txt`'s exact content isn't published in this writeup on purpose, so anyone using it as a guide still has to complete this last step themselves.

---

## 2. Medium lab — NFS/RDP/MSSQL server

> **Scenario:** a second server that **everyone on the internal network can reach** — a classic priority target for real-world attackers. A user named **`HTB`** was created for the assessment, and we need to recover its credentials.

### Official Skills Assessment question

| Question | Answer |
|----------|--------|
| *Enumerate the server carefully and find the username "HTB" and its password. Then, submit this user's password as the answer.* | 🔒 Answer intentionally omitted — the SQL query that returns `HTB`'s password is documented in §2.5; the exact value isn't published here so it doesn't hand the lab's answer to anyone still working through it |

### 2.1 Reconnaissance

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.180 -oN AllPorts
```

![](Imagenes/07-medio-nmap-todos-los-puertos.png)

Ports typical of a **Windows Server with Active Directory / NFS enabled**: 111 (rpcbind), 135 (msrpc), 139/445 (SMB), 2049 (NFS), 3389 (RDP), 5985/47001 (WinRM/WSMan), and a range of dynamic RPC ports.

```bash
nmap -sS -Pn -sCV -T5 -n -p111,135,139,445,2049,3389,5985,47001,49664-49668,49679-49681 10.129.49.180 -oN Ports
```

![](Imagenes/08-medio-nmap-version-servicios-smb-rdp.png)

> 💡 Nmap's `ssl-cert`/`rdp-ntlm-info` scripts already reveal the **NetBIOS name `WINMEDIUM`**, along with SMB signing "enabled but not required" — useful indicators, though not directly exploitable without being aggressive.

### 2.2 NFS enumeration

With NFS open on 2049, we run every NFS-specific NSE script Nmap ships:

```bash
nmap --script nfs* -sV -p111,2049 -oN NFS 10.129.49.180
```

> 🛠️ **`--script nfs*`** — runs every Nmap NSE script whose name starts with `nfs` (wildcard `*`): lists exported shares (`nfs-ls`), their permissions, filesystem stats, and more. A quick way to enumerate NFS without mounting it first.

![](Imagenes/09-medio-nmap-scripts-nfs.png)

The `nfs-ls` script reveals an exported share called **`/TechSupport`**, accessible with `Read Lookup` permissions and no authentication, containing multiple `ticketXXXXXXXXXXXX.txt` files, all 0 bytes.

We mount the share:

```bash
sudo mount -t nfs 10.129.49.180:/ ./lab_NFS -o nolock
ls
sudo su
cd TechSupport
ls -la
```

> 🛠️ **`mount -t nfs`** — mounts a remote NFS share at a local mount point (here, `./lab_NFS`), making it appear as just another local folder. `-o nolock` disables NFS file locking, avoiding common errors when the client's `rpc.statd` service isn't available.
>
> 🛠️ **`sudo su`** — escalates to a root shell. Classic (v3) NFS bases access control on the client's UID/GID, so being root locally is often required to read certain files on the mounted share exactly as the server sees them.

![](Imagenes/10-medio-mount-nfs-techsupport.png)
![](Imagenes/11-medio-sudo-su-cd-techsupport.png)
![](Imagenes/12-medio-ls-la-tickets-txt.png)

> 💡 Dozens of support tickets, almost all empty (0 B) — typical noise meant to make manual searching harder. We look for the one that actually has content.

Among all of them, one stands out at **1.3 KB**: `ticket4238791283782.txt`.

![](Imagenes/13-medio-ticket-con-contenido.png)

Its content is a support chat transcript between an employee (`alex`) and an operator, where Alex pastes the SMTP server's own configuration file to ask for help:

![](Imagenes/14-medio-ticket-chat-credenciales-alex.png)

```
host=smtp.web.dev.inlanefreight.htb
user="alex"
password="lol123!mD"
from="alex.g@web.dev.inlanefreight.htb"
```

> 💡 A classic human-process failure: pasting a config file with plaintext credentials into a support ticket that's reachable over NFS with no authentication at all.

We unmount the share once we've pulled the information we need:

```bash
umount -l ./lab_NFS
```

> 🛠️ **`umount -l`** — unmounts a filesystem. `-l` (*lazy unmount*) unmounts it even if it's "busy" (e.g. a terminal still has that directory as its working directory), releasing the reference as soon as it's no longer in use.

![](Imagenes/15-medio-umount-nfs.png)

### 2.3 Initial access — RDP with leaked credentials

With `alex`'s credentials (`lol123!mD`), we try direct RDP access:

```bash
xfreerdp /u:alex /p:'lol123!mD' /v:10.129.49.180
```

> 🛠️ **`xfreerdp`** — a Linux **RDP** (Remote Desktop Protocol) client. `/u:` username, `/p:` password, `/v:` target server address. Lets us validate credentials directly against the Windows remote desktop.

![](Imagenes/16-medio-xfreerdp-alex.png)

Access confirmed — a Windows 10 desktop with **SQL Server Management Studio** installed, a sign this box hosts an MSSQL database:

![](Imagenes/17-medio-escritorio-rdp-alex.png)

Browsing `alex`'s filesystem, there's a text file inside a shared folder (`devshare`):

![](Imagenes/18-medio-devshare-important-txt-sa-password.png)

```
sa:87N1ns@s11s83
```

> 💡 Credentials for MSSQL's `sa` (System Administrator) account, stored in plaintext in a file named — with zero irony — "important." Besides trying it against MSSQL, we test whether this password is also reused at the operating-system level.

### 2.4 Escalation — reusing `sa`'s credentials as Administrator

```bash
xfreerdp /u:administrator /p:'87N1ns@s11s83' /v:10.129.49.180
```

![](Imagenes/19-medio-xfreerdp-administrator-sa-password.png)

The password is reused, and we get access as **Administrator** of the local `WINMEDIUM` domain.

### 2.5 Post-exploitation — the HTB user's credentials

We open **SQL Server Management Studio** (SSMS) with Windows authentication (inheriting the Administrator context):

> 🛠️ **SQL Server Management Studio (SSMS)** — Microsoft's official GUI tool for administering SQL Server instances: connecting, browsing databases/tables, and running SQL queries directly, no command line required.

![](Imagenes/20-medio-ssms-conexion-winmedium.png)

We query the `accounts` database, table `dbo.devsacc`, filtering for the user the client asked us to find:

```sql
select * from dbo.devsacc where name = 'htb';
```

> 🛠️ **`SELECT ... WHERE ...`** — a basic SQL query statement: `SELECT *` requests every column, `FROM dbo.devsacc` names the table, and `WHERE name = 'htb'` filters down to just the row whose `name` field matches the user we're after, instead of dumping the whole table.

![](Imagenes/21-medio-ssms-consulta-devsacc-htb-password.png)

The query returns the requested row, with the **`HTB`** user and their password stored in plaintext in the table's `password` column — the proof requested by the client for this lab (see the question table at the top of this section).

---

## 3. Hard lab — SNMP/IMAP/MySQL server

> **Scenario:** the third server acts as the internal network's **MX and management server**, and also serves as a backup for domain accounts. A user named **`HTB`** was created here too, and we need to recover its credentials.

### Official Skills Assessment question

| Question | Answer |
|----------|--------|
| *Enumerate the server carefully and find the username "HTB" and its password. Then, submit HTB's password as the answer.* | 🔒 Answer intentionally omitted — the full access chain (SNMP → IMAP → SSH → MySQL) is fully documented in this section; the exact password value isn't published here so it doesn't hand the lab's answer to anyone still working through it |

### 3.1 Reconnaissance

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.192 -oN AllPorts
```

![](Imagenes/22-dificil-nmap-todos-los-puertos.png)

Only mail-related TCP services: **22** (ssh), **110/995** (pop3/pop3s), and **143/993** (imap/imaps). We follow up with a UDP scan, since a management server often exposes SNMP:

```bash
nmap -sU -T5 -F --open -oN UDP 10.129.49.192
```

> 🛠️ **`-sU`** — scans **UDP** ports (as opposed to `-sS`, which is TCP). It's inherently slower due to the protocol, which is why it's often paired with **`-F`** (*fast scan*, only the 100 most common UDP ports instead of all 65535) to keep the scan time reasonable.

![](Imagenes/23-dificil-nmap-udp-snmp.png)

Confirmed: **161/udp open (SNMP)**.

### 3.2 SNMP enumeration — community string

The default community string (`public`) doesn't work, so we brute-force it with a wordlist of common community strings:

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.49.192
```

> 🛠️ **`onesixtyone`** — a tool specialized in brute-forcing SNMP **community strings** (it's named after port 161). SNMP v1/v2c doesn't use a username/password as such — this shared string is the *only* access control — so if it's weak or default, anyone can read (and sometimes write) the device's configuration.

![](Imagenes/24-dificil-onesixtyone-community-string-backup.png)

The valid community is **`backup`**. It also confirms the hostname (**NIXHARD**) and the Linux kernel. With the community in hand, we dump every available OID:

```bash
snmpwalk -v2c -c backup 10.129.49.192
```

> 🛠️ **`snmpwalk`** — walks the entire **OID** (Object Identifier) tree exposed by an SNMP agent, dumping every value: system info, network interfaces, and — as here — often the **running process table**. `-v2c` sets the protocol version (SNMPv2c), `-c backup` the community string we just found.

![](Imagenes/25-dificil-snmpwalk-proceso-tom-password.png)

Among the output (contact info `Admin <tech@inlanefreight.htb>`, location `Inlanefreight`, etc.), the SNMP `HOST-RESOURCES-MIB` module's **running process table** shows up, including a script's **command-line arguments**:

```
iso.3.6.1.2.1.25.1.7.1.2.1.2.6.66.65.67.75.85.80 = STRING: "/opt/tom-recovery.sh"
iso.3.6.1.2.1.25.1.7.1.2.1.3.6.66.65.67.75.85.80 = STRING: "tom NMds732Js2761"
```

> 💡 A classic SNMP pitfall: when SNMP can read the system's process table (`hrSWRunParameters`), any script run with a password as a command-line argument (instead of reading it from a config file or environment variable) is exposed to anyone holding the read-only community string. Here it directly leaks credentials: `tom:NMds732Js2761`.

### 3.3 Initial access — IMAP with leaked credentials

With `tom:NMds732Js2761`, we manually connect to the IMAP service over TLS using `openssl s_client`:

```bash
openssl s_client -connect 10.129.49.192:imaps
```

> 🛠️ **`openssl s_client -connect`** — sets up a manual TLS/SSL connection to a service, useful when there's no dedicated client on hand, or when you want to interact directly with the raw protocol (here, IMAP over TLS on port 993) to understand exactly what's happening in the session.

![](Imagenes/26-dificil-openssl-sclient-imaps.png)

Inside the TLS session, we authenticate using raw IMAP and list the mailbox's folders:

```
a01 LOGIN tom NMds732Js2761
a02 LIST "" *
a03 SELECT INBOX
```

> 🛠️ **Raw IMAP commands** — IMAP is a plaintext protocol (inside the TLS tunnel): every command is prefixed with an arbitrary tag (`a01`, `a02`...) that the server echoes back in its response to identify which request it's replying to. `LOGIN` authenticates, `LIST "" *` lists every mailbox folder, `SELECT <folder>` opens it so you can operate on its messages.

![](Imagenes/27-dificil-imap-login-tom-listado-carpetas.png)

Login successful. The mailbox has folders `Notes`, `Meetings`, `Important`, and `INBOX` (with 1 message). We retrieve the message:

```
a05 FETCH 1 BODY[]
```

> 🛠️ **`FETCH <id> BODY[]`** — an IMAP command that downloads the full content (headers + body) of a specific message in the selected folder, identified by its sequence number (`1` = the first message).

![](Imagenes/28-dificil-imap-fetch-correo-clave-privada.png)

The email, subject **"KEY"**, sent from `tech@dev.inlanefreight.htb` to `tom@inlanefreight.htb`, contains a **complete SSH private key** in its body (`-----BEGIN OPENSSH PRIVATE KEY-----` block).

### 3.4 Getting a shell

```bash
chmod 600 id_rsa
ssh -i id_rsa tom@10.129.49.192
```

![](Imagenes/29-dificil-ssh-tom-clave-privada.png)

Access confirmed as `tom` on host **NIXHARD**.

### 3.5 Post-exploitation — MySQL and HTB's credentials

```bash
cat /etc/passwd
```

> 🛠️ **`/etc/passwd`** — Linux's file listing system accounts (username, UID, GID, home directory, assigned shell). It doesn't hold passwords (that's `/etc/shadow`, unreadable without privileges), but it's key to figuring out **which users have an interactive shell** versus which are just service accounts.

![](Imagenes/30-dificil-etc-passwd.png)

Besides the usual system accounts, three users have an interactive shell: `ubuntu`, **`cry0llt3`** (the same user seen in the Easy lab), and **`tom`**. The `mysql` service is also present.

MySQL is running locally, so we try reusing `tom`'s password, already known from SNMP:

```bash
mysql -u tom -p
```

> 🛠️ **`mysql -u <user> -p`** — MySQL/MariaDB's command-line client. `-u` specifies the database user, and `-p` (with no value attached) makes it prompt for the password interactively, avoiding leaving it visible in the command history.

![](Imagenes/31-dificil-mysql-login-tom.png)

The password is reused, and we get into the MySQL monitor. We list the available databases:

```sql
show databases;
use users;
show tables;
select * from users;
```

> 🛠️ **`show databases` / `use <db>` / `show tables`** — the standard MySQL exploration flow: list which databases exist, select one as the active context (`use`), and list which tables it contains. `select * from <table>` dumps every row and column of a specific table.

![](Imagenes/32-dificil-mysql-show-databases-users.png)

The `users` database, table `users`, holds a list of credentials (`id`, `username`, `password`) for multiple domain accounts. Somewhere in there should be the row for the **`HTB`** user the client asked for.

> 💡 The available screenshot shows the start of the table dump (`ppavlata0`, `ktofanini1`, `rallwell2`, `efernier3`, `fpoon4`, `jgurnell5`...). A filtered query would have isolated it directly:
>
> ```sql
> select * from users where username = 'htb';
> ```
>
> 🔒 As noted in this section's question table, `HTB`'s exact password isn't published in this writeup on purpose.

---

## 4. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| An employee's own credentials reused verbatim for their personal FTP service on a non-standard port | Easy lab | A "hidden" port on 2121 protects nothing if the credentials were already known |
| SSH private key freely accessible via FTP | Easy lab | Direct account compromise, no cracking or brute-forcing required |
| NFS share exported with no access restriction (`no_root_squash` / no authentication) | Medium lab | Any client on the network can read internal content, including support tickets with secrets |
| Service config (SMTP) with a plaintext password pasted into a support ticket | Medium lab | Credential exposure via human error, not a direct technical flaw |
| MSSQL's `sa` account password reused as the operating system's `Administrator` password | Medium lab | A single leaked credential compromises both the database and the entire server |
| SNMP with a weak, dictionary-guessable community string | Hard lab | Read access to sensitive system information without strong authentication |
| Password passed as a command-line argument to a script | Hard lab | Any mechanism that can read the process table (SNMP, `/proc`, `ps`) exposes the credential |
| Full SSH private key sent by plaintext email | Hard lab | Email isn't end-to-end encrypted by default; anyone with mailbox access gets the key |
| Password reuse across services (SNMP → MySQL) and across environments (the same `cry0llt3` user showing up in two different labs) | All three labs | A single weak or leaked credential cascades across the whole environment |

## Defensive recommendations

- Never reuse credentials across services, system accounts, and different environments.
- Restrict NFS share access by IP/range, and require authentication when the content could be sensitive.
- Don't paste configs containing secrets into support tickets, chats, or any system without strict access control.
- Never pass passwords as command-line arguments in scripts — use environment variables, tightly-permissioned config files, or a secrets manager.
- Change default SNMP community strings, and use SNMPv3 with authentication and encryption instead of SNMPv1/v2c.
- Don't send private keys by email; if unavoidable, encrypt them (GPG) and send the passphrase over a separate channel.
- Periodically audit shared resources (FTP, NFS, SMB) for forgotten or over-permissioned sensitive content.
- Apply least privilege: MSSQL's `sa` account should have no relationship whatsoever with the OS's administrative credentials.

---

## 5. Tool & command glossary

Quick reference for everything used in this module — useful for CPTS exam revision:

| Tool / command | What it's for |
|------------------|-----------------|
| `nmap` | Scan a host or network for open ports, services, and versions |
| `nmap -sU` | Scan UDP ports (slower; pair with `-F` to keep it fast) |
| `nmap --script nfs*` | Enumerate exported NFS shares without needing to mount them |
| `ftp` | Command-line client for transferring files over FTP |
| `chmod 600` | Restrict a file's permissions (mandatory for SSH private keys) |
| `ssh -i <key>` | Connect via SSH using a specific private key |
| `mount -t nfs` / `umount -l` | Mount/unmount an NFS share on the local system |
| `sudo su` | Obtain a root shell |
| `xfreerdp` | RDP client for connecting to remote Windows desktops |
| **SQL Server Management Studio (SSMS)** | Microsoft's GUI tool for administering and querying SQL Server databases |
| `SELECT ... WHERE ...` (SQL) | Query specific rows of a table instead of dumping the whole thing |
| **`onesixtyone`** | Brute-force SNMP community strings |
| **`snmpwalk`** | Dump the full OID tree exposed by an SNMP agent |
| `openssl s_client -connect` | Set up a manual TLS/SSL connection to interact with a raw protocol |
| IMAP commands (`LOGIN`, `LIST`, `SELECT`, `FETCH`) | Authenticate, list folders, and read mailbox messages over IMAP without a GUI client |
| `cat /etc/passwd` | View a Linux system's accounts and which users have an interactive shell |
| `mysql -u <user> -p` | Command-line client for connecting to MySQL/MariaDB |
| `show databases` / `use` / `show tables` (SQL) | The standard exploration flow for a MySQL instance |

---

*CPTS notes by [Arabot](https://github.com/Caan31) · HTB Academy · Footprinting module · 2026*
