# Footprinting — HTB Academy (CPTS)

**Plataforma:** Hack The Box Academy
**Módulo:** Footprinting
**Tipo:** Skills Assessment (3 laboratorios independientes: Fácil, Medio, Difícil)
**Certificación:** CPTS (Certified Penetration Testing Specialist)
**Fecha de resolución:** 2026
**Idioma:** Español — [🇬🇧 English version](./Footprinting_Writeup_EN.md)

---

## Contexto del ejercicio

La empresa ficticia **Inlanefreight Ltd** nos ha encargado probar tres servidores diferentes en su red interna. La empresa utiliza muchos servicios distintos y el departamento de seguridad de TI consideró necesaria una prueba de penetración para comprender mejor su postura de seguridad general.

El objetivo en los tres laboratorios es el mismo: enumerar exhaustivamente cada servicio, identificar qué información puede extraerse de él y cómo esa información puede encadenarse para obtener acceso, **sin explotar activamente vulnerabilidades** (los servicios están en producción). Los compañeros de equipo aportan una pista inicial común a los tres laboratorios: las credenciales `ceil:qwer1234` y el rumor de que algunos empleados hablan de claves SSH en un foro.

## Índice
1. [Laboratorio Fácil — Servidor DNS/FTP/SSH](#1-laboratorio-fácil--servidor-dnsftpssh)
2. [Laboratorio Medio — Servidor NFS/RDP/MSSQL](#2-laboratorio-medio--servidor-nfsrdpmssql)
3. [Laboratorio Difícil — Servidor SNMP/IMAP/MySQL](#3-laboratorio-difícil--servidor-snmpimapmysql)
4. [Lección aprendida](#4-lección-aprendida)
5. [Glosario de herramientas y comandos](#5-glosario-de-herramientas-y-comandos)

---

## 1. Laboratorio Fácil — Servidor DNS/FTP/SSH

> **Escenario:** el primer servidor es un **servidor DNS interno**. El cliente quiere saber qué información se puede obtener de él y cómo podría usarse en su contra. Los administradores dejaron un `flag.txt` en el servidor para acreditar el compromiso.

### Pregunta oficial del Skills Assessment

| Pregunta | Respuesta |
|----------|-----------|
| *Enumerate the server carefully and find the flag.txt file. Submit the contents of this file as the answer.* | 🔒 Respuesta omitida a propósito — el procedimiento completo hasta llegar a `cat flag.txt` está documentado en §1.4; el contenido exacto del flag no se publica para no regalar la respuesta del laboratorio a quien lo esté cursando |

### 1.1 Reconocimiento

Escaneo completo de puertos TCP:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.177 -oN AllPorts
```

> 🛠️ **`nmap`** — escáner de puertos y servicios de referencia en cualquier pentest. `-sS` (SYN scan, no completa el *handshake* TCP, más sigiloso), `-Pn` (no descarta el host aunque no responda a ping — muy usado en labs de HTB, donde el ICMP suele estar filtrado), `--min-rate 5000` (fuerza un envío rápido de paquetes), `--open` (solo muestra puertos abiertos, reduce ruido), `-p-` (escanea los 65535 puertos), `-oN` (guarda la salida en formato normal a fichero, útil para adjuntar evidencias).

![](Imagenes/01-facil-nmap-todos-los-puertos.png)

Cuatro puertos abiertos: **21** (ftp), **22** (ssh), **53** (domain) y **2121** (un segundo FTP en puerto no estándar). Escaneo de versión y scripts por defecto:

```bash
nmap -sS -Pn -sCV -T5 -n -p21,22,53,2121 10.129.49.177 -oN Ports
```

> 🛠️ **`-sCV`** — combina `-sC` (ejecuta los scripts NSE por defecto de Nmap: banners, comprobaciones básicas de configuración) con `-sV` (detección de versión exacta del servicio). **`-T5`** ajusta la plantilla de temporización al máximo, razonable en un laboratorio sin IDS real de por medio. **`-n`** desactiva la resolución DNS inversa, ganando algo de velocidad.

![](Imagenes/02-facil-nmap-version-servicios.png)

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 21 | ProFTPD | Banner `ftp.int.inlanefreight.htb` |
| 22 | OpenSSH 8.2p1 (Ubuntu) | — |
| 53 | ISC BIND 9.16.1 (Ubuntu) | — |
| 2121 | ProFTPD | Banner **"Ceil's FTP"** |

> 💡 El segundo servicio FTP en el puerto 2121 se identifica explícitamente como el FTP de **Ceil** en su banner — encaja directamente con las credenciales `ceil:qwer1234` que nos había pasado el equipo.

### 1.2 Acceso inicial — FTP con credenciales filtradas

Nos conectamos al FTP del puerto no estándar (2121) con las credenciales conocidas:

```bash
ftp 10.129.49.177 2121
Name: ceil
Password: qwer1234
```

> 🛠️ **`ftp`** — cliente de línea de comandos para el protocolo **FTP** (File Transfer Protocol). Tras conectar, se autentica de forma interactiva pidiendo usuario y contraseña; una vez dentro admite comandos propios del protocolo (`ls`, `cd`, `get`, `put`...), distintos de los del shell del sistema operativo.

![](Imagenes/03-facil-ftp-2121-login-ceil.png)

Dentro del `home` de `ceil` hay una carpeta `.ssh`. La listamos y descargamos su clave privada:

```
ftp> cd .ssh
ftp> ls -la
ftp> get id_rsa
```

> 🛠️ **`get <fichero>`** (comando interno de FTP) — descarga un fichero del servidor remoto a la máquina local. Aquí es la forma de extraer la clave privada SSH sin necesidad de otro protocolo.

![](Imagenes/04-facil-ftp-ssh-carpeta-descarga-id-rsa.png)

> 💡 Confirma la pista del equipo sobre "claves SSH": la propia cuenta `ceil` tiene su clave privada expuesta vía FTP, con `authorized_keys` presente también en la misma carpeta — es su propia clave de acceso.

### 1.3 Obtención de shell

Con la clave privada descargada, ajustamos permisos y nos conectamos por SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa ceil@10.129.49.177
```

> 🛠️ **`chmod 600`** — restringe los permisos de un fichero a lectura/escritura solo para su propietario. **Obligatorio** para claves privadas SSH: OpenSSH rechaza usar una clave con permisos más abiertos ("UNPROTECTED PRIVATE KEY FILE"), precisamente para evitar que cualquier otro usuario del sistema pueda leerla.
>
> 🛠️ **`ssh -i <clave>`** — cliente SSH indicando explícitamente qué clave privada usar para la autenticación (`-i` de *identity file*), en vez de depender de contraseña o de las claves por defecto en `~/.ssh/`.

![](Imagenes/05-facil-ssh-login-ceil-clave-privada.png)

Acceso confirmado como `ceil` en el host **NIXEASY**.

### 1.4 Post-explotación y flag

```bash
cd /home
ls
```

> 🛠️ **`cd` / `ls`** — comandos básicos de navegación: `cd` cambia de directorio, `ls` lista su contenido. El primer paso lógico tras obtener shell en cualquier host es mirar qué otros usuarios/directorios existen en `/home`.

![](Imagenes/06-facil-home-flag-cat-flag-txt.png)

Además de `ceil`, hay otros dos directorios: **`cry0llt3`** (otro usuario del sistema) y **`flag`**. Entramos en este último y mostramos el fichero solicitado:

```bash
cd flag
cat flag.txt
```

> 🛠️ **`cat <fichero>`** — imprime el contenido de un fichero de texto por pantalla. Es el comando estándar para leer flags, ficheros de configuración, logs, etc.

> 🔒 Como se indica en la tabla de preguntas de esta sección, el contenido exacto de `flag.txt` no se publica en este writeup a propósito, para que quien lo use de guía tenga que completar este último paso por su cuenta.

---

## 2. Laboratorio Medio — Servidor NFS/RDP/MSSQL

> **Escenario:** un segundo servidor al que **todos en la red interna tienen acceso** — uno de los objetivos prioritarios típicos de atacantes reales. Para la prueba se creó un usuario **`HTB`**, cuyas credenciales hay que recuperar.

### Pregunta oficial del Skills Assessment

| Pregunta | Respuesta |
|----------|-----------|
| *Enumerate the server carefully and find the username "HTB" and its password. Then, submit this user's password as the answer.* | 🔒 Respuesta omitida a propósito — la consulta SQL que devuelve la contraseña de `HTB` está documentada en §2.5; el valor exacto no se publica para no regalar la respuesta del laboratorio |

### 2.1 Reconocimiento

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.180 -oN AllPorts
```

![](Imagenes/07-medio-nmap-todos-los-puertos.png)

Puertos típicos de un **Windows Server con Active Directory / NFS habilitado**: 111 (rpcbind), 135 (msrpc), 139/445 (SMB), 2049 (NFS), 3389 (RDP), 5985/47001 (WinRM/WSMan) y un rango de puertos RPC dinámicos.

```bash
nmap -sS -Pn -sCV -T5 -n -p111,135,139,445,2049,3389,5985,47001,49664-49668,49679-49681 10.129.49.180 -oN Ports
```

![](Imagenes/08-medio-nmap-version-servicios-smb-rdp.png)

> 💡 El script `ssl-cert`/`rdp-ntlm-info` de nmap ya revela el **NetBIOS name `WINMEDIUM`**, y el firmado SMB "enabled but not required" — indicios útiles aunque no explotables sin agresividad.

### 2.2 Enumeración NFS

Con NFS abierto en el 2049, lanzamos todos los scripts de nmap específicos para ese servicio:

```bash
nmap --script nfs* -sV -p111,2049 -oN NFS 10.129.49.180
```

> 🛠️ **`--script nfs*`** — ejecuta todos los scripts NSE de Nmap cuyo nombre empieza por `nfs` (comodín `*`): listan los recursos exportados (`nfs-ls`), sus permisos, estadísticas del sistema de ficheros, etc. Una forma rápida de enumerar NFS sin tener que montarlo primero.

![](Imagenes/09-medio-nmap-scripts-nfs.png)

El script `nfs-ls` revela un recurso exportado llamado **`/TechSupport`**, accesible con permisos `Read Lookup` sin autenticación, que contiene múltiples ficheros `ticketXXXXXXXXXXXX.txt` de tamaño 0.

Montamos el recurso:

```bash
sudo mount -t nfs 10.129.49.180:/ ./lab_NFS -o nolock
ls
sudo su
cd TechSupport
ls -la
```

> 🛠️ **`mount -t nfs`** — monta un recurso compartido remoto de tipo NFS en un punto de montaje local (aquí, `./lab_NFS`), como si fuera una carpeta más del sistema de ficheros local. `-o nolock` desactiva el bloqueo de ficheros NFS, evitando errores comunes cuando el servicio `rpc.statd` no está disponible en el cliente.
>
> 🛠️ **`sudo su`** — eleva a una shell de root. NFS clásico (v3) basa el control de acceso en el UID/GID del cliente, así que a menudo hace falta ser root localmente para poder leer ciertos ficheros del recurso montado tal y como los ve el servidor.

![](Imagenes/10-medio-mount-nfs-techsupport.png)
![](Imagenes/11-medio-sudo-su-cd-techsupport.png)
![](Imagenes/12-medio-ls-la-tickets-txt.png)

> 💡 Docenas de tickets de soporte, casi todos vacíos (0 B) — típico ruido para dificultar la búsqueda manual. Buscamos el que realmente tenga contenido.

Entre todos ellos, uno destaca por tener **1.3 KB** de contenido: `ticket4238791283782.txt`.

![](Imagenes/13-medio-ticket-con-contenido.png)

Su contenido es una transcripción de chat de soporte entre un empleado (`alex`) y un operador, en la que Alex pega directamente el fichero de configuración del servidor SMTP para pedir ayuda:

![](Imagenes/14-medio-ticket-chat-credenciales-alex.png)

```
host=smtp.web.dev.inlanefreight.htb
user="alex"
password="lol123!mD"
from="alex.g@web.dev.inlanefreight.htb"
```

> 💡 Un fallo de proceso humano clásico: pegar un fichero de configuración con credenciales en texto plano dentro de un ticket de soporte que queda accesible por NFS sin autenticación.

Desmontamos el recurso una vez extraída la información:

```bash
umount -l ./lab_NFS
```

> 🛠️ **`umount -l`** — desmonta un sistema de ficheros. `-l` (*lazy unmount*) lo desmonta aunque esté "ocupado" (por ejemplo, si aún hay una terminal con el directorio como *working directory*), liberando la referencia en cuanto deja de estar en uso.

![](Imagenes/15-medio-umount-nfs.png)

### 2.3 Acceso inicial — RDP con credenciales filtradas

Con las credenciales de `alex` (`lol123!mD`), probamos acceso RDP directo:

```bash
xfreerdp /u:alex /p:'lol123!mD' /v:10.129.49.180
```

> 🛠️ **`xfreerdp`** — cliente de **RDP** (Remote Desktop Protocol) para Linux. `/u:` usuario, `/p:` contraseña, `/v:` dirección del servidor. Permite validar credenciales directamente contra el escritorio remoto de Windows.

![](Imagenes/16-medio-xfreerdp-alex.png)

Acceso confirmado — escritorio de Windows 10 con **SQL Server Management Studio** instalado, señal de que este equipo aloja una base de datos MSSQL:

![](Imagenes/17-medio-escritorio-rdp-alex.png)

Explorando el sistema de ficheros de `alex`, en una carpeta compartida (`devshare`) hay un fichero de texto:

![](Imagenes/18-medio-devshare-important-txt-sa-password.png)

```
sa:87N1ns@s11s83
```

> 💡 Credenciales del usuario `sa` (System Administrator) de MSSQL guardadas en texto plano en un fichero llamado, sin ironía, "important". Además de para MSSQL, probamos si esta contraseña se reutiliza a nivel de sistema operativo.

### 2.4 Escalada — reutilización de credenciales de `sa` como Administrator

```bash
xfreerdp /u:administrator /p:'87N1ns@s11s83' /v:10.129.49.180
```

![](Imagenes/19-medio-xfreerdp-administrator-sa-password.png)

La contraseña se reutiliza y obtenemos acceso como **Administrator** del dominio local `WINMEDIUM`.

### 2.5 Post-explotación — credenciales del usuario HTB

Abrimos **SQL Server Management Studio** (SSMS) con autenticación de Windows (heredando el contexto de Administrator):

> 🛠️ **SQL Server Management Studio (SSMS)** — la herramienta gráfica oficial de Microsoft para administrar instancias de SQL Server: conectar, explorar bases de datos/tablas y ejecutar consultas SQL directamente, sin necesidad de línea de comandos.

![](Imagenes/20-medio-ssms-conexion-winmedium.png)

Consultamos la base de datos `accounts`, tabla `dbo.devsacc`, filtrando por el usuario que nos pidió el cliente:

```sql
select * from dbo.devsacc where name = 'htb';
```

> 🛠️ **`SELECT ... WHERE ...`** — sentencia SQL básica de consulta: `SELECT *` pide todas las columnas, `FROM dbo.devsacc` indica la tabla, y `WHERE name = 'htb'` filtra solo la fila cuyo campo `name` coincide con el usuario buscado, en vez de volcar toda la tabla.

![](Imagenes/21-medio-ssms-consulta-devsacc-htb-password.png)

La consulta devuelve la fila solicitada, con el usuario **`HTB`** y su contraseña almacenada en texto plano en la columna `password` de la tabla — la prueba pedida por el cliente para este laboratorio (ver tabla de preguntas al inicio de esta sección).

---

## 3. Laboratorio Difícil — Servidor SNMP/IMAP/MySQL

> **Escenario:** el tercer servidor actúa como **servidor MX y de gestión** de la red interna, además de backup de cuentas del dominio. También aquí se creó un usuario **`HTB`** cuyas credenciales hay que recuperar.

### Pregunta oficial del Skills Assessment

| Pregunta | Respuesta |
|----------|-----------|
| *Enumerate the server carefully and find the username "HTB" and its password. Then, submit HTB's password as the answer.* | 🔒 Respuesta omitida a propósito — la vía de acceso (SNMP → IMAP → SSH → MySQL) está documentada al completo en esta sección; el valor exacto de la contraseña no se publica para no regalar la respuesta del laboratorio |

### 3.1 Reconocimiento

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.192 -oN AllPorts
```

![](Imagenes/22-dificil-nmap-todos-los-puertos.png)

Solo servicios de correo por TCP: **22** (ssh), **110/995** (pop3/pop3s) y **143/993** (imap/imaps). Ampliamos con un escaneo UDP, ya que un servidor de gestión suele exponer SNMP:

```bash
nmap -sU -T5 -F --open -oN UDP 10.129.49.192
```

> 🛠️ **`-sU`** — escaneo de puertos **UDP** (a diferencia de `-sS`, que es TCP). Es más lento por naturaleza del protocolo, por eso se suele combinar con **`-F`** (*fast scan*, solo los 100 puertos UDP más comunes en vez de los 65535) para acotar el tiempo de escaneo.

![](Imagenes/23-dificil-nmap-udp-snmp.png)

Confirmado: **161/udp abierto (SNMP)**.

### 3.2 Enumeración SNMP — community string

La community string por defecto (`public`) no funciona, así que la buscamos por fuerza bruta con un diccionario de communities habituales:

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.49.192
```

> 🛠️ **`onesixtyone`** — herramienta especializada en fuerza bruta de **community strings** de SNMP (el "puerto 161" le da nombre). SNMP v1/v2c no usa usuario/contraseña como tal, sino esta cadena compartida como único control de acceso — si es débil o por defecto, cualquiera puede leer (y a veces escribir) la configuración del dispositivo.

![](Imagenes/24-dificil-onesixtyone-community-string-backup.png)

La community válida es **`backup`**. Confirma además el hostname (**NIXHARD**) y el kernel Linux. Con la community en mano, volcamos todos los OIDs disponibles:

```bash
snmpwalk -v2c -c backup 10.129.49.192
```

> 🛠️ **`snmpwalk`** — recorre (hace un *walk*) todo el árbol de **OIDs** (Object Identifiers) que expone un agente SNMP, volcando cada valor: información del sistema, interfaces de red, y en muchos casos —como aquí— la **tabla de procesos en ejecución**. `-v2c` indica la versión del protocolo (SNMPv2c), `-c backup` la community string ya descubierta.

![](Imagenes/25-dificil-snmpwalk-proceso-tom-password.png)

Entre la salida (información de contacto `Admin <tech@inlanefreight.htb>`, ubicación `Inlanefreight`, etc.) aparece la **tabla de procesos en ejecución** expuesta por el módulo `HOST-RESOURCES-MIB` de SNMP, incluyendo los **argumentos de línea de comandos** de un script:

```
iso.3.6.1.2.1.25.1.7.1.2.1.2.6.66.65.67.75.85.80 = STRING: "/opt/tom-recovery.sh"
iso.3.6.1.2.1.25.1.7.1.2.1.3.6.66.65.67.75.85.80 = STRING: "tom NMds732Js2761"
```

> 💡 Fallo clásico de SNMP: cuando SNMP puede leer la tabla de procesos del sistema (`hrSWRunParameters`), cualquier script ejecutado con la contraseña como argumento en línea de comandos (en vez de leerla de un fichero de configuración o variable de entorno) queda expuesto a cualquiera que tenga la community string de lectura. Aquí se filtran directamente unas credenciales: `tom:NMds732Js2761`.

### 3.3 Acceso inicial — IMAP con credenciales filtradas

Con `tom:NMds732Js2761`, nos conectamos al servicio IMAP sobre TLS manualmente con `openssl s_client`:

```bash
openssl s_client -connect 10.129.49.192:imaps
```

> 🛠️ **`openssl s_client -connect`** — establece una conexión TLS/SSL manual contra un servicio, útil cuando no hay un cliente dedicado a mano o se quiere interactuar directamente con el protocolo en crudo (aquí, IMAP sobre TLS en el puerto 993) para entender exactamente qué está pasando en la sesión.

![](Imagenes/26-dificil-openssl-sclient-imaps.png)

Dentro de la sesión TLS, autenticamos por el protocolo IMAP en crudo y listamos las carpetas del buzón:

```
a01 LOGIN tom NMds732Js2761
a02 LIST "" *
a03 SELECT INBOX
```

> 🛠️ **Comandos IMAP en crudo** — IMAP es un protocolo de texto plano (dentro del túnel TLS): cada comando se precede de una etiqueta arbitraria (`a01`, `a02`...) que el servidor repite en su respuesta para identificar a qué petición corresponde. `LOGIN` autentica, `LIST "" *` lista todas las carpetas del buzón, `SELECT <carpeta>` la abre para poder operar sobre sus mensajes.

![](Imagenes/27-dificil-imap-login-tom-listado-carpetas.png)

Login correcto. El buzón tiene carpetas `Notes`, `Meetings`, `Important` e `INBOX` (con 1 mensaje). Recuperamos el mensaje:

```
a05 FETCH 1 BODY[]
```

> 🛠️ **`FETCH <id> BODY[]`** — comando IMAP que descarga el contenido completo (cabeceras + cuerpo) de un mensaje concreto de la carpeta seleccionada, identificado por su número de secuencia (`1` = el primer mensaje).

![](Imagenes/28-dificil-imap-fetch-correo-clave-privada.png)

El correo, de asunto **"KEY"**, enviado por `tech@dev.inlanefreight.htb` a `tom@inlanefreight.htb`, contiene en el cuerpo una **clave privada SSH completa** (bloque `-----BEGIN OPENSSH PRIVATE KEY-----`).

### 3.4 Obtención de shell

```bash
chmod 600 id_rsa
ssh -i id_rsa tom@10.129.49.192
```

![](Imagenes/29-dificil-ssh-tom-clave-privada.png)

Acceso confirmado como `tom` en el host **NIXHARD**.

### 3.5 Post-explotación — MySQL y credenciales de HTB

```bash
cat /etc/passwd
```

> 🛠️ **`/etc/passwd`** — fichero de Linux con la lista de cuentas del sistema (usuario, UID, GID, home, shell asignada). No contiene contraseñas (eso está en `/etc/shadow`, no legible sin privilegios), pero es clave para saber **qué usuarios tienen shell interactiva** y cuáles son solo cuentas de servicio.

![](Imagenes/30-dificil-etc-passwd.png)

Además de las cuentas de sistema habituales, hay tres usuarios con shell interactiva: `ubuntu`, **`cry0llt3`** (el mismo usuario visto en el laboratorio Fácil) y **`tom`**. También se observa el servicio `mysql` instalado.

Hay MySQL corriendo localmente, así que probamos a reutilizar de nuevo la contraseña de `tom` que ya conocíamos por SNMP:

```bash
mysql -u tom -p
```

> 🛠️ **`mysql -u <usuario> -p`** — cliente de línea de comandos de MySQL/MariaDB. `-u` indica el usuario de la base de datos, y `-p` (sin valor pegado) hace que pida la contraseña de forma interactiva, evitando dejarla visible en el historial de comandos.

![](Imagenes/31-dificil-mysql-login-tom.png)

La contraseña se reutiliza y accedemos al monitor de MySQL. Listamos las bases de datos disponibles:

```sql
show databases;
use users;
show tables;
select * from users;
```

> 🛠️ **`show databases` / `use <bd>` / `show tables`** — el flujo de exploración estándar en MySQL: listar qué bases de datos existen, seleccionar una como contexto activo (`use`), y listar qué tablas contiene. `select * from <tabla>` vuelca todas las filas y columnas de una tabla concreta.

![](Imagenes/32-dificil-mysql-show-databases-users.png)

La base de datos `users`, tabla `users`, contiene una lista de credenciales (`id`, `username`, `password`) de múltiples cuentas del dominio. Entre ellas debería estar la fila correspondiente al usuario **`HTB`** solicitado por el cliente.

> 💡 La captura disponible muestra el inicio del volcado de la tabla (`ppavlata0`, `ktofanini1`, `rallwell2`, `efernier3`, `fpoon4`, `jgurnell5`...). Una consulta filtrada la habría aislado directamente:
>
> ```sql
> select * from users where username = 'htb';
> ```
>
> 🔒 Como se indica en la tabla de preguntas de esta sección, el valor exacto de la contraseña de `HTB` no se publica en este writeup a propósito.

---

## 4. Lección aprendida

| Vulnerabilidad | Dónde | Impacto |
|----------------|-------|---------|
| Credenciales de un empleado reutilizadas literalmente para su propio servicio FTP en un puerto no estándar | Laboratorio Fácil | Un puerto "oculto" en 2121 no protege nada si las credenciales ya se conocían |
| Clave privada SSH accesible sin restricciones vía FTP | Laboratorio Fácil | Compromiso directo de la cuenta sin necesidad de crackear ni fuerza bruta |
| Recurso NFS exportado sin restricción de acceso (`no_root_squash` / sin autenticación) | Laboratorio Medio | Cualquier cliente en la red puede leer contenido interno, incluidos tickets de soporte con secretos |
| Configuración de servicio (SMTP) con contraseña en texto plano pegada en un ticket de soporte | Laboratorio Medio | Exposición de credenciales por error humano, no por fallo técnico directo |
| Contraseña de la cuenta `sa` de MSSQL reutilizada como contraseña de `Administrator` del sistema operativo | Laboratorio Medio | Una sola credencial filtrada compromete tanto la base de datos como el servidor completo |
| SNMP con community string débil, adivinable por diccionario | Laboratorio Difícil | Acceso de lectura a información sensible del sistema sin autenticación fuerte |
| Contraseña pasada como argumento de línea de comandos a un script | Laboratorio Difícil | Cualquier mecanismo que pueda leer la tabla de procesos (SNMP, `/proc`, `ps`) expone la credencial |
| Clave privada SSH completa enviada por correo electrónico en texto plano | Laboratorio Difícil | El correo es un canal no cifrado de extremo a extremo por defecto; cualquiera con acceso al buzón obtiene la clave |
| Reutilización de contraseñas entre servicios (SNMP → MySQL) y entre entornos (mismos usuarios `cry0llt3` en dos laboratorios distintos) | Los tres laboratorios | Una sola credencial débil o filtrada se propaga en cascada por todo el entorno |

## Recomendaciones defensivas

- No reutilizar credenciales entre servicios, cuentas de sistema y entornos distintos.
- Restringir el acceso a recursos NFS por IP/rango y exigir autenticación cuando el contenido pueda ser sensible.
- No pegar configuraciones con secretos en tickets de soporte, chats o cualquier sistema sin control de acceso estricto.
- Nunca pasar contraseñas como argumento de línea de comandos en scripts — usar variables de entorno, ficheros de configuración con permisos restringidos o gestores de secretos.
- Cambiar las community strings de SNMP por defecto y usar SNMPv3 con autenticación y cifrado en vez de SNMPv1/v2c.
- No enviar claves privadas por correo electrónico; si es imprescindible, cifrarlas (GPG) y transmitir la passphrase por un canal distinto.
- Auditar periódicamente los recursos compartidos (FTP, NFS, SMB) en busca de contenido sensible olvidado o mal permisado.
- Aplicar el principio de menor privilegio: la cuenta `sa` de MSSQL no debería tener ninguna relación con las credenciales de administración del sistema operativo.

---

## 5. Glosario de herramientas y comandos

Referencia rápida de todo lo usado en este módulo, pensada para repasar antes del examen CPTS:

| Herramienta / comando | Para qué sirve |
|------------------------|-----------------|
| `nmap` | Escanear puertos, servicios y versiones de un host o red |
| `nmap -sU` | Escanear puertos UDP (más lento; combinar con `-F` para acotar tiempo) |
| `nmap --script nfs*` | Enumerar recursos NFS exportados sin necesidad de montarlos |
| `ftp` | Cliente de línea de comandos para transferir ficheros por FTP |
| `chmod 600` | Restringir permisos de un fichero (obligatorio en claves privadas SSH) |
| `ssh -i <clave>` | Conectarse por SSH usando una clave privada concreta |
| `mount -t nfs` / `umount -l` | Montar/desmontar un recurso compartido NFS en el sistema local |
| `sudo su` | Obtener una shell de root |
| `xfreerdp` | Cliente RDP para conectarse a escritorios remotos Windows |
| **SQL Server Management Studio (SSMS)** | Herramienta gráfica de Microsoft para administrar y consultar bases de datos SQL Server |
| `SELECT ... WHERE ...` (SQL) | Consultar filas concretas de una tabla en vez de volcarla entera |
| **`onesixtyone`** | Fuerza bruta de community strings de SNMP |
| **`snmpwalk`** | Volcar el árbol completo de OIDs expuesto por un agente SNMP |
| `openssl s_client -connect` | Establecer una conexión TLS/SSL manual para interactuar con un protocolo en crudo |
| Comandos IMAP (`LOGIN`, `LIST`, `SELECT`, `FETCH`) | Autenticarse, listar carpetas y leer mensajes de un buzón por IMAP sin cliente gráfico |
| `cat /etc/passwd` | Ver las cuentas de un sistema Linux y qué usuarios tienen shell interactiva |
| `mysql -u <user> -p` | Cliente de línea de comandos para conectarse a MySQL/MariaDB |
| `show databases` / `use` / `show tables` (SQL) | Flujo de exploración estándar de una instancia MySQL |

---

*Apuntes de CPTS por [Arabot](https://github.com/Caan31) · HTB Academy · Módulo Footprinting · 2026*
