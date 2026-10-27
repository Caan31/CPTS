# Footprinting — Cheatsheet

> 📋 Cheatsheet de referencia (comandos, tablas, metodología) — **no** es un writeup de laboratorio con capturas. Para los writeups con capturas de Skills Assessments resueltos, ver `$ ls modulos/` en el README principal.

## Contenido

- [Footprinting — Índice, cheatsheet y metodología](#footprinting--índice-cheatsheet-y-metodología)
- [Footprinting — Principios de la enumeración](#footprinting--principios-de-la-enumeración)
- **Servicios de archivos**
  - [FTP — Footprinting](#ftp--footprinting)
  - [SMB — Footprinting](#smb--footprinting)
  - [NFS — Footprinting](#nfs--footprinting)
- **Correo**
  - [SMTP — Footprinting](#smtp--footprinting)
  - [IMAP / POP3 — Footprinting](#imap--pop3--footprinting)
- **Bases de datos**
  - [MySQL — Footprinting](#mysql--footprinting)
  - [MSSQL — Footprinting](#mssql--footprinting)
  - [Oracle TNS — Footprinting](#oracle-tns--footprinting)
- **Gestión remota**
  - [Protocolos de gestión remota de Linux](#protocolos-de-gestión-remota-de-linux)
  - [Protocolos de administración remota de Windows](#protocolos-de-administración-remota-de-windows)
- **Otros servicios**
  - [SNMP — Footprinting](#snmp--footprinting)
  - [IPMI — Footprinting](#ipmi--footprinting)
  - [DNS — Footprinting](#dns--footprinting)

---

## Footprinting — Índice, cheatsheet y metodología

Nota índice del módulo: enlaces a todas las notas (agrupadas por tipo de servicio), la **metodología** y la **cheatsheet** por servicio. En todo, `<IP>` es el objetivo.

---

### Índice de notas

- **Introducción** → **Introtuction** (principios de enumeración, las 6 capas)
- **Servicios de archivos** → **FTP** · **SMB** · **NFS**
- **Correo** → **SMTP** · **IMAP POP3**
- **Bases de datos** → **MYSQL** · **MSSQL** · **Oracle TNS**
- **Gestión remota** → **Protocolos de gestión remota de Linux** · **Protocolos de Administración Remota de Windows**
- **Otros servicios** → **DNS (servicio, 53)** · **SNMP** · **IPMI**

---

### Metodología recomendada

1. **Escaneo de puertos** — rápido y amplio, luego profundo:
   ```bash
   nmap -sS -Pn -n --min-rate 5000 -p- --open -oN allports <IP>
   nmap -sS -Pn -n -sCV -p<puertos_abiertos> -oA deep <IP>
   ```
2. **No olvides UDP** (SNMP, IPMI, DNS): `sudo nmap -sU --top-ports 20 <IP>`
3. **Por cada servicio abierto → enumerar**: banner, versión (CVEs) y config.
4. **Prueba SIEMPRE las 3 puertas fáciles**: acceso anónimo/sesión nula · credenciales por defecto · misconfiguraciones.
5. **Loot**: usuarios, versiones, ficheros y, sobre todo, **credenciales**.
6. **Reutiliza credenciales** contra el resto de servicios/máquinas.
7. **Documenta** con `-oA`.

> [!TIP]
> **Reglas de oro**
> Enumera, enumera, enumera. Un servicio "aburrido" suele filtrar lo que necesitas para el siguiente. Toda credencial → pruébala en todo.

---

### Cheatsheet por servicio

| Servicio | Puerto(s) | Comando(s) clave | Qué buscar |
|----------|-----------|------------------|------------|
| **FTP** | 21 | `ftp <IP>` · `wget -m --no-passive ftp://anonymous:anonymous@<IP>` | Login anónimo |
| **SMB** | 139/445 | `smbclient -N -L //<IP>` · `crackmapexec smb <IP> -u '' -p '' --shares` · `enum4linux-ng <IP> -A` | Sesión nula, shares, usuarios |
| **NFS** | 111/2049 | `showmount -e <IP>` · `sudo mount -t nfs <IP>:/ /mnt -o nolock` | Exports, `no_root_squash` |
| **DNS** | 53 | `dig axfr <dom> @<IP>` · `dig any <dom> @<IP>` | Transferencia de zona |
| **SMTP** | 25 | `telnet <IP> 25` → `VRFY <user>` | Usuarios, open relay |
| **IMAP/POP3** | 143/993 · 110/995 | `openssl s_client -connect <IP>:imaps` | Login con creds |
| **SNMP** | 161/udp | `onesixtyone -c <wl> <IP>` · `snmpwalk -v2c -c public <IP>` | Community `public`/`private` |
| **MySQL** | 3306 | `mysql -u root -h <IP>` | `root` sin pass, `secure_file_priv` |
| **MSSQL** | 1433 | `mssqlclient.py <user>@<IP> -windows-auth` | `sysadmin`, `xp_cmdshell` |
| **Oracle** | 1521 | `nmap --script oracle-sid-brute -p1521 <IP>` · `sqlplus <u>/<p>@<IP>/<SID>` | SID, `scott/tiger`, `sys.user$` |
| **IPMI** | 623/udp | `msf ipmi_dumphashes` · `hashcat -m 7300` | Volcado de hashes (CVE-2013-4786) |
| **SSH** | 22 | `ssh <user>@<IP>` · `ssh-audit.py <IP>` | Creds débiles, config insegura |
| **Rsync** | 873 | `rsync -av --list-only rsync://<IP>/` | Módulos sin auth |
| **RDP** | 3389 | `xfreerdp /u:<u> /p:"<p>" /v:<IP>` | Creds válidas |
| **WinRM** | 5985/6 | `evil-winrm -i <IP> -u <u> -p <p>` | Shell remota |
| **WMI** | 135 | `wmiexec.py <u>:"<p>"@<IP> "whoami"` | RCE con creds |

---

### Credenciales por defecto (probar siempre)

| Servicio | Usuario : Contraseña |
|----------|----------------------|
| FTP | `anonymous:anonymous` |
| SNMP | `public` / `private` |
| MySQL | `root:` (vacía) |
| Oracle | `scott:tiger` · `dbsnmp:dbsnmp` |
| IPMI | iDRAC `root:calvin` · Supermicro `ADMIN:ADMIN` |

### Recordatorio de puertos

`21` FTP · `22` SSH · `25` SMTP · `53` DNS · `110/995` POP3 · `135` WMI · `137-139/445` SMB · `143/993` IMAP · `161/udp` SNMP · `512-514` R-services · `623/udp` IPMI · `873` Rsync · `1433` MSSQL · `1521` Oracle · `2049` NFS · `3306` MySQL · `3389` RDP · `5985/5986` WinRM

---

## Footprinting — Principios de la enumeración

La enumeración es la fase de **recopilación de información** sobre el objetivo. Se apoya en **OSINT** (Open Source Intelligence): la recopilación **pasiva** de información, que puede obtenerse de dominios, IPs, servicios accesibles y muchas otras fuentes públicas.

> [!NOTE]
> **Enfoque por capas**
> La metodología se organiza en **capas**, de lo más externo (la presencia del objetivo en Internet) a lo más interno (la configuración del sistema operativo). La idea es ir "pelando" el objetivo capa a capa hasta entenderlo lo suficiente para comunicarte con él y explotarlo.

---

### Las 6 capas de la enumeración

| Capa | Nombre                    | Objetivo                                                                                                                                                                      |
| ---- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | **Presencia en Internet** | Identificar los posibles sistemas e interfaces objetivo que se pueden probar.                                                                                                 |
| 2    | **Gateway**               | Comprender a qué nos enfrentamos y a qué debemos prestar atención (cómo se protege/enruta el objetivo).                                                                       |
| 3    | **Servicios Accesibles**  | Comprender la razón y la funcionalidad del sistema objetivo, y obtener los conocimientos necesarios para comunicarnos con él y explotarlo eficazmente.                        |
| 4    | **Procesos**              | Cada comando o función procesa datos (de usuario o del sistema). Esto inicia procesos con tareas específicas, y dichas tareas tienen al menos un **origen** y un **destino**. |
| 5    | **Privilegios**           | Identificar los privilegios disponibles y comprender **qué es y qué no es posible** con ellos.                                                                                |
| 6    | **Configuración del SO**  | Ver cómo los administradores gestionan los sistemas y qué **información interna sensible** se puede obtener de ellos.                                                         |

```mermaid
flowchart TD
    A["1 · Presencia en Internet\n(dominios, subdominios, IPs — crt.sh, Shodan)"] --> B["2 · Gateway\n(cómo está enrutado/protegido)"]
    B --> C["3 · Servicios accesibles\n(FTP, SMB, DNS, SMTP, SNMP, bases de datos...)"]
    C --> D["4 · Procesos\n(qué tarea corre cada servicio, origen→destino)"]
    D --> E["5 · Privilegios\n(qué puedo y no puedo hacer con el acceso logrado)"]
    E --> F["6 · Configuración del SO\n(ficheros de config, info interna sensible)"]
    style A fill:#e8f0fe
    style C fill:#fff4e5
    style F fill:#e6f4ea
```

> [!TIP]
> **Dónde pasas más tiempo**
> Las capas 1-2 son rápidas (recon pasivo). El **grueso del trabajo** está en la **capa 3** — es donde viven las notas de este módulo (FTP, SMB, DNS, bases de datos, gestión remota...). Las capas 4-6 ya dependen de haber conseguido acceso.

---

### Capa 1 — Presencia en Internet / Enumeración de infraestructura

El objetivo de esta capa es descubrir la superficie del objetivo expuesta en Internet: dominios, subdominios e IPs.

#### Certificate Transparency (crt.sh)

```bash
curl -s https://crt.sh/\?q\=<target-domain>\&output\=json | jq .
```
Consulta los **logs públicos de Certificate Transparency** (a través de `crt.sh`) para el dominio objetivo y formatea la salida JSON con `jq`.

> [!NOTE]
> **Por qué es útil**
> Cada certificado TLS emitido queda registrado en logs públicos de **Certificate Transparency**. Consultarlos permite **descubrir subdominios** que no aparecen en el DNS público ni están enlazados en ninguna web, ampliando la superficie de ataque de forma totalmente **pasiva**.

#### Shodan sobre una lista de IPs

```bash
for i in $(cat ip-addresses.txt); do shodan host $i; done
```
Recorre cada IP del fichero `ip-addresses.txt` y consulta su información en **Shodan** con `shodan host`.

> [!NOTE]
> **Shodan**
> **Shodan** es un buscador de dispositivos conectados a Internet. `shodan host <IP>` muestra los **puertos abiertos, servicios y banners** conocidos de esa IP sin escanearla tú directamente (la info ya está indexada por Shodan → sigue siendo pasivo).
>
> Requiere el **CLI de Shodan** instalado y una **API key** configurada (`shodan init <API_KEY>`).

---

### Recursos en la nube

Las empresas usan cada vez más almacenamiento en la nube (S3, Blobs de Azure, Google Cloud Storage…), que a veces queda expuesto. Hay muchas formas de encontrarlo; una de las más fáciles y utilizadas es la **búsqueda en Google combinada con Google Dorks**.

Por ejemplo, se pueden usar los dorks `inurl:` e `intext:` para restringir la búsqueda a términos específicos:

```text
inurl:<término>     # filtra por texto contenido en la URL
intext:<término>    # filtra por texto contenido en el cuerpo de la página
```

> [!NOTE]
> **Google Dorks**
> - `inurl:` → limita los resultados a páginas cuya **URL** contiene el término (útil para buscar rutas de buckets/almacenamiento).
> - `intext:` → limita a páginas cuyo **contenido** contiene el término (p. ej. el nombre de la empresa).
>
> Combinando ambos con el nombre de la organización se pueden localizar recursos de almacenamiento en la nube expuestos.

---

### Resumen rápido (cheatsheet)

| Comando / Dork | Qué hace |
|----------------|----------|
| `curl -s https://crt.sh/\?q\=<dominio>\&output\=json \| jq .` | Certificate Transparency: descubrir subdominios/certificados |
| `for i in $(cat ip-addresses.txt); do shodan host $i; done` | Consultar cada IP de una lista en Shodan |
| `shodan init <API_KEY>` | Configurar la API key de Shodan (paso previo) |
| `inurl:<término>` | Google Dork: filtra por texto en la URL |
| `intext:<término>` | Google Dork: filtra por texto en la página |

#### Las 6 capas de un vistazo

| Capa | Nombre |
|------|--------|
| 1 | Presencia en Internet |
| 2 | Gateway |
| 3 | Servicios Accesibles |
| 4 | Procesos |
| 5 | Privilegios |
| 6 | Configuración del SO |

---

Relacionado: **0 - Footprinting (Índice y cheatsheet)** (índice del módulo, por servicio)

---

## Servicios de archivos

### FTP — Footprinting

**FTP (File Transfer Protocol)** es un protocolo de transferencia de archivos entre cliente y servidor. Escucha por defecto en el **puerto 21/TCP** (canal de control); en modo activo usa además el **puerto 20/TCP** para los datos.

> [!NOTE]
> **Puertos**
> - **21/TCP** → canal de control (comandos).
> - **20/TCP** → canal de datos en modo activo.
> - En **modo pasivo**, el servidor abre un puerto de datos aleatorio y se lo indica al cliente (relevante para el flag `--no-passive` de `wget` más abajo).

---

#### Interacción con el servicio FTP

Distintas formas de conectarse al servicio para enumerarlo manualmente y leer su banner.

```bash
ftp <FQDN/IP>
```
Interactúa con el servicio FTP usando el **cliente `ftp` nativo**. Es la forma estándar de conectarse; abre una sesión interactiva donde puedes autenticarte y usar los comandos (`get`, `put`, `ls`…).

```bash
nc -nv <FQDN/IP> 21
```
Interactúa con el servicio FTP mediante **netcat** (`-n` sin DNS, `-v` verbose). Útil para leer el **banner** y hablar el protocolo a mano.

```bash
telnet <FQDN/IP> 21
```
Interactúa con el servicio FTP mediante **telnet**. Igual que `nc`, sirve para ver el banner e interactuar en texto plano con el canal de control.

```bash
openssl s_client -connect <FQDN/IP>:21 -starttls ftp
```
Interactúa con el servicio usando una **conexión cifrada**. `-starttls ftp` negocia TLS sobre el puerto 21 (FTPS explícito, comando `AUTH TLS`).

> [!NOTE]
> **Cuándo usar `openssl s_client`**
> Cuando el servidor obliga o soporta cifrado (`ssl_enable=YES`), `nc`/`telnet` en claro no te dejarán autenticarte. `openssl s_client -starttls ftp` establece el canal TLS y además te muestra el **certificado del servidor**, que puede filtrar información (dominios, nombres internos).

---

#### Comandos del cliente FTP interactivo

Una vez dentro de la sesión (`ftp <IP>`), estos son comandos habituales:

| Comando | Descripción |
|---------|-------------|
| `connect` | Establece el host remoto, y opcionalmente el puerto, para las transferencias de archivos. |
| `get` | Transfiere un archivo o un conjunto de archivos desde el host remoto al host local (descarga). |
| `put` | Transfiere un archivo o un conjunto de archivos desde el host local al host remoto (subida). |
| `quit` | Sale de tftp. |
| `status` | Muestra el estado actual de tftp, incluyendo el modo de transferencia actual (ascii o binario), el estado de la conexión, el valor de tiempo de espera, etc. |
| `verbose` | Activa o desactiva el modo detallado, que muestra información adicional durante la transferencia de archivos. |

> [!NOTE]
> **FTP vs TFTP**
> Estos comandos son comunes al **cliente FTP y al cliente TFTP** interactivos (por eso algunas descripciones mencionan `tftp`). El texto se mantiene tal como estaba en las notas originales.

Además, dentro de la sesión conviene conocer:

| Comando | Descripción |
|---------|-------------|
| `debug` | Activa la salida de depuración: muestra información extra del protocolo. |
| `trace` | Muestra la traza de los paquetes durante la transferencia. |

> [!TIP]
> `debug` y `trace` deben usarse **ocasionalmente**, ya que hacen que el servidor muestre más información aprovechable para la enumeración.

##### Referencia completa de comandos (cliente `ftp`/`tnftp`)

Al escribir `help` (o `?`) dentro de la sesión se listan **todos** los comandos disponibles. En la práctica se usa un subconjunto; agrupados por función:

| Categoría | Comandos | Para qué |
|-----------|----------|----------|
| **Transferir** | `get`, `put`, `mget`, `mput`, `append`, `reget` | Descargar / subir (los `m*` = varios ficheros a la vez) |
| **Modo de datos** | `binary`, `ascii`, `type` | Binario (zips, imágenes, binarios) vs texto — usa `binary` salvo para texto plano |
| **Navegar remoto** | `cd`, `cdup`, `pwd`, `ls`, `dir` | Moverte por el servidor |
| **Navegar local** | `lcd`, `lpwd` | Cambiar tu carpeta **local** de origen/destino |
| **Ficheros remotos** | `delete`, `mdelete`, `rename`, `mkdir`, `rmdir`, `chmod`, `size` | Operar sobre archivos del servidor |
| **Ejecutar en local** | `!` | `!ls`, `!cat flag.txt` → comando en **tu** máquina sin salir del prompt |
| **Sesión** | `open`, `close`, `user`, `bye` / `quit` / `exit` | Conectar / desconectar / cambiar de usuario / salir |
| **Conexión de datos** | `passive`, `epsv`, `mode` | Alternar pasivo/activo si las transferencias se cuelgan |
| **Info** | `status`, `features`, `system`, `verbose`, `debug`, `trace` | Ver estado y sacar más info del servidor |

**Avanzados útiles en pentesting:**

| Comando | Para qué |
|---------|----------|
| `quote <CMD>` | Envía un comando **crudo** del protocolo FTP (`quote SYST`, `quote PASV`, `quote USER admin`) — hablar el protocolo a mano |
| `site <CMD>` | Comandos `SITE` propios del servidor (ej. `site chmod 777 archivo`, si el server lo permite) |
| `prompt` | Apaga la confirmación `y/n` de cada fichero en `mget`/`mput` (bajar/subir en masa) |

> [!NOTE]
> El resto (`bell`, `hash`, `glob`, `sunique`, `runique`, `ntrans`, `nmap` —aquí es *filename mapping*, **no** el escáner—, `tenex`, `macdef`, `sndbuf`…) son ajustes finos o de comodidad que casi nunca tocarás. Ese set tan amplio (`mlsd`, `mlst`, `features`, `edit`, `epsv`…) es del cliente **tnftp**; `mlsd`/`mlst` dan listados en formato máquina, más fiables para parsear que `ls`.

---

#### Inicio de sesión anónimo

Muchos servidores FTP permiten el acceso **anónimo**: usuario `anonymous` y, como contraseña, cualquier valor (a menudo `anonymous` o un email). Es de lo primero que hay que probar al enumerar FTP.

```bash
ftp <FQDN/IP>
# Name: anonymous
# Password: anonymous
```

> [!WARNING]
> **fail2ban**
> Hoy en día, soluciones como [fail2ban](https://en.wikipedia.org/wiki/Fail2ban) son una implementación estándar en cualquier infraestructura: **registran la IP y bloquean todo acceso** tras cierto número de intentos de inicio de sesión fallidos. Ten cuidado con los ataques de fuerza bruta: pueden dejarte fuera (baneo de IP).

---

#### Descargar todos los archivos disponibles (wget)

```bash
wget -m --no-passive ftp://anonymous:anonymous@10.129.14.136
```
Descarga **de forma recursiva (mirror, `-m`) todos los archivos** disponibles en el servidor FTP, autenticándose como anónimo. `--no-passive` fuerza el **modo activo** (útil cuando el modo pasivo está bloqueado o mal configurado).

Una vez descargados, `wget` crea un directorio con el **nombre de la IP del objetivo** y guarda ahí todos los archivos, que puedes inspeccionar localmente.

---

#### Scripts FTP de Nmap

```bash
sudo nmap --script-updatedb
```
Actualiza la **base de datos de scripts NSE** (necesario tras añadir scripts nuevos para que Nmap los indexe).

```bash
find / -type f -name 'ftp*' 2>/dev/null | grep scripts
```
Localiza en el sistema los **scripts NSE relacionados con FTP** (`ftp-anon`, `ftp-syst`, etc.). `2>/dev/null` descarta los errores de permisos para dejar la salida limpia.

> [!NOTE]
> **Corrección de sintaxis**
> En las notas el patrón aparecía como `ftp*` sin comillas. Es recomendable **entrecomillarlo** (`'ftp*'`) para que sea `find` quien expanda el comodín y no el shell, evitando resultados inesperados si hay ficheros que empiecen por `ftp` en el directorio actual.

---

#### Configuración del servidor (vsftpd)

El fichero de configuración principal de **vsftpd** es `/etc/vsftpd.conf`. Revisarlo (si tienes acceso) revela cómo está endurecido —o no— el servicio.

```bash
cat /etc/vsftpd.conf
```

| Configuración | Descripción |
|---------------|-------------|
| `listen=NO` | ¿Ejecutar desde inetd o como un demonio independiente? |
| `listen_ipv6=YES` | ¿Escuchar en IPv6? |
| `anonymous_enable=NO` | ¿Habilitar el acceso anónimo? |
| `local_enable=YES` | ¿Permitir que los usuarios locales inicien sesión? |
| `dirmessage_enable=YES` | ¿Mostrar mensajes de directorio activo cuando los usuarios entran en ciertos directorios? |
| `use_localtime=YES` | ¿Usar la hora local? |
| `xferlog_enable=YES` | ¿Activar el registro de subidas/descargas? |
| `connect_from_port_20=YES` | ¿Conectar desde el puerto 20? |
| `secure_chroot_dir=/var/run/vsftpd/empty` | Nombre de un directorio vacío. |
| `pam_service_name=vsftpd` | Nombre del servicio PAM que vsftpd utilizará. |
| `rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem` | Ubicación del certificado RSA para las conexiones cifradas SSL. |
| `rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key` | Clave privada RSA asociada al certificado anterior. |
| `ssl_enable=NO` | ¿Habilitar el cifrado SSL/TLS? |

> [!NOTE]
> **`/etc/ftpusers`**
> El fichero `/etc/ftpusers` contiene la **lista de usuarios a los que se les DENIEGA** el acceso por FTP (típicamente `root`, `daemon`, etc.). Es una lista negra: si un usuario aparece aquí, no puede autenticarse por FTP aunque sus credenciales sean válidas.

---

#### Configuraciones peligrosas

Estas directivas, mal combinadas, convierten el FTP en un punto de entrada. Especialmente peligroso es permitir **subida/escritura anónima**.

| Configuración | Descripción |
|---------------|-------------|
| `anonymous_enable=YES` | ¿Permitir el inicio de sesión anónimo? |
| `anon_upload_enable=YES` | ¿Permitir que anónimos suban archivos? |
| `anon_mkdir_write_enable=YES` | ¿Permitir que anónimos creen nuevos directorios? |
| `no_anon_password=YES` | ¿No pedir contraseña a los anónimos? |
| `anon_root=/home/username/ftp` | Directorio raíz para anónimos. |
| `write_enable=YES` | ¿Permitir los comandos FTP `STOR`, `DELE`, `RNFR`, `RNTO`, `MKD`, `RMD`, `APPE` y `SITE`? |

> [!WARNING]
> **Por qué son peligrosas**
> Si se combina el **acceso anónimo** con la **escritura/subida** (`anon_upload_enable=YES` + `write_enable=YES`), un atacante sin credenciales puede **subir archivos** al servidor. Si ese directorio está expuesto por un servicio web, puede derivar en la subida de una **webshell** y ejecución de código. `anon_root` mal configurado también puede exponer directorios sensibles del sistema.

---

#### Configuración práctica: habilitar y usar el anónimo (laboratorio)

Notas de cuando monté un servidor vsftpd propio y quería conectarme y subir ficheros desde otra máquina.

##### Ruta del usuario anónimo

```bash
getent passwd ftp
```
El **home del usuario `ftp` es la raíz del anónimo** (lo que controla `anon_root`). Por defecto es `/srv/ftp` en Debian/Ubuntu y `/var/ftp` en CentOS/RHEL. Todo lo que se sirve al anónimo cuelga de ahí.

##### Qué activar en `/etc/vsftpd.conf` para el anónimo

```
anonymous_enable=YES         # permitir login anónimo (usuario: anonymous)
write_enable=YES             # habilitar la escritura en general
anon_upload_enable=YES       # permitir que el anónimo SUBA archivos
anon_mkdir_write_enable=YES  # (opcional) permitir que cree directorios
```

> [!WARNING]
> **Estas son las "configuraciones peligrosas"**
> Justo estas directivas son las de la sección **Configuraciones peligrosas** de más arriba: habilitar la subida anónima es un riesgo real en producción. Aquí se activan **solo para el laboratorio**.

##### Crear un directorio de subida escribible

El anónimo **no puede escribir en la raíz** (`/srv/ftp`): esa carpeta debe seguir siendo de `root` y **no** escribible, o vsftpd se niega a arrancar. Se crea una **subcarpeta** propiedad del usuario `ftp`:

```bash
sudo mkdir /srv/ftp/upload
```
```bash
sudo chown ftp:ftp /srv/ftp/upload
```

##### Aplicar los cambios

```bash
sudo systemctl restart vsftpd
```

##### Probar desde el cliente (otra máquina)

```bash
ftp <IP-del-servidor>
# Name: anonymous
# Password: anonymous
```

Y dentro de la sesión, subir **dentro de la subcarpeta**:

```bash
cd upload
binary
put flag.txt
```

> [!WARNING]
> **Por qué el `put` daba `550 Permission denied`**
> Me pasó estando como anónimo. Dos causas, en este orden:
> 1. Faltaba `anon_upload_enable=YES` (o `write_enable=YES`) → vsftpd rechaza el `STOR` con **550**.
> 2. Aun activándolos, hacer `put` en la **raíz** falla igual: hay que subir **dentro de la subcarpeta** (`cd upload`), porque la raíz no puede ser escribible.
>
> Regla para distinguir: `550 Permission denied` = lo bloquea **vsftpd** (una directiva). `553 Could not create file` = lo bloquean los **permisos del directorio** (owner/`chmod`).

> [!NOTE]
> **Si aún no puedes subir**
> - **SELinux** (CentOS/RHEL): `sudo setsebool -P ftpd_full_access on`.
> - Confirma que escribes en la subcarpeta correcta y que es `ftp:ftp` con permiso de escritura: `ls -ld /srv/ftp/upload`.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `ftp <FQDN/IP>` | Conectar con el cliente FTP nativo |
| `nc -nv <FQDN/IP> 21` | Interactuar / leer banner con netcat |
| `telnet <FQDN/IP> 21` | Interactuar / leer banner con telnet |
| `openssl s_client -connect <FQDN/IP>:21 -starttls ftp` | Conexión **cifrada** (FTPS explícito) |
| Login `anonymous` / `anonymous` | Probar acceso anónimo |
| `wget -m --no-passive ftp://anonymous:anonymous@<IP>` | Descargar **todos** los archivos (mirror, modo activo) |
| `get` / `put` | Descargar / subir archivos dentro de la sesión |
| `debug` / `trace` | Sacar más información del servidor |
| `sudo nmap --script-updatedb` | Actualizar la base de datos de scripts NSE |
| `find / -type f -name 'ftp*' 2>/dev/null \| grep scripts` | Localizar scripts NSE de FTP |
| `/etc/vsftpd.conf` | Config principal de vsftpd |
| `/etc/ftpusers` | Usuarios a los que se **deniega** FTP (lista negra) |
| `anon_upload_enable` + `write_enable` | Config peligrosa: subida anónima de archivos |

---

Relacionado: otros servicios de archivos → **SMB** · **NFS** · **0 - Footprinting (Índice y cheatsheet)**

---

### SMB — Footprinting

**SMB (Server Message Block)** es un protocolo cliente-servidor que regula el acceso a archivos, directorios y otros recursos de red (impresoras, routers…). Los derechos de acceso se definen mediante **listas de control de acceso (ACL)**.

---

#### SMB, CIFS y Samba

**Samba** implementa el protocolo **CIFS (Common Internet File System)**, lo que le permite comunicarse con sistemas **Windows**. CIFS es un dialecto concreto de SMB.

**Versiones:**
- **SMBv1** → obsoleta.
- **SMB2** y **SMB3** → versiones actuales.

> [!WARNING]
> **SMBv1**
> SMBv1 está obsoleta y es **insegura** (vulnerabilidades históricas como EternalBlue se apoyan en ella). Si la encuentras habilitada en un objetivo, es un hallazgo relevante. En producción debería usarse SMB2/SMB3.

---

#### Puertos

| Puerto | Protocolo | Servicio |
|--------|-----------|----------|
| **137** | UDP | NetBIOS Name Service (resolución de nombres NetBIOS) |
| **138** | UDP | NetBIOS Datagram Service |
| **139** | TCP | NetBIOS Session Service — **SMB sobre NetBIOS** (heredado) |
| **445** | TCP | **SMB directo sobre TCP** (CIFS) — el moderno |

> [!NOTE]
> Los puertos **137-139** son el SMB "antiguo" encapsulado en NetBIOS; el **445** es SMB directamente sobre TCP. En objetivos actuales lo normal es ver el **445** abierto.

---

#### Daemons y configuración

El servidor SMB (Samba) lo forman dos daemons:
- **`smbd`** → provee la compartición de archivos e impresión (el servicio SMB en sí).
- **`nmbd`** → provee el servicio de nombres NetBIOS (resolución de nombres, anuncios en la red).

Fichero de configuración por defecto:

```bash
cat /etc/samba/smb.conf
```

---

#### Configuración por defecto (smb.conf)

| Configuración | Descripción |
|---------------|-------------|
| `[sharename]` | El nombre del recurso compartido de red. |
| `workgroup = WORKGROUP/DOMAIN` | Grupo de trabajo que aparecerá cuando los clientes consulten. |
| `path = /path/here/` | El directorio al que se le va a dar acceso al usuario. |
| `server string = STRING` | La cadena que aparecerá cuando se inicie una conexión. |
| `unix password sync = yes` | ¿Sincronizar la contraseña de UNIX con la contraseña de SMB? |
| `usershare allow guests = yes` | ¿Permitir a los usuarios no autenticados acceder al recurso compartido definido? |
| `map to guest = bad user` | ¿Qué hacer cuando una solicitud de inicio de sesión de un usuario no coincide con un usuario válido de UNIX? |
| `browseable = yes` | ¿Debería mostrarse este recurso compartido en la lista de recursos compartidos disponibles? |
| `guest ok = yes` | ¿Permitir la conexión al servicio sin usar una contraseña? |
| `read only = yes` | ¿Permitir a los usuarios solo leer archivos? |
| `create mask = 0700` | ¿Qué permisos deben establecerse para los archivos recién creados? |

---

#### Ajustes peligrosos

| Configuración | Descripción |
|---------------|-------------|
| `browseable = yes` | ¿Permitir la enumeración de los recursos compartidos disponibles en el recurso compartido actual? |
| `read only = no` | ¿Prohibir la creación y modificación de archivos? |
| `writable = yes` | ¿Permitir a los usuarios crear y modificar archivos? |
| `guest ok = yes` | ¿Permitir la conexión al servicio sin usar una contraseña? |
| `enable privileges = yes` | ¿Respetar los privilegios asignados a un SID específico? |
| `create mask = 0777` | ¿Qué permisos se deben asignar a los archivos recién creados? |
| `directory mask = 0777` | ¿Qué permisos se deben asignar a los directorios recién creados? |
| `logon script = script.sh` | ¿Qué script debe ejecutarse en el inicio de sesión del usuario? |
| `magic script = script.sh` | ¿Qué script debe ejecutarse cuando el script se cierra? |
| `magic output = script.out` | ¿Dónde debe almacenarse la salida del script mágico? |

> [!WARNING]
> **Por qué son peligrosas**
> - Combinar `guest ok = yes` con `writable = yes` / `read only = no` permite a **cualquiera (anónimo) escribir** en el recurso compartido.
> - `create mask`/`directory mask = 0777` deja los ficheros/directorios nuevos con permisos totales para todos.
> - `logon script` y `magic script` permiten **ejecutar scripts**: si puedes controlar ese script, puede derivar en **ejecución de código** en el servidor.

---

#### Reiniciar Samba

```bash
sudo systemctl restart smbd
```
Tras modificar `/etc/samba/smb.conf` hay que reiniciar el servicio para aplicar los cambios.

---

#### Enumeración desde el cliente

##### `smbclient` — Listar recursos compartidos (sesión nula)

```bash
smbclient -N -L //10.129.14.128
```
Lista (`-L`) los recursos compartidos del servidor. La opción `-N` usa una **sesión nula** (*null session*): un acceso **anónimo**, sin usuario ni contraseña válidos.

> [!NOTE]
> **Sesión nula**
> La sesión nula es lo primero que se prueba en SMB. En sistemas modernos o endurecidos suele estar **deshabilitada**, pero cuando funciona puede filtrar shares, usuarios y políticas sin credenciales.

##### `smbclient` — Conectar a un recurso compartido concreto

```bash
smbclient //<FQDN/IP>/<share>
```
Se conecta a un **share específico** (el que hayas descubierto con `-L`) y abre una sesión interactiva para navegarlo (`ls`, `get`, `put`, `cd`…).

> [!TIP]
> Dentro de la sesión de `smbclient` puedes ejecutar **comandos del sistema local** anteponiendo un signo de exclamación (`!<cmd>`, ej. `!ls`) **sin interrumpir la conexión**.

##### `smbstatus` — Ver conexiones (lado administrativo)

```bash
smbstatus
```
Desde el punto de vista administrativo, muestra las conexiones activas. Aparte de la **versión de Samba**, revela **quién**, desde **qué host** y a **qué recurso compartido** está conectado cada cliente.

---

#### rpcclient — Funciones MS-RPC

`rpcclient` es una herramienta para realizar **funciones MS-RPC** contra el servidor SMB. Ofrece muchas consultas con las que ejecutar funciones específicas para obtener información (lista completa en la [página del manual](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html)).

```bash
rpcclient -U "" <FQDN/IP>
```
Se conecta al objetivo por **RPC** usando un **usuario nulo** (`-U ""`), es decir, sesión anónima. Una vez dentro, se lanzan las consultas de la tabla.

| Consulta                  | Descripción                                                                         |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `srvinfo`                 | Información del servidor.                                                           |
| `enumdomains`             | Enumera todos los dominios que están desplegados en la red.                         |
| `querydominfo`            | Proporciona información de dominio, servidor y usuario de los dominios desplegados. |
| `netshareenumall`         | Enumera todos los recursos compartidos disponibles.                                 |
| `netsharegetinfo <share>` | Proporciona información sobre un recurso compartido específico.                     |
| `enumdomusers`            | Enumera todos los usuarios del dominio.                                             |
| `queryuser <RID>`         | Proporciona información sobre un usuario específico.                                |

**Ejemplo de enumeración de usuarios** (salida de mis notas):

```
rpcclient> enumdomusers
user:[mrb3n] rid:[0x3e8]
user:[cry0l1t3] rid:[0x3e9]

rpcclient> queryuser 0x3e9
rpcclient> querygroup
```

> [!NOTE]
> **RID**
> El **RID** (Relative Identifier) es la parte final del **SID** de una cuenta. `enumdomusers` te da el RID de cada usuario (ej. `0x3e9` para `cry0l1t3`), y ese mismo RID es el que pasas a `queryuser <RID>` para sacar los detalles de esa cuenta.

##### samrdump.py (Impacket)

```bash
samrdump.py <FQDN/IP>
```
**Enumeración de usuarios** usando los scripts de **Impacket** (interroga el servicio SAMR por RPC). Script de referencia:
<https://github.com/fortra/impacket/blob/master/examples/samrdump.py>

---

#### Enumeración automatizada

##### smbmap

```bash
smbmap -H 10.129.14.128
```
Enumera los **recursos compartidos** de SMB y, muy útil, **los permisos** (lectura/escritura) que tienes sobre cada uno.

##### crackmapexec

```bash
crackmapexec smb 10.129.14.128 --shares -u '' -p ''
crackmapexec smb 10.129.14.128 -u /diccionario -p /dicionario
```
Enumera los shares de SMB usando **sesión nula** (`-u '' -p ''`, usuario y contraseña vacíos).

> [!NOTE]
> **crackmapexec → NetExec**
> `crackmapexec` (CME) está descontinuado; su sucesor mantenido es **NetExec (`nxc`)**, con la misma sintaxis (`nxc smb <IP> --shares -u '' -p ''`). Si `crackmapexec` no está en tu sistema, prueba `nxc`.

##### enum4linux

```bash
enum4linux 10.129.14.128 -A
```
Enumeración **completa** (`-A`, *do all*) de un objetivo SMB: usuarios, shares, grupos, políticas, etc.

```bash
enum4linux-ng.py <FQDN/IP> -A
```
**`enum4linux-ng`** es la reescritura moderna y mantenida de `enum4linux` (salida más limpia y en JSON/YAML). Es la que aparece en la cheatsheet de HTB.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `smbclient -N -L //<FQDN/IP>` | Listar shares con **sesión nula** (anónimo) |
| `smbclient //<FQDN/IP>/<share>` | Conectar a un **share concreto** |
| `!<cmd>` (dentro de smbclient) | Ejecutar un comando local sin cortar la conexión |
| `smbstatus` | Ver conexiones activas (lado servidor) |
| `rpcclient -U "" <FQDN/IP>` | Conexión **RPC** con usuario nulo |
| `enumdomusers` / `queryuser <RID>` | (en rpcclient) enumerar usuarios / detalle por RID |
| `netshareenumall` | (en rpcclient) enumerar todos los shares |
| `samrdump.py <FQDN/IP>` | Enumerar usuarios con **Impacket** |
| `smbmap -H <FQDN/IP>` | Enumerar shares y **permisos** |
| `crackmapexec smb <FQDN/IP> --shares -u '' -p ''` | Enumerar shares con sesión nula (hoy: `nxc`) |
| `enum4linux -A <FQDN/IP>` / `enum4linux-ng.py <FQDN/IP> -A` | Enumeración SMB completa |
| `sudo systemctl restart smbd` | Reiniciar el servidor Samba |
| `/etc/samba/smb.conf` | Fichero de configuración de Samba |

---

Relacionado: otros servicios de archivos → **FTP** · **NFS** · mismo entorno Windows → **Protocolos de Administración Remota de Windows** · fuerza bruta/creds → **Servicios en red** · **0 - Footprinting (Índice y cheatsheet)**

---

### NFS — Footprinting

**NFS (Network File System)** es un sistema de archivos en red: permite montar directorios de un servidor remoto como si fueran locales. Es el equivalente Unix/Linux a lo que SMB es en el mundo Windows.

---

#### Configuración por defecto (/etc/exports)

El fichero `/etc/exports` define qué directorios se exportan y con qué opciones.

```bash
cat /etc/exports
```

| Opción | Descripción |
|--------|-------------|
| `rw` | Permisos de lectura y escritura. |
| `ro` | Permisos de solo lectura. |
| `sync` | Transferencia de datos síncrona (un poco más lento). |
| `async` | Transferencia de datos asíncrona (un poco más rápido). |
| `secure` | No se usarán puertos por encima de 1024. |
| `insecure` | Se usarán puertos por encima de 1024. |
| `no_subtree_check` | Deshabilita la comprobación de árboles de subdirectorios. |
| `root_squash` | Asigna los permisos de los archivos del UID/GID 0 (`root`) al UID/GID anónimo, lo que **impide que `root` acceda** a los archivos en un montaje NFS. |

> [!WARNING]
> **`root_squash` vs `no_root_squash`**
> - **`root_squash`** (por defecto y seguro): el `root` del cliente se "aplasta" a un usuario anónimo (`nobody`) en el servidor.
> - **`no_root_squash`** (peligroso): el `root` del cliente **se mantiene como `root`** en el servidor → es una **vía de escalada de privilegios** (ver sección final).

---

#### Reconocimiento del servicio

NFS se apoya en **RPC**, así que al enumerarlo interesan los puertos **TCP 111 y 2049**:

| Puerto | Servicio |
|--------|----------|
| **111** | `rpcbind` / portmapper (localiza los servicios RPC) |
| **2049** | `nfs` (el servicio NFS en sí) |

```bash
sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049
```
Lanza todos los **scripts NSE de NFS** (`nfs-ls`, `nfs-showmount`, `nfs-statfs`) junto a la detección de versión (`-sV`) sobre los puertos 111 y 2049.

En la salida, fíjate en la línea de **permisos de acceso**:

```
| access: Read Lookup NoModify NoExtend NoDelete NoExecute
```

> [!NOTE]
> **La línea `access:`**
> Indica **qué puedes hacer** sobre ese export sin montarlo: aquí `Read`/`Lookup` (leer y listar) pero `NoModify`/`NoDelete`/… (no escribir). Si vieras permisos de escritura, es un buen indicio para la fase de explotación.

---

#### Enumerar recursos compartidos (showmount)

```bash
showmount -e <FQDN/IP>
```
Muestra (`-e`, *exports*) la lista de **recursos NFS exportados** por el servidor y qué clientes/redes tienen permitido montarlos. Es la forma rápida de ver qué hay antes de montar nada.

> [!NOTE]
> `showmount` viene en el paquete **`nfs-common`** (Debian/Ubuntu). Si no lo tienes: `sudo apt install nfs-common`.

---

#### Montar un recurso NFS

```bash
mkdir target-NFS
```
```bash
sudo mount -t nfs 10.129.14.128:/ ./target-NFS/ -o nolock
```
```bash
cd target-NFS
tree .
```
Monta el export del servidor (`10.129.14.128:/`) en el directorio local `./target-NFS/`. `-o nolock` **desactiva el bloqueo de ficheros** (evita problemas con el servicio de locking). Con `tree .` inspeccionas todo el contenido montado.

---

#### Desmontaje

```bash
cd ..
sudo umount -l ./target-NFS
```
Sal del directorio montado antes de desmontar (si estás dentro, `umount` fallará con "target is busy").

---

#### root_squash / no_root_squash y escritura

Si montas el recurso y **tienes permisos de escritura**, puedes **crear un archivo** en el montaje y comprobar desde la máquina víctima que aparece ahí (ambos lados ven el mismo directorio).

> [!WARNING]
> **Escalada de privilegios con `no_root_squash`**
> Si el export tiene **`no_root_squash`** y creas un fichero como **root desde tu máquina**, ese fichero **conserva permisos de root** en el servidor. El abuso clásico:
> 1. Como root en tu máquina, copias un binario (ej. una shell) al montaje y le pones **bit SUID** (`chmod +s`).
> 2. En la máquina víctima, ese binario es propiedad de `root` con SUID → al ejecutarlo obtienes una shell **root**.
>
> Es una de las escaladas de privilegios más habituales cuando encuentras `no_root_squash`.

---

#### Resumen rápido (cheatsheet)

| Comando                                                        | Qué hace                                               |
| -------------------------------------------------------------- | ------------------------------------------------------ |
| `sudo nmap --script "nfs*" <IP> -sV -p111,2049`                | Enumerar NFS con scripts NSE                           |
| `showmount -e <FQDN/IP>`                                       | Listar los recursos NFS exportados                     |
| `mkdir target-NFS`                                             | Crear el punto de montaje local                        |
| `sudo mount -t nfs <FQDN/IP>:/<share> ./target-NFS/ -o nolock` | Montar el recurso NFS                                  |
| `tree .`                                                       | Inspeccionar el contenido montado                      |
| `sudo umount ./target-NFS`                                     | Desmontar (sal del directorio primero)                 |
| `/etc/exports`                                                 | Config de exports del servidor                         |
| `no_root_squash`                                               | Config peligrosa → escalada de privilegios (SUID root) |

---

Relacionado: otros servicios de archivos → **FTP** · **SMB** · **0 - Footprinting (Índice y cheatsheet)**

---

## Correo

### SMTP — Footprinting

**SMTP (Simple Mail Transfer Protocol)** es el protocolo para **enviar** correos en una red. Es cliente-servidor y se combina con **IMAP** o **POP3**, que son los que se encargan de **recibir/leer** el correo.

---

#### Puertos

| Puerto | Uso |
|--------|-----|
| **25** | Puerto SMTP estándar (envío/retransmisión entre servidores). |
| **587** | *Submission*: envío desde clientes **autenticados**; usa **STARTTLS** para pasar la conexión de texto plano a cifrada. |
| **465** | SMTP sobre **TLS** implícito (SMTPS), conexión cifrada desde el inicio. |

> [!NOTE]
> `STARTTLS` no es un puerto sino un comando: negocia el cifrado **sobre una conexión ya abierta** (típicamente en el 587). El 465 en cambio arranca ya cifrado.

---

#### Cómo viaja un correo (agentes)

Una de las funciones de SMTP es la **prevención de spam**. El recorrido de un correo pasa por varios agentes:

- **MUA** (*Mail User Agent*) → el **cliente**. Tras redactar el correo, lo convierte en **encabezado + cuerpo** y lo sube al servidor SMTP.
- **MTA** (*Mail Transfer Agent*) → la base de software para **enviar y recibir** correos. Comprueba el tamaño del correo, entre otras cosas.
- **MSA** (*Mail Submission Agent*) → recibe el correo del cliente para **aliviar la carga del MTA**.
- **MDA** (*Mail Delivery Agent*) → en el servidor de destino, una vez reensamblado el correo, lo **entrega al buzón** del destinatario.

| Cliente (`MUA`) | ➞ | Agente de envío (`MSA`) | ➞ | Retransmisión abierta (`MTA`) | ➞ | Agente de entrega (`MDA`) | ➞ | Buzón de correo (`POP3`/`IMAP`) |
|---|---|---|---|---|---|---|---|---|

> [!NOTE]
> **Sin confirmación y ESMTP**
> - SMTP **no devuelve confirmación** de entrega; solo un mensaje (en inglés) si hubo un error con el encabezado.
> - Originalmente los usuarios **no se autenticaban** al conectar, lo que permitía **spam masivo**. Para mitigarlo se creó la extensión **ESMTP (Extended SMTP)**, que añade autenticación y más capacidades (se activa con `EHLO` en vez de `HELO`).

---

#### Configuración por defecto

```bash
cat /etc/postfix/main.cf
```
Fichero de configuración principal de **Postfix** (uno de los MTA más comunes en Linux).

---

#### Interacción con telnet

##### Conectar al servicio

```bash
telnet <FQDN/IP> 25
```
Abre una conexión TCP en claro con el servidor SMTP para **hablar el protocolo a mano** y leer el banner.

##### HELO / EHLO — iniciar la sesión

```bash
HELO mail1.inlanefreight.htb
```
La sesión SMTP se inicia con **`HELO`** (básico) o **`EHLO`** (extendido, ESMTP — lista además las capacidades del servidor). Es el primer comando tras conectar.

##### VRFY — enumerar usuarios

```bash
VRFY root
```
El comando **`VRFY`** se usa para **enumerar usuarios** existentes en el sistema. No siempre funciona.

> [!NOTE]
> **Código 252**
> Según cómo esté configurado el servidor, `VRFY` puede devolver el **código 252** y "confirmar" un usuario que **no existe** realmente. Es decir, un 252 no garantiza que el usuario sea válido → los resultados de `VRFY` hay que tomarlos con cautela.

##### Enviar un correo

```bash
EHLO inlanefreight.htb
```
Tras el `EHLO` se especifica el asunto, destinatario, CC, BCC y el contenido. La secuencia estándar del protocolo para enviar un correo a mano es:

```
MAIL FROM: <remitente@dominio.htb>
RCPT TO: <destinatario@dominio.htb>
DATA
Subject: asunto del correo

Cuerpo del mensaje.
.
```

> [!NOTE]
> **Añadido de referencia**
> Esa secuencia (`MAIL FROM` → `RCPT TO` → `DATA` → línea con un único `.` para terminar) es el flujo estándar de SMTP; no estaba en las notas originales, se añade como referencia. El punto solo en una línea marca el **fin del mensaje**.

---

#### Configuraciones peligrosas

##### Retransmisión abierta (open relay)

```
mynetworks = 0.0.0.0/0
```

Con esta configuración, el servidor SMTP acepta retransmitir correo **desde cualquier origen**, por lo que puede usarse para **enviar correos falsos** e iniciar comunicación entre múltiples partes. Otra vía de ataque es **suplantar (spoofing)** el correo.

> [!WARNING]
> **Open relay**
> Un *open relay* es un servidor que reenvía correo de **cualquiera hacia cualquiera** sin autenticación. Es un fallo grave: se abusa para **spam** y **spoofing** de remitentes. Se comprueba con el script `smtp-open-relay` (abajo).

---

#### Reconocimiento del servicio (Nmap)

Los scripts por defecto de Nmap incluyen **`smtp-commands`**, que usa `EHLO` para **listar todos los comandos** que se pueden ejecutar en el servidor SMTP objetivo.

También se puede usar el script NSE [`smtp-open-relay`](https://nmap.org/nsedoc/scripts/smtp-open-relay.html) para comprobar si el servidor es un **open relay**.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `telnet <FQDN/IP> 25` | Conectar al servicio SMTP a mano |
| `HELO` / `EHLO <dominio>` | Iniciar la sesión (`EHLO` = extendido, lista capacidades) |
| `VRFY <usuario>` | Enumerar usuarios (ojo al **código 252**) |
| `MAIL FROM:` / `RCPT TO:` / `DATA` / `.` | Secuencia para enviar un correo a mano |
| `cat /etc/postfix/main.cf` | Config del MTA (Postfix) |
| `mynetworks = 0.0.0.0/0` | Config peligrosa → **open relay** |
| Nmap `smtp-commands` | Listar comandos del servidor |
| Nmap `smtp-open-relay` | Comprobar si es open relay |
| Puertos | 25 (estándar) · 587 (submission/STARTTLS) · 465 (SMTPS) |

---

Relacionado: recibir el correo → **IMAP POP3** · registros MX → **DNS** · **0 - Footprinting (Índice y cheatsheet)**

---

### IMAP / POP3 — Footprinting

- **IMAP** permite la **gestión online** de los correos directamente en el servidor y admite **estructuras de carpetas**. Es cliente-servidor y, sin cifrar, transmite los comandos en texto plano (ASCII).
- **POP3** lista, recupera y elimina correos en el servidor de correo (modelo más simple, sin carpetas).

Ambos son protocolos para **recibir/leer** correo; el envío lo hace SMTP.

---

#### SMTP vs IMAP/POP3 (push vs pull)

- **SMTP** (puertos 25/587/465) → se usa para **enviar** correo y para el **transporte entre servidores** (cuando el servidor de Gmail le pasa un correo al de Outlook, usan SMTP entre ellos). Es de **"empuje" (push)**: tu cliente empuja el mensaje hacia el servidor, y los servidores se lo empujan entre sí.
- **IMAP** (143/993) y **POP3** (110/995) → se usan para que **tú, como cliente, recuperes** los correos ya guardados en el servidor. Son de **"tirar" (pull)**: tu cliente (Outlook, Thunderbird, el móvil) le pregunta al servidor "¿qué correos tienes para mí?".

---

#### Puertos

| Servicio | Puerto en claro | Puerto cifrado (TLS/SSL) |
|----------|-----------------|--------------------------|
| **IMAP** | 143 | **993** (IMAPS) |
| **POP3** | 110 | **995** (POP3S) |

> [!NOTE]
> Los puertos **altos (993/995)** usan **TLS implícito** (cifrado desde el inicio de la conexión). Los de texto plano (143/110) pueden también cifrarse **bajo demanda** con `STARTTLS`.

---

#### Configuraciones peligrosas (Dovecot)

| Configuración | Descripción |
|---------------|-------------|
| `auth_debug` | Habilita todos los registros de depuración de autenticación. |
| `auth_debug_passwords` | Ajusta la verbosidad de los registros: se registran las contraseñas enviadas y el esquema. |
| `auth_verbose` | Registra los intentos de autenticación fallidos y sus motivos. |
| `auth_verbose_passwords` | Las contraseñas usadas para autenticar se registran (y pueden truncarse). |
| `auth_anonymous_username` | Especifica el nombre de usuario al iniciar sesión con el mecanismo **ANONYMOUS SASL**. |

> [!WARNING]
> **Por qué son peligrosas**
> Estas directivas de **Dovecot** hacen que se **registren contraseñas en los logs** (`auth_debug_passwords`, `auth_verbose_passwords`). Si consigues leer esos logs, tienes **credenciales en claro**. `auth_anonymous_username` habilita además el **login anónimo**.

---

#### Interacción con el servicio

> [!NOTE]
> Si consigues las credenciales de acceso de un empleado, puedes **iniciar sesión** en el servidor de correo para **leer** (o incluso enviar) sus mensajes.

##### cURL — Login en IMAPS

```bash
curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd
```
Inicia sesión en el servicio **IMAPS** (IMAP sobre TLS) con `curl`. `-k` **ignora la validación del certificado** (útil con certificados autofirmados de laboratorio). Añade `-v` para ver el diálogo completo.

##### OpenSSL — Interacción cifrada TLS con POP3

```bash
openssl s_client -connect 10.129.14.128:pop3s
```
Abre una conexión **TLS** al servicio **POP3S** (995) para hablar el protocolo a mano de forma cifrada.

##### OpenSSL — Interacción cifrada TLS con IMAP

```bash
openssl s_client -connect 10.129.14.128:imaps
```
Igual pero contra **IMAPS** (993).

---

#### Comandos IMAP

> [!NOTE]
> **Los "tags" (a1, a2…)**
> En IMAP, **cada comando se antepone con una etiqueta (tag)** que tú eliges: `a1`, `a2`, `a3`… (o `1`, `2`…). Sirve para que el cliente **case cada respuesta con su petición**. Por eso en una sesión real escribes `a1 LOGIN …`, `a2 LIST …`, etc., incrementando el tag en cada comando.

| Comando | Descripción |
|---------|-------------|
| `1 LOGIN username password` | Inicio de sesión del usuario. |
| `1 LIST "" *` | Lista todos los directorios. |
| `1 CREATE "INBOX"` | Crea un buzón con un nombre específico. |
| `1 DELETE "INBOX"` | Elimina un buzón. |
| `1 RENAME "ToRead" "Important"` | Renombra un buzón. |
| `1 LSUB "" *` | Devuelve los nombres que el usuario ha declarado como `activos` o `suscritos`. |
| `1 SELECT INBOX` | Selecciona un buzón para poder acceder a sus mensajes. |
| `1 UNSELECT INBOX` | Sale del buzón seleccionado. |
| `1 FETCH <ID> all` | Recupera los datos asociados a un mensaje del buzón. |
| `1 CLOSE` | Elimina todos los mensajes con el indicador `Deleted` establecido. |
| `1 LOGOUT` | Cierra la conexión con el servidor IMAP. |

> [!TIP]
> **Leer un mensaje con FETCH**
> Tras `SELECT INBOX`, el servidor indica cuántos mensajes hay (ej. `* 1 EXISTS` → hay 1 mensaje). Para leer su cuerpo hay que usar el **número de secuencia real**, no la letra ni el `UIDVALIDITY`. Como solo hay 1 mensaje, su número de secuencia es `1`:
> ```
> a6 FETCH 1 BODY[]
> ```

---

#### Comandos POP3

| Comando | Descripción |
|---------|-------------|
| `USER username` | Identifica al usuario. |
| `PASS password` | Autentica al usuario con su contraseña. |
| `STAT` | Pide al servidor el número de correos guardados. |
| `LIST` | Pide el número y tamaño de todos los correos. |
| `RETR id` | Pide al servidor que entregue el correo indicado por ID. |
| `DELE id` | Pide al servidor que elimine el correo indicado por ID. |
| `CAPA` | Pide al servidor que muestre sus capacidades. |
| `RSET` | Pide al servidor que restablezca la información transmitida. |
| `QUIT` | Cierra la conexión con el servidor POP3. |

> [!NOTE]
> A diferencia de IMAP, **POP3 no usa tags**: los comandos se envían directamente (`USER`, `PASS`, `RETR 1`…).

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `curl -k 'imaps://<FQDN/IP>' --user <user>:<pass>` | Login en IMAPS con cURL |
| `openssl s_client -connect <FQDN/IP>:imaps` | Conexión TLS a IMAPS (993) |
| `openssl s_client -connect <FQDN/IP>:pop3s` | Conexión TLS a POP3S (995) |
| IMAP: `a1 LOGIN user pass` → `a2 LIST "" *` → `a3 SELECT INBOX` → `a4 FETCH 1 BODY[]` | Flujo típico para leer correo |
| POP3: `USER user` → `PASS pass` → `LIST` → `RETR 1` | Flujo típico para leer correo |
| Puertos | IMAP 143 / 993 (TLS) · POP3 110 / 995 (TLS) |
| Dovecot `auth_*_passwords` | Config peligrosa → contraseñas en los logs |

---

Relacionado: enviar correo → **SMTP** · **0 - Footprinting (Índice y cheatsheet)**

---

## Bases de datos

### MySQL — Footprinting

**MySQL** es un sistema de gestión de bases de datos relacional (RDBMS), **propiedad de Oracle**, que funciona en modelo **cliente-servidor**.

- Escucha por defecto en el **puerto TCP 3306**.

> [!NOTE]
> **MySQL / MariaDB**
> **MariaDB** es el *fork* comunitario de MySQL. El cliente `mysql` sirve para conectarse a ambos, así que estos comandos valen igual.

---

#### Configuración por defecto

```bash
cat /etc/mysql/mysql.conf.d/mysqld.cnf
```
Fichero de configuración del daemon MySQL (`mysqld`). Revisarlo (si tienes acceso local) muestra usuario de ejecución, direcciones de escucha y opciones de seguridad.

---

#### Configuraciones peligrosas

| Ajuste | Descripción |
|--------|-------------|
| `user` | Establece con qué usuario del sistema se ejecutará el servicio MySQL. |
| `password` | Establece la contraseña del usuario de MySQL. |
| `admin_address` | La IP en la que se escucharán las conexiones TCP/IP en la interfaz de red **administrativa**. |
| `debug` | Indica la configuración de depuración actual. |
| `sql_warnings` | Controla si las sentencias `INSERT` de una sola fila producen una cadena de información cuando hay advertencias. |
| `secure_file_priv` | Limita el efecto de las operaciones de **importación/exportación** de datos (qué directorios pueden leerse/escribirse). |

> [!WARNING]
> **`secure_file_priv` (clave para pentest)**
> Esta variable controla desde/hacia qué rutas puede MySQL **leer y escribir ficheros** con `LOAD_FILE()` e `INTO OUTFILE`/`INTO DUMPFILE`.
> - Si está **vacía** → MySQL puede leer/escribir en **cualquier ruta** (peligroso): permite **leer ficheros del sistema** (ej. `/etc/passwd`, credenciales) o **escribir una webshell** en el directorio web.
> - Si apunta a un directorio concreto → limita esas operaciones a esa ruta.
> Compruébala con: `SHOW VARIABLES LIKE 'secure_file_priv';`

---

#### Conexión al servidor

```bash
mysql -u root -h 10.129.14.132
```
Conexión sin contraseña (o para que la pida). `-u` = usuario, `-h` = host.

```bash
mysql -u root -pP4SSw0rd -h 10.129.14.128
```
Conexión indicando la contraseña en línea.

> [!WARNING]
> **La contraseña va PEGADA a `-p`**
> **No** debe haber espacio entre `-p` y la contraseña: `-pP4SSw0rd` (correcto) vs `-p P4SSw0rd` (incorrecto — MySQL interpretaría `P4SSw0rd` como el nombre de la base de datos). Si pones **`-p` solo** (sin valor), MySQL te **pedirá la contraseña de forma interactiva** (más seguro, no queda en el historial).

> [!TIP]
> **Credenciales por defecto**
> Prueba primero **`root` sin contraseña** o con contraseñas débiles/por defecto. Es de lo primero que se intenta al encontrar un MySQL expuesto.

---

#### Comandos SQL básicos

Una vez dentro del cliente (`mysql>`):

| Comando | Descripción |
|---------|-------------|
| `mysql -u <user> -p<password> -h <IP>` | Conectarse al servidor MySQL (`-p` **sin espacio** antes de la contraseña). |
| `show databases;` | Mostrar todas las bases de datos. |
| `use <database>;` | Seleccionar una de las bases de datos existentes. |
| `show tables;` | Mostrar todas las tablas de la base de datos seleccionada. |
| `show columns from <table>;` | Mostrar todas las columnas de la tabla seleccionada. |
| `select * from <table>;` | Mostrar todo el contenido de la tabla. |
| `select * from <table> where <column> = "<string>";` | Buscar una `string` concreta en la tabla. |

> [!NOTE]
> Toda sentencia SQL termina en **`;`**. Si el prompt cambia a `->` es que falta cerrar el comando (te falta el punto y coma o una comilla).

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `mysql -u <user> -p<password> -h <FQDN/IP>` | Login en el servidor MySQL (`-p` pegado) |
| `mysql -u root -h <FQDN/IP>` | Login sin contraseña (o la pide) |
| `show databases;` | Listar bases de datos |
| `use <database>;` | Seleccionar una base de datos |
| `show tables;` | Listar tablas |
| `show columns from <table>;` | Listar columnas de una tabla |
| `select * from <table>;` | Volcar una tabla |
| `SHOW VARIABLES LIKE 'secure_file_priv';` | Ver si se pueden leer/escribir ficheros |
| Puerto | 3306/TCP |

---

Relacionado: otras bases de datos → **MSSQL** · **Oracle TNS** · **0 - Footprinting (Índice y cheatsheet)**

---

### MSSQL — Footprinting

**Microsoft SQL Server (MSSQL)** se ejecuta sobre el framework **.NET**, es de **código cerrado** y corre principalmente en **Windows**. Escucha por defecto en el **puerto TCP 1433** y su lenguaje de consultas es **T-SQL (Transact-SQL)**, el dialecto SQL de Microsoft.

> [!NOTE]
> **Clientes**
> El cliente gráfico oficial es **SSMS (SQL Server Management Studio)**, que puede instalarse como característica de MSSQL. Desde Linux/pentest se usa **`mssqlclient.py`** (Impacket).

---

#### Bases de datos del sistema

| Base de datos | Descripción |
|---------------|-------------|
| `master` | Rastrea toda la información del sistema de una instancia de SQL Server. |
| `model` | Plantilla que actúa como estructura para cada nueva base de datos. Cualquier cambio en `model` se refleja en las bases de datos creadas después. |
| `msdb` | La usa el **Agente de SQL Server** para programar trabajos y alertas. |
| `tempdb` | Almacena objetos temporales. |
| `resource` | Base de datos de **solo lectura** con los objetos del sistema incluidos con SQL Server. |

---

#### Configuración por defecto

- El servicio se ejecuta como **`NT SERVICE\MSSQLSERVER`** cuando un administrador lo instala y configura.
- La autenticación suele estar en **Windows Authentication**: el SO Windows procesa la solicitud y usa la base de datos **SAM local** o el **Active Directory**.

> [!NOTE]
> **Windows Auth vs SQL Auth**
> - **Windows Authentication** → usa cuentas del SO/dominio (SAM o AD). Es la que activa el flag `-windows-auth` de `mssqlclient.py`.
> - **SQL Authentication** → usuarios propios de MSSQL (ej. `sa`), con su contraseña gestionada por el servidor.

---

#### Reconocimiento del servicio

MSSQL escucha en el puerto **`1433/TCP`**.

```
scanner/mssql/mssql_ping
```
Módulo de **Metasploit** para detectar instancias MSSQL en la red.

---

#### Conexión (mssqlclient.py)

Si conseguimos (adivinar u obtener) credenciales, podemos conectarnos remotamente e interactuar con las bases de datos usando **T-SQL**.

```bash
python3 mssqlclient.py Administrator@10.129.201.248 -windows-auth
```
Conexión con **autenticación de Windows/dominio** (`-windows-auth`).

```bash
python3 mssqlclient.py usuario:password@<FQDN/IP>
```
Conexión con **autenticación SQL nativa** (usuario/contraseña propios de MSSQL, sin `-windows-auth`).

---

#### Lenguaje de consultas (T-SQL)

> [!NOTE]
> Toda sentencia termina en `;`. En `mssqlclient.py` puedes escribir el T-SQL directamente en el prompt `SQL>`.

##### Reconocimiento inicial (ejecutar siempre al entrar)

```sql
SELECT @@VERSION;                      -- versión del servidor (buscar exploits conocidos)
SELECT SYSTEM_USER;                    -- con qué usuario has entrado
SELECT IS_SRVROLEMEMBER('sysadmin');   -- 1 = eres sysadmin, 0 = no
SELECT name FROM sys.databases;        -- listar todas las bases de datos
```

##### Navegar bases de datos y tablas

```sql
USE nombre_basedatos;                  -- seleccionar una base de datos

SELECT table_name FROM information_schema.tables;   -- listar tablas

SELECT column_name FROM information_schema.columns
WHERE table_name = 'nombre_tabla';                  -- columnas de una tabla

SELECT * FROM nombre_tabla;            -- volcar el contenido de la tabla
```

> [!NOTE]
> **`information_schema`**
> Es un conjunto de vistas estándar (existe también en MySQL) que describe la propia estructura de la base de datos: qué tablas y columnas hay. Es la forma de "mapear" una BD desconocida sin saber sus nombres de antemano.

##### Buscar credenciales / datos sensibles rápido

```sql
-- columnas cuyo nombre suene a contraseña, en toda la BD
SELECT table_name, column_name
FROM information_schema.columns
WHERE column_name LIKE '%pass%' OR column_name LIKE '%pwd%';
```

> [!TIP]
> Guarda cualquier credencial que encuentres en las tablas: a veces hay **contraseñas reutilizadas** para otros servicios de la máquina.

---

#### Explotación

> [!NOTE]
> **Cubierto en Attacking Common Services (actualizado)**
> Las vías de explotación de MSSQL (requieren normalmente rol **`sysadmin`**) ya están desarrolladas en detalle en **Atacando bases de datos SQL**:
> - **`xp_cmdshell`** → ejecutar comandos del sistema (**RCE**). `sysadmin` = jackpot.
> - **`OPENROWSET(BULK …)`** → leer ficheros del sistema desde SQL.
> - **`xp_dirtree`/`xp_subdirs`** → robar el hash de la cuenta de servicio MSSQL.
> - **Linked servers** (`sysservers`, `EXEC (...) AT [servidor]`) → movimiento lateral a otros SQL enlazados.

---

#### Resumen rápido (cheatsheet)

| Comando / Consulta | Qué hace |
|--------------------|----------|
| `mssqlclient.py <user>@<FQDN/IP> -windows-auth` | Conectar con autenticación de Windows |
| `mssqlclient.py user:password@<FQDN/IP>` | Conectar con autenticación SQL nativa |
| `SELECT @@VERSION;` | Versión del servidor |
| `SELECT SYSTEM_USER;` | Usuario actual |
| `SELECT IS_SRVROLEMEMBER('sysadmin');` | ¿Eres sysadmin? (1/0) |
| `SELECT name FROM sys.databases;` | Listar bases de datos |
| `USE <db>;` | Seleccionar base de datos |
| `SELECT table_name FROM information_schema.tables;` | Listar tablas |
| `SELECT column_name FROM information_schema.columns WHERE table_name='<t>';` | Listar columnas |
| `SELECT * FROM <tabla>;` | Volcar una tabla |
| Puerto | 1433/TCP · Metasploit `mssql_ping` |

---

Relacionado: otras bases de datos → **MYSQL** · **Oracle TNS** · mismo entorno Windows → **Protocolos de Administración Remota de Windows** · **0 - Footprinting (Índice y cheatsheet)**

---

### Oracle TNS — Footprinting

**Oracle TNS (Transparent Network Substrate)** es la capa de red de Oracle: comunica las **bases de datos y aplicaciones de Oracle a través de la red**. Se ha convertido en una solución preferida para gestionar bases de datos grandes y complejas. Habilita el **cifrado** entre la comunicación del cliente y el servidor mediante una capa adicional.

- Escucha por defecto en el **puerto TCP 1521** (el *listener* de Oracle).

> [!NOTE]
> **TNS y SID**
> - El servicio que escucha en 1521 es el **TNS listener**.
> - Para conectarte necesitas un **SID (System Identifier)** válido, que identifica la **instancia** de la base de datos. Por eso el primer paso suele ser **descubrir el SID** (fuerza bruta con Nmap, abajo) y luego usarlo en `sqlplus …@IP/SID`.

---

#### Configuración (tnsnames.ora)

El fichero **`tnsnames.ora`** contiene la información necesaria para que los clientes se conecten al servicio: la **ubicación de red** del servicio y el **nombre de la base de datos o servicio** que deben usar al conectarse.

---

#### Credenciales por defecto

> [!WARNING]
> **Credenciales por defecto de Oracle**
> Oracle es conocido por sus cuentas demo/por defecto, que conviene probar siempre:
> - **`DBSNMP` / `dbsnmp`** → cuenta de gestión con contraseña por defecto.
> - **`scott` / `tiger`** → credenciales demo legendarias de Oracle (las que se usan en el `sqlplus` de abajo).

---

#### Reconocimiento — Fuerza bruta de SID (Nmap)

```bash
sudo nmap -p1521 -sV 10.129.204.235 --open --script oracle-sid-brute
```
Usa el script NSE **`oracle-sid-brute`** para **descubrir SIDs válidos** en el listener del puerto 1521. Sin un SID válido no puedes conectarte con `sqlplus`.

---

#### ODAT (Oracle Database Attacking Tool)

```bash
./odat.py all -s 10.129.204.23
```
`odat.py` realiza una **batería de escaneos** para enumerar y recopilar información sobre los servicios de la base de datos Oracle y sus componentes: nombres de bases de datos, versiones, procesos en ejecución, cuentas de usuario, vulnerabilidades, configuraciones incorrectas, etc. La opción **`all`** prueba **todos los módulos** de la herramienta.

> [!NOTE]
> **Revisa la IP**
> En tus notas esta línea usa `10.129.204.23`, mientras el resto de comandos usa `10.129.204.235`. Parece que le falta un dígito (**typo**) — verifica cuál es la IP real de tu objetivo. Lo dejo tal como lo escribiste.

> [!NOTE]
> `odat` no viene por defecto; hay que instalarlo. Es la "navaja suiza" para auditar Oracle.

##### Subir un fichero al servidor (utlfile)

```bash
./odat.py utlfile -s <FQDN/IP> -d <db> -U <user> -P <pass> --sysdba --putFile C:\\insert\\path file.txt ./file.txt
```
Usa el módulo **`utlfile`** de ODAT (que abusa del paquete `UTL_FILE` de Oracle) para **subir un fichero** al sistema de ficheros del servidor. Formato de `--putFile`: `<directorio_remoto> <nombre_remoto> <fichero_local>`. Requiere credenciales y normalmente `--sysdba`.

> [!WARNING]
> **Por qué es importante**
> Poder **escribir ficheros** en el servidor abre la puerta a **RCE**: por ejemplo, subir una **webshell** al directorio web, o un script/binario en una ruta que luego se ejecute. Junto con la lectura de ficheros y el volcado de hashes, es una de las vías de compromiso total de un Oracle.

---

#### SQLplus — Inicio de sesión

```bash
sqlplus scott/tiger@10.129.204.235/XE
```
Se conecta a la base de datos con el cliente **`sqlplus`**, en formato `usuario/contraseña@IP/SID`. Aquí el SID es **`XE`** (Oracle **Express Edition**, el SID por defecto de esa edición).

##### Conectar como SYSDBA y volcar credenciales

```bash
sqlplus scott/tiger@10.129.204.235/XE as sysdba
```
Añadir **`as sysdba`** al conectar solicita el rol **SYSDBA**, el privilegio administrativo **más alto** de Oracle (equivalente al `root` de la base de datos: control total). Si el usuario tiene ese rol concedido, entras con permisos totales.

Ya dentro como SYSDBA, se pueden **volcar los hashes de contraseñas** de todas las cuentas:

```sql
SELECT name, password FROM sys.user$;
```
`sys.user$` es la tabla interna de Oracle que guarda **usuarios y sus hashes de contraseña**. Al leerla como SYSDBA obtienes los hashes de todas las cuentas.

> [!WARNING]
> **Por qué es importante**
> Esta consulta es uno de los objetivos clave al comprometer un Oracle: te da los **hashes de todos los usuarios de la BD**, que puedes **crackear offline** (p. ej. con `hashcat`) y reutilizar contra otros servicios/máquinas. Es la vía típica de robo de credenciales en Oracle.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `sudo nmap -p1521 -sV <IP> --open --script oracle-sid-brute` | Descubrir **SIDs** válidos (fuerza bruta) |
| `./odat.py all -s <IP>` | Enumeración completa del Oracle (todos los módulos) |
| `./odat.py utlfile -s <IP> -d <db> -U <user> -P <pass> --sysdba --putFile <dir> <nombre> <local>` | **Subir** un fichero al servidor (→ posible RCE) |
| `sqlplus <user>/<pass>@<IP>/<SID>` | Conectar a la base de datos (ej. `scott/tiger@IP/XE`) |
| `sqlplus <user>/<pass>@<IP>/<SID> as sysdba` | Conectar con el rol **SYSDBA** (control total) |
| `SELECT name, password FROM sys.user$;` | Volcar **hashes de contraseñas** (como SYSDBA) → crackear offline |
| Puerto | 1521/TCP (TNS listener) |
| Credenciales por defecto | `DBSNMP`/`dbsnmp` · `scott`/`tiger` |
| `tnsnames.ora` | Fichero de configuración de conexión de los clientes |

---

Relacionado: otras bases de datos → **MYSQL** · **MSSQL** · crackear los hashes de `sys.user$` → **Introduction to Hashcat** · **0 - Footprinting (Índice y cheatsheet)**

---

## Gestión remota

### Protocolos de gestión remota de Linux

Cubre los servicios que permiten administrar una máquina Linux por red: **SSH** (el estándar seguro), **Rsync** (sincronización de ficheros) y los antiguos **R-Services** (`rlogin`, `rsh`, `rexec`…).

---

#### SSH

**SSH (Secure Shell)** permite ejecutar comandos y administrar un host remoto de forma **cifrada**. Escucha por defecto en el **puerto TCP 22**. La implementación más conocida es **OpenSSH** (de OpenBSD, <https://www.openssh.com/>).

> [!NOTE]
> **SSH-2 vs SSH-1**
> **SSH-2** (SSH versión 2) es más avanzado que la versión 1 en cifrado, velocidad, estabilidad y seguridad. **SSH-1** es vulnerable a ataques **MITM** (Man-in-the-Middle); SSH-2 **no** lo es. Ver `Protocol 1` en configuraciones peligrosas.

##### Auditar el servicio (ssh-audit)

```bash
ssh-audit.py <FQDN/IP>
```
Realiza una **auditoría de seguridad remota** del servicio SSH: versión, algoritmos de cifrado/MAC/KEX soportados, y **marca los débiles o inseguros**. Es el primer paso para saber cómo está configurado el SSH del objetivo.

##### Conexión básica

```bash
ssh usuario@<IP>
```
```bash
ssh -i clave_privada usuario@<IP>     # con clave privada
ssh -p 2222 usuario@<IP>              # puerto no estándar
```
```bash
ssh usuario@<IP> -o PreferredAuthentications=password
```
`-o PreferredAuthentications=password` **fuerza la autenticación por contraseña**, ignorando las claves. Útil para probar credenciales (fuerza bruta / password spraying) cuando el servidor también acepta claves.

##### Autenticación por clave pública

1. **Autenticación del servidor**: el servidor envía su **clave pública de host** (`public host key`) al cliente, que la usa para verificar la identidad del servidor.
2. **Autenticación del cliente**: tras eso, el cliente debe demostrar al servidor que tiene autorización de acceso.

Con contraseña, el servidor ya guarda el **hash cifrado** de la contraseña del usuario, y hay que introducirla en cada inicio de sesión. La alternativa es un **par de clave pública/privada**:

- La **clave privada** se genera en el ordenador del usuario, se protege con una **frase de contraseña (passphrase)** —que debería ser más larga que una contraseña normal—, se almacena **solo localmente** y permanece **en secreto**.
- Para conectar, primero se introduce la passphrase (que desbloquea la clave privada), y esta prueba la identidad del cliente ante el servidor.

##### Configuración por defecto

```bash
cat /etc/ssh/sshd_config
```
Fichero de configuración del daemon SSH (`sshd`).

##### Configuraciones peligrosas

| Configuración | Descripción |
|---------------|-------------|
| `PasswordAuthentication yes` | Permite la autenticación basada en contraseña. |
| `PermitEmptyPasswords yes` | Permite el uso de **contraseñas vacías**. |
| `PermitRootLogin yes` | Permite iniciar sesión directamente como **root**. |
| `Protocol 1` | Usa una versión de cifrado **obsoleta** (SSH-1, vulnerable a MITM). |
| `X11Forwarding yes` | Permite el reenvío de X11 (aplicaciones GUI). |
| `AllowTcpForwarding yes` | Permite el reenvío de puertos TCP. |
| `PermitTunnel` | Permite la tunelización. |
| `DebianBanner yes` | Muestra un banner específico al iniciar sesión. |

> [!WARNING]
> **Por qué son peligrosas**
> - `PermitRootLogin yes` + `PasswordAuthentication yes` → un atacante puede **fuerza bruta directamente contra root**.
> - `PermitEmptyPasswords yes` → login **sin contraseña** si alguna cuenta la tiene vacía.
> - `Protocol 1` → cifrado roto, vulnerable a **MITM**.
> - `AllowTcpForwarding`/`PermitTunnel` → facilitan **pivoting** hacia la red interna si consigues acceso.

---

#### Rsync

**Rsync** copia y sincroniza archivos de forma **local y remota**. Usa el **puerto 873** y puede configurarse para usar **SSH** para transferencias seguras.

```bash
nc -nv 127.0.0.1 873
```
Leer el **banner** del servicio rsync con netcat.

```bash
rsync -av --list-only rsync://127.0.0.1/dev
```
**Lista** (`--list-only`) el contenido del módulo `dev` sin descargar. `-a` = modo archivo (conserva permisos/tiempos), `-v` = verbose.

```bash
rsync -av rsync://127.0.0.1/dev
```
**Descarga** el contenido del módulo `dev`.

> [!TIP]
> **Rsync sobre SSH**
> Si Rsync está configurado para transferir sobre SSH, añade la bandera **`-e ssh`** (o `-e "ssh -p2222"` para un puerto distinto).

> [!NOTE]
> Un módulo rsync accesible **sin autenticación** puede filtrar ficheros sensibles (config, claves, credenciales). Primero **lista** (`--list-only`) qué módulos/carpetas hay expuestos, y luego descarga lo interesante.

---

#### R-Services

Servicios de administración remota **antiguos e inseguros** (texto plano), predecesores de SSH. Usan los **puertos 512, 513 y 514**.

| Comando | Demonio | Puerto | Transporte | Descripción |
|---------|---------|--------|------------|-------------|
| `rcp` | `rshd` | 514 | TCP | Copia archivos/directorios entre sistemas (como `cp`), pero **sin avisar** al sobrescribir ficheros existentes. |
| `rsh` | `rshd` | 514 | TCP | Abre un shell en una máquina remota **sin login**. Se basa en la confianza de `/etc/hosts.equiv` y `.rhosts`. |
| `rexec` | `rexecd` | 512 | TCP | Ejecuta comandos en una máquina remota. Requiere usuario/contraseña por un **socket no cifrado**; la autenticación se **anula** con las entradas de confianza de `/etc/hosts.equiv` y `.rhosts`. |
| `rlogin` | `rlogind` | 513 | TCP | Login en un host remoto (similar a `telnet`, pero solo a hosts Unix). La autenticación se **anula** con la confianza de `/etc/hosts.equiv` y `.rhosts`. |

##### Ficheros de confianza

- **`/etc/hosts.equiv`** → lista de **hosts de confianza**; otorga acceso a otros sistemas de la red. Es la configuración **global** para todos los usuarios del sistema.
- **`.rhosts`** → configuración **por usuario** (en el home del usuario).

> [!WARNING]
> **Abuso de la confianza (.rhosts)**
> Si una cuenta confía en hosts/usuarios vía `hosts.equiv`/`.rhosts`, `rsh`/`rlogin`/`rexec` te dejan entrar **sin contraseña**. Una entrada permisiva (ej. `+ +`) permite el acceso desde cualquier host/usuario → acceso directo a la máquina.

##### Iniciar sesión con Rlogin

```bash
rlogin 10.0.17.2 -l htb-student
```
Inicia sesión (`-l` especifica el usuario) en el host remoto. Si funciona sin pedir contraseña, es por la relación de confianza.

Tras entrar, se puede abusar de **`rwho`** para listar **todas las sesiones interactivas** de la red local (envía solicitudes al **puerto UDP 513**).

##### Listar usuarios autenticados con Rusers

```bash
rusers -al 10.0.17.5
```
Lista los **usuarios autenticados** en el host (información útil para enumerar cuentas válidas).

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `ssh-audit.py <FQDN/IP>` | Auditoría de seguridad del SSH |
| `ssh <user>@<IP>` | Conexión SSH (puerto 22) |
| `ssh -i <clave> <user>@<IP>` | Conexión SSH con clave privada |
| `ssh <user>@<IP> -o PreferredAuthentications=password` | Forzar autenticación por contraseña |
| `cat /etc/ssh/sshd_config` | Config del servidor SSH |
| `nc -nv <IP> 873` | Banner de Rsync |
| `rsync -av --list-only rsync://<IP>/<módulo>` | Listar un módulo Rsync |
| `rsync -av rsync://<IP>/<módulo>` | Descargar un módulo Rsync |
| `rsync … -e ssh` | Rsync sobre SSH |
| `rlogin <IP> -l <user>` | Login por R-services (513) |
| `rusers -al <IP>` | Listar usuarios autenticados |
| `rwho` | Listar sesiones interactivas (UDP 513) |
| Puertos | SSH 22 · Rsync 873 · R-services 512/513/514 |
| Config peligrosa | SSH `PermitRootLogin yes` / `Protocol 1` · R-services `.rhosts` con `+ +` |

---

Relacionado: equivalente en Windows → **Protocolos de Administración Remota de Windows** · fuerza bruta remota (SSH, etc.) → **Servicios en red** · **0 - Footprinting (Índice y cheatsheet)**

---

### Protocolos de administración remota de Windows

Servicios para administrar máquinas Windows por red: **RDP** (escritorio remoto gráfico), **WinRM** (shell remoto) y **WMI** (gestión/ejecución vía RPC).

---

#### RDP (Remote Desktop Protocol)

Desarrollado por **Microsoft**. Escucha en el **puerto TCP 3389** (también puede usar **UDP 3389** para la administración remota).

- Para establecer una sesión RDP, tanto el **firewall de la red** como el **firewall del servidor** deben permitir conexiones desde el exterior.
- Maneja **TLS/SSL**, por lo que las conexiones están **bien cifradas** en la red.

##### Comprobar la seguridad del RDP

```bash
rdp-sec-check.pl <FQDN/IP>
```
Revisa la **configuración de seguridad** del servicio RDP (nivel de cifrado, NLA — *Network Level Authentication*, protocolos soportados). Sirve para detectar configuraciones débiles antes de conectar.

##### Conectar al RDP desde Linux

```bash
xfreerdp /u:cry0l1t3 /p:"P455w0rd!" /v:10.129.201.248
```
Inicia sesión en el servidor RDP con `xfreerdp`: `/u:` usuario, `/p:` contraseña (entre comillas si tiene caracteres especiales), `/v:` el objetivo.

---

#### WinRM (Windows Remote Management)

Sistema de **administración remota integrado en Windows**, basado en línea de comandos (implementa el estándar **WS-Management**).

- A partir de **Windows 10** debe **habilitarse y configurarse explícitamente** (no está activo por defecto).
- Puertos: **TCP 5985** (HTTP) y **5986** (HTTPS).

##### Conectar al WinRM

```bash
evil-winrm -i 10.129.201.248 -u Cry0l1t3 -p P455w0rD!
```
Inicia sesión en el servidor WinRM con **`evil-winrm`** (`-i` host, `-u` usuario, `-p` contraseña), obteniendo una **shell de PowerShell** remota.

> [!NOTE]
> Para entrar por WinRM, el usuario debe ser **administrador local** o pertenecer al grupo **`Remote Management Users`**. `evil-winrm` es la herramienta estándar en pentest de Windows/AD.

---

#### WMI (Windows Management Instrumentation)

Permite el **acceso de lectura y escritura a casi todas las configuraciones** de los sistemas Windows. Normalmente se accede a WMI a través de **PowerShell**, **VBScript** o la consola **WMIC** (Instrumental de administración de Windows).

- Recon: usa **RPC** sobre el **puerto TCP 135** (endpoint mapper / DCOM), y luego puertos altos dinámicos.

> [!NOTE]
> **Corrección**
> En las notas WMI se describía como "Administración Empresarial Basada en Web". En realidad **WMI = Windows Management Instrumentation**, que es la **implementación de Microsoft** del estándar **WBEM (Web-Based Enterprise Management)**. De ahí la confusión.

##### Ejecutar comandos con wmiexec.py

```bash
wmiexec.py Cry0l1t3:"P455w0rD!"@10.129.201.248 "hostname"
```
Ejecuta un comando del sistema en el objetivo usando el servicio **WMI**, con **`wmiexec.py`** (Impacket). Formato: `usuario:"contraseña"@<IP> "<comando>"`.

> [!NOTE]
> En tus notas usabas la ruta completa `/usr/share/doc/python3-impacket/examples/wmiexec.py`. Si Impacket está instalado con los binarios en el `PATH`, basta con `wmiexec.py`. `wmiexec` da una **shell semi-interactiva** (ejecuta comando a comando) sin dejar apenas rastro en disco.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `rdp-sec-check.pl <FQDN/IP>` | Comprobar la seguridad del servicio RDP |
| `xfreerdp /u:<user> /p:"<password>" /v:<FQDN/IP>` | Conectar al RDP desde Linux |
| `evil-winrm -i <FQDN/IP> -u <user> -p <password>` | Shell remota por WinRM |
| `wmiexec.py <user>:"<password>"@<FQDN/IP> "<comando>"` | Ejecutar comandos vía WMI |
| Puertos | RDP 3389 · WinRM 5985/5986 · WMI 135 (+ puertos dinámicos) |

---

Relacionado: equivalente en Linux → **Protocolos de gestión remota de Linux** · mismo entorno Windows → **SMB** · **MSSQL** · fuerza bruta remota (RDP/WinRM) y Pass-the-Hash → **Servicios en red** · **Pass the Hash (PtH)** · **0 - Footprinting (Índice y cheatsheet)**

---

## Otros servicios

### SNMP — Footprinting

**SNMP (Simple Network Management Protocol)** es un protocolo para la **monitorización y gestión** de dispositivos de red. Permite gestionar tareas de configuración y realizar ajustes **de forma remota**. La versión actual es **SNMPv3**, que aumenta la seguridad pero también la complejidad.

- Transmite sobre el **puerto UDP 161**.
- Para intercambiar valores, los objetos SNMP deben tener **direcciones únicas** conocidas en ambos lados.

> [!NOTE]
> **Puertos**
> - **161/UDP** → consultas normales (get/set).
> - **162/UDP** → *traps* (notificaciones que el dispositivo envía al gestor).

> [!WARNING]
> **Community string**
> En SNMPv1/v2c, la **community string** funciona como una "contraseña" para acceder al servicio. Valores por defecto muy comunes:
> - **`public`** → acceso de **solo lectura**.
> - **`private`** → acceso de **lectura-escritura**.
>
> Probar estos dos por defecto es lo primero al enumerar SNMP.

---

#### MIB y OID

- **MIB (Management Information Base)**: base de información de gestión que **almacena la información de los dispositivos**. Es un archivo de texto (en **ASCII**) donde los datos están listados en una **jerarquía de árbol**, y contiene identificadores de objeto (**OID**).
- **OID (Object Identifier)**: representa un **nodo** dentro de ese espacio de nombres jerárquico (ej. `.1.3.6.1.2.1…`). Cada dato consultable tiene su OID.

---

#### Versiones

| Versión | Autenticación | Cifrado | Notas |
|---------|---------------|---------|-------|
| **SNMPv1** | Ninguna | No | Cualquiera con acceso a la red puede **leer y modificar** datos. |
| **SNMPv2c** | Community string | No | A efectos de seguridad, **a la par de v1** (la `c` = *community-based*). Es la más común en CTF. |
| **SNMPv3** | Usuario + contraseña | Sí | Autenticación por usuario y **cifrado** de la transmisión. Más seguro, más complejo. |

---

#### Configuración por defecto

```bash
cat /etc/snmp/snmpd.conf
```
Fichero de configuración del daemon SNMP (`snmpd`). Revisarlo revela las community strings y los permisos configurados.

---

#### Ajustes peligrosos

| Ajuste | Descripción |
|--------|-------------|
| `rwuser noauth` | Proporciona acceso al **árbol OID completo sin autenticación**. |
| `rwcommunity <community string> <IPv4>` | Acceso al árbol OID completo **independientemente del origen** de las solicitudes. |
| `rwcommunity6 <community string> <IPv6>` | Igual que `rwcommunity` pero sobre **IPv6**. |

> [!WARNING]
> **Por qué son peligrosas**
> La `rw` significa **read-write**: no solo permiten **leer** todo el árbol OID, sino **modificar la configuración** del dispositivo. Combinado con `noauth` o una community string por defecto, un atacante puede reconfigurar el equipo de forma remota.

---

#### Footprinting del servicio

Herramientas principales: **`snmpwalk`** (consultar OIDs), **`onesixtyone`** (fuerza bruta de community strings) y **`braa`** (fuerza bruta de OIDs).

##### snmpwalk — Consultar OIDs

```bash
snmpwalk -v2c -c public 10.129.14.128
```
Recorre y consulta los **OIDs** del objetivo con toda su información. `-v2c` indica la **versión 2c** y `-c public` la **community string**. Si no conoces la community, primero hazle fuerza bruta con `onesixtyone`.

##### onesixtyone — Fuerza bruta de community strings

```bash
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128
```
Prueba un diccionario (`-c`) de **community strings** contra el objetivo para descubrir cuál es válida. Es rápido porque SNMP va sobre UDP.

##### braa — Fuerza bruta de OIDs

```bash
braa <community string>@10.129.14.128:.1.3.6.*
```
Con una community string ya conocida, hace **fuerza bruta / recorrido masivo de OIDs** de forma mucho más rápida que `snmpwalk` (útil cuando el árbol es enorme). El patrón `.1.3.6.*` acota la rama del árbol a recorrer.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `snmpwalk -v2c -c <community> <FQDN/IP>` | Consultar los OIDs del objetivo |
| `onesixtyone -c <wordlist> <FQDN/IP>` | Fuerza bruta de **community strings** |
| `braa <community>@<FQDN/IP>:.1.3.6.*` | Fuerza bruta / recorrido rápido de **OIDs** |
| `cat /etc/snmp/snmpd.conf` | Config del daemon SNMP |
| Community por defecto | `public` (lectura) · `private` (lectura-escritura) |
| Puertos | 161/UDP (consultas) · 162/UDP (traps) |
| `rwcommunity` / `rwuser noauth` | Config peligrosa → escritura del árbol OID |

---

Relacionado: otro servicio de gestión por UDP → **IPMI** · **0 - Footprinting (Índice y cheatsheet)**

---

### IPMI — Footprinting

**IPMI (Intelligent Platform Management Interface)** sirve para la **gestión y monitorización** de sistemas. Funciona de forma **independiente de la BIOS, la CPU y el sistema operativo**, lo que permite a los administradores **gestionar y monitorizar** el equipo **incluso si está apagado** o en un estado que no responde.

- Puede monitorizar temperaturas, voltaje, estado de los ventiladores, etc.
- El módulo IPMI requiere una **fuente de alimentación** y una **conexión LAN** para funcionar.

> [!NOTE]
> **BMC**
> IPMI lo implementa el **BMC (Baseboard Management Controller)**: un microcontrolador dedicado en la placa, separado de la CPU principal, que sigue funcionando aunque el sistema esté apagado. Marcas conocidas: **Dell iDRAC**, **HP iLO**, **Supermicro IPMI**.

---

#### Reconocimiento del servicio

IPMI se comunica a través del **puerto 623/UDP**.

```bash
sudo nmap -sU --script ipmi-version -p 623 ilo.inlanfreight.local
```
Escaneo UDP (`-sU`) con el script NSE **`ipmi-version`** para detectar el servicio IPMI y su versión en el puerto 623.

**Metasploit** — módulo equivalente para detectar la versión:
```
auxiliary/scanner/ipmi/ipmi_version
```
Referencia: <https://www.rapid7.com/db/modules/auxiliary/scanner/ipmi/ipmi_version/>

---

#### BMC y contraseñas por defecto

Durante pruebas de penetración internas es habitual encontrar **BMCs** cuyos administradores **no han cambiado la contraseña por defecto**. Algunas por defecto que conviene tener a mano:

| Producto | Usuario | Contraseña |
|----------|---------|------------|
| Dell iDRAC | `root` | `calvin` |
| HP iLO | `Administrator` | cadena aleatoria de **8 caracteres** (números y letras mayúsculas) |
| Supermicro IPMI | `ADMIN` | `ADMIN` |

---

#### Volcado de hashes (CVE-2013-4786)

```
auxiliary/scanner/ipmi/ipmi_dumphashes
```
Módulo de **Metasploit** que **vuelca los hashes de contraseña** de las cuentas del BMC **sin necesidad de credenciales**.

> [!WARNING]
> **Por qué funciona**
> El protocolo **IPMI 2.0** tiene un fallo de diseño en su autenticación **RAKP** (CVE-2013-4786): durante el proceso, el servidor **envía un hash HMAC de la contraseña del usuario antes de completar la autenticación**. Es decir, cualquiera que pueda hablar con el BMC puede **pedir el hash de un usuario válido** y luego **crackearlo offline**. Es la vía de ataque característica de IPMI.

---

#### Crackear el hash

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```
Crackea el hash volcado. `-m 7300` es el modo **IPMI2 RAKP HMAC-SHA1**. La máscara `?1?1?1?1?1?1?1?1` con `-1 ?d?u` prueba **todas las combinaciones de dígitos y letras mayúsculas para una contraseña de 8 caracteres** — justo el formato de la contraseña por defecto de **HP iLO** (por eso este ataque es tan efectivo contra iLO).

---

#### Flujo de explotación (paso a paso)

Con **IPMI 2.0** detectado (`nmap` muestra `IPMI-2.0` / `Level: … 2.0`), el ataque estándar es: **volcar hashes → crackear → entrar**.

**1. Volcar los hashes (Metasploit):**

```bash
msfconsole -q
```
```
use auxiliary/scanner/ipmi/ipmi_dumphashes
set RHOSTS <IP>
set OUTPUT_HASHCAT_FILE ipmi_hashes.txt
run
```
El módulo enumera usuarios (`ADMIN`, `root`, `Administrator`…) y, por el fallo RAKP, se trae el **hash HMAC-SHA1** de cada uno **sin autenticarse**. También prueba contraseñas por defecto, así que a veces te da la password directamente.

**2. Crackear el hash:** con `hashcat -m 7300` (ver sección [Crackear el hash](#crackear-el-hash)) — diccionario primero, o la máscara de HP iLO si aplica.

**3. Usar las credenciales:**

- **Interfaz web del BMC** → normalmente en `https://<IP>` (consola remota, montar ISOs, encender/apagar…).
- **Por línea de comandos** con `ipmitool`:

```bash
ipmitool -I lanplus -H <IP> -U <user> -P <password> user list
```

> [!TIP]
> **Antes de ponerte a crackear**
> `ipmi_dumphashes` ya te **confirma usuarios válidos** y prueba defaults. Si el BMC es iDRAC/iLO/Supermicro, prueba primero las **credenciales por defecto** de la tabla de arriba (`root:calvin`, `ADMIN:ADMIN`) — a veces te ahorras el crackeo entero.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `sudo nmap -sU --script ipmi-version -p 623 <host>` | Detectar IPMI y su versión (UDP) |
| `auxiliary/scanner/ipmi/ipmi_version` | Metasploit: detectar versión |
| `auxiliary/scanner/ipmi/ipmi_dumphashes` | Metasploit: **volcar hashes** sin credenciales (CVE-2013-4786) |
| `hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u` | Crackear el hash (máscara de HP iLO) |
| `ipmitool -I lanplus -H <IP> -U <user> -P <pass> user list` | Usar las credenciales ya obtenidas |
| Web del BMC | `https://<IP>` (consola remota, montar ISOs…) |
| Puerto | 623/UDP |
| Defaults | iDRAC `root:calvin` · iLO `Administrator:<8 chars>` · Supermicro `ADMIN:ADMIN` |

---

Relacionado: otro servicio de gestión por UDP → **SNMP** · crackeo de hashes → **Introduction to Hashcat** · **0 - Footprinting (Índice y cheatsheet)**

---

### DNS — Footprinting

El **DNS (Domain Name System)** es un sistema para resolver nombres de computadoras (dominios) en direcciones IP.

> [!NOTE]
> **Puerto 53**
> - **53/UDP** → consultas normales (la mayoría del tráfico DNS).
> - **53/TCP** → **transferencias de zona (AXFR)** y respuestas grandes (>512 bytes). Por eso el AXFR va por TCP.

---

#### Tipos de registro DNS

| Registro | Descripción |
|----------|-------------|
| `A` | Devuelve una dirección **IPv4** del dominio solicitado. |
| `AAAA` | Devuelve una dirección **IPv6** del dominio solicitado. |
| `MX` | Devuelve los **servidores de correo** responsables del dominio. |
| `NS` | Devuelve los **servidores DNS** (*nameservers*) del dominio. |
| `TXT` | Registro polivalente: puede contener información diversa. Se usa, por ejemplo, para validar Google Search Console o certificados SSL. También aloja entradas **SPF** y **DMARC** para validar el tráfico de correo y protegerlo del spam. |
| `CNAME` | Actúa como **alias** de otro nombre de dominio. Ej.: para que `www.hackthebox.eu` apunte a la misma IP que `hackthebox.eu`, se crea un registro `A` para `hackthebox.eu` y un `CNAME` para `www.hackthebox.eu`. |
| `PTR` | Funciona al revés (**búsqueda inversa** / *reverse lookup*): convierte direcciones IP en nombres de dominio. |
| `SOA` | Proporciona información sobre la **zona DNS** correspondiente y la dirección de correo del contacto administrativo. |

---

#### Footprinting del servicio (dig)

> [!NOTE]
> **El operador `@`**
> En `dig`, `@<nameserver>` **dirige la consulta a ese servidor concreto** en vez de al resolvedor por defecto del sistema. Clave para interrogar directamente a cada nameserver del objetivo.

##### dig — Consulta NS

```bash
dig ns <domain.tld> @<nameserver>
```
Consulta al servidor DNS **qué otros servidores de nombres (NS)** conoce del dominio. Es el punto de partida: te da los nameservers autoritativos contra los que seguir enumerando.

##### dig — Consulta de versión (CHAOS TXT)

```bash
dig CH TXT version.bind 10.129.120.85
```
A veces es posible **consultar la versión** de un servidor DNS con una consulta de clase **CHAOS** (`CH`) y tipo `TXT`. Conocer la versión (ej. de BIND) ayuda a buscar vulnerabilidades asociadas.

##### dig — Consulta ANY

```bash
dig any <domain.tld> @<nameserver>
```
La opción `ANY` pide **todos los registros disponibles**: el servidor mostrará todas las entradas que esté dispuesto a revelar.

> [!NOTE]
> Los servidores modernos suelen **limitar o negar** las consultas `ANY` (RFC 8482), devolviendo una respuesta mínima. Aun así, merece la pena probarlo por si filtra algo.

##### dig — Transferencia de zona (AXFR)

```bash
dig axfr internal.inlanefreight.htb @10.129.14.128
```
Te entrega literalmente **toda la zona** de un tirón: todos los subdominios, IPs internas, registros MX, etc. Es el **"premio gordo"** del reconocimiento DNS. **Pruébalo siempre contra cada nameserver** que encuentres.

> [!WARNING]
> **Cuándo funciona el AXFR**
> Una transferencia de zona debería estar **restringida** a los servidores secundarios autorizados. Si un nameserver te la sirve a ti (un cliente cualquiera), es un **fallo de configuración** que filtra toda la zona interna. Lo normal es que falle (esté bien configurado); cuando funciona, es un hallazgo enorme.

---

#### Fuerza bruta de subdominios (dnsenum)

```bash
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt inlanefreight.htb
```
Si el AXFR falla (que es lo normal), esto hace **fuerza bruta de subdominios** contra ese servidor usando un diccionario (`-f`). Los flags `-p 0 -s 0` limitan el número de subredes/páginas de Whois a consultar (para ir más rápido y centrarte solo en el enum).

- `--dnsserver 10.129.14.128` → el servidor DNS al que preguntar.
- `-o subdomains.txt` → fichero de salida con los subdominios encontrados.
- `-f <wordlist>` → diccionario de subdominios (aquí, uno de **SecLists**).

---

#### Metodología recomendada

1. **`dig ns`** → descubrir todos los nameservers autoritativos.
2. Contra **cada uno**, prueba **`dig axfr`** — es gratis y a veces funciona (premio gordo).
3. Si no funciona, **`dig any`** para ver qué se filtra igualmente.
4. Si sigues sin nada, **fuerza bruta** con `dnsenum` o `fierce`/`ffuf` con wordlists tipo **SecLists**.

---

#### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `dig ns <domain.tld> @<nameserver>` | Consulta los **nameservers (NS)** del dominio |
| `dig any <domain.tld> @<nameserver>` | Pide **todos los registros** (ANY) |
| `dig axfr <domain.tld> @<nameserver>` | **Transferencia de zona** completa (AXFR) |
| `dig CH TXT version.bind <IP>` | Consultar la **versión** del servidor DNS |
| `dnsenum --dnsserver <ns> --enum -p 0 -s 0 -o found_subdomains.txt -f <wordlist> <domain.tld>` | **Fuerza bruta** de subdominios |
| `@<nameserver>` | Dirigir la consulta a un servidor DNS concreto |

---

Relacionado: registros MX → **SMTP** · recon de subdominios a fondo → **DNS (Information Gathering Web)** · **0 - Footprinting (Índice y cheatsheet)**

---
