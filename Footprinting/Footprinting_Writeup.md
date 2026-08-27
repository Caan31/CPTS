# Footprinting — HTB Academy (CPTS)

**Plataforma:** Hack The Box Academy
**Módulo:** Footprinting
**Tipo:** Skills Assessment (3 laboratorios independientes: Fácil, Medio, Difícil)
**Certificación:** CPTS (Certified Penetration Testing Specialist)
**Fecha de resolución:** 2026

---

## Contexto del ejercicio

La empresa ficticia **Inlanefreight Ltd** encarga una prueba de penetración de reconocimiento (footprinting) sobre tres servidores distintos de su red interna. El objetivo en los tres casos es el mismo: enumerar exhaustivamente cada servicio, identificar qué información puede extraerse de él y cómo esa información puede encadenarse para obtener acceso, **sin explotar activamente vulnerabilidades** (los servicios están en producción). Los compañeros de equipo aportan una pista inicial común a los tres laboratorios: las credenciales `ceil:qwer1234` y el rumor de que algunos empleados hablan de claves SSH en un foro.

## Índice
1. [Laboratorio Fácil — Servidor DNS/FTP/SSH](#1-laboratorio-fácil--servidor-dnsftpssh)
2. [Laboratorio Medio — Servidor NFS/RDP/MSSQL](#2-laboratorio-medio--servidor-nfsrdpmssql)
3. [Laboratorio Difícil — Servidor SNMP/IMAP/MySQL](#3-laboratorio-difícil--servidor-snmpimapmysql)
4. [Lección aprendida](#4-lección-aprendida)

---

## 1. Laboratorio Fácil — Servidor DNS/FTP/SSH

**Objetivo:** el cliente señala este primer servidor como un **servidor DNS interno** y pide averiguar toda la información posible sobre él. Los administradores dejaron un `flag.txt` en el servidor para acreditar el compromiso.

### 1.1 Reconocimiento

Escaneo completo de puertos TCP:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.177 -oN AllPorts
```

![](Imagenes/01-facil-nmap-todos-los-puertos.png)

Cuatro puertos abiertos: **21** (ftp), **22** (ssh), **53** (domain) y **2121** (un segundo FTP en puerto no estándar). Escaneo de versión y scripts por defecto:

```bash
nmap -sS -Pn -sCV -T5 -n -p21,22,53,2121 10.129.49.177 -oN Ports
```

![](Imagenes/02-facil-nmap-version-servicios.png)

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 21 | ProFTPD | Banner `ftp.int.inlanefreight.htb` |
| 22 | OpenSSH 8.2p1 (Ubuntu) | — |
| 53 | ISC BIND 9.16.1 (Ubuntu) | — |
| 2121 | ProFTPD | Banner **"Ceil's FTP"** |

> 💡 El segundo servicio FTP en el puerto 2121 se identifica explícitamente como el FTP de **Ceil** en su banner — encaja directamente con las credenciales `ceil:qwer1234` que nos había pasado el equipo.

### 1.2 Acceso inicial — FTP con credenciales filtradas

Nos conectamos al FTP del puerto filtrado (2121) con las credenciales conocidas:

```bash
ftp 10.129.49.177 2121
Name: ceil
Password: qwer1234
```

![](Imagenes/03-facil-ftp-2121-login-ceil.png)

Dentro del `home` de `ceil` hay una carpeta `.ssh`. La listamos y descargamos su clave privada:

```
ftp> cd .ssh
ftp> ls -la
ftp> get id_rsa
```

![](Imagenes/04-facil-ftp-ssh-carpeta-descarga-id-rsa.png)

> 💡 Confirma la pista del equipo sobre "claves SSH": la propia cuenta `ceil` tiene su clave privada expuesta vía FTP, con `authorized_keys` presente también en la misma carpeta — es su propia clave de acceso.

### 1.3 Obtención de shell

Con la clave privada descargada, nos conectamos por SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa ceil@10.129.49.177
```

![](Imagenes/05-facil-ssh-login-ceil-clave-privada.png)

Acceso confirmado como `ceil` en el host **NIXEASY**.

### 1.4 Post-explotación y flag

```bash
cd /home
ls
```

![](Imagenes/06-facil-home-flag-cat-flag-txt.png)

Además de `ceil`, hay otros dos directorios: **`cry0llt3`** (otro usuario del sistema) y **`flag`**. Entramos en este último y mostramos el fichero solicitado:

```bash
cd flag
cat flag.txt
```

> 💡 Las capturas disponibles muestran la ejecución de `cat flag.txt` pero no llegan a capturar la línea de salida con el contenido del flag, así que no se documenta aquí un valor inventado — el procedimiento hasta la lectura del flag queda acreditado igualmente.

---

## 2. Laboratorio Medio — Servidor NFS/RDP/MSSQL

**Objetivo:** un segundo servidor, accesible para todos en la red interna, por lo que es un objetivo prioritario típico de atacantes reales. Aquí se creó un usuario **`HTB`** cuyas credenciales hay que recuperar como prueba.

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

![](Imagenes/15-medio-umount-nfs.png)

### 2.3 Acceso inicial — RDP con credenciales filtradas

Con las credenciales de `alex` (`lol123!mD`), probamos acceso RDP directo:

```bash
xfreerdp /u:alex /p:'lol123!mD' /v:10.129.49.180
```

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

Abrimos SQL Server Management Studio con autenticación de Windows (heredando el contexto de Administrator):

![](Imagenes/20-medio-ssms-conexion-winmedium.png)

Consultamos la base de datos `accounts`, tabla `dbo.devsacc`, filtrando por el usuario que nos pidió el cliente:

```sql
select * from dbo.devsacc where name = 'htb';
```

![](Imagenes/21-medio-ssms-consulta-devsacc-htb-password.png)

La consulta devuelve la fila solicitada, con el usuario **`HTB`** y su contraseña almacenada en texto plano en la columna `password` de la tabla — la prueba pedida por el cliente para este laboratorio.

---

## 3. Laboratorio Difícil — Servidor SNMP/IMAP/MySQL

**Objetivo:** el tercer servidor actúa como **servidor MX y de gestión** de la red interna, además de backup de cuentas del dominio. También aquí se creó un usuario **`HTB`** cuyas credenciales hay que recuperar.

### 3.1 Reconocimiento

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 10.129.49.192 -oN AllPorts
```

![](Imagenes/22-dificil-nmap-todos-los-puertos.png)

Solo servicios de correo por TCP: **22** (ssh), **110/995** (pop3/pop3s) y **143/993** (imap/imaps). Ampliamos con un escaneo UDP, ya que un servidor de gestión suele exponer SNMP:

```bash
nmap -sU -T5 -F --open -oN UDP 10.129.49.192
```

![](Imagenes/23-dificil-nmap-udp-snmp.png)

Confirmado: **161/udp abierto (SNMP)**.

### 3.2 Enumeración SNMP — community string

La community string por defecto (`public`) no funciona, así que la buscamos por fuerza bruta con un diccionario de communities habituales:

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.49.192
```

![](Imagenes/24-dificil-onesixtyone-community-string-backup.png)

La community válida es **`backup`**. Confirma además el hostname (**NIXHARD**) y el kernel Linux. Con la community en mano, volcamos todos los OIDs disponibles:

```bash
snmpwalk -v2c -c backup 10.129.49.192
```

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

![](Imagenes/26-dificil-openssl-sclient-imaps.png)

Dentro de la sesión TLS, autenticamos por el protocolo IMAP en crudo y listamos las carpetas del buzón:

```
a01 LOGIN tom NMds732Js2761
a02 LIST "" *
a03 SELECT INBOX
```

![](Imagenes/27-dificil-imap-login-tom-listado-carpetas.png)

Login correcto. El buzón tiene carpetas `Notes`, `Meetings`, `Important` e `INBOX` (con 1 mensaje). Recuperamos el mensaje:

```
a05 FETCH 1 BODY[]
```

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

![](Imagenes/30-dificil-etc-passwd.png)

Además de las cuentas de sistema habituales, hay tres usuarios con shell interactiva: `ubuntu`, **`cry0llt3`** (el mismo usuario visto en el laboratorio Fácil) y **`tom`**. También se observa el servicio `mysql` instalado.

Hay MySQL corriendo localmente, así que probamos a reutilizar de nuevo la contraseña de `tom` que ya conocíamos por SNMP:

```bash
mysql -u tom -p
```

![](Imagenes/31-dificil-mysql-login-tom.png)

La contraseña se reutiliza y accedemos al monitor de MySQL. Listamos las bases de datos disponibles:

```sql
show databases;
use users;
show tables;
select * from users;
```

![](Imagenes/32-dificil-mysql-show-databases-users.png)

La base de datos `users`, tabla `users`, contiene una lista de credenciales (`id`, `username`, `password`) de múltiples cuentas del dominio. Entre ellas debería estar la fila correspondiente al usuario **`HTB`** solicitado por el cliente.

> 💡 La captura disponible muestra el inicio del volcado de la tabla (`ppavlata0`, `ktofanini1`, `rallwell2`, `efernier3`, `fpoon4`, `jgurnell5`...) pero no llega a capturar la fila específica del usuario `htb`, así que no se documenta aquí un valor inventado — el procedimiento hasta la consulta que la contiene queda acreditado igualmente. Una consulta filtrada (`select * from users where username='htb';`) habría aislado directamente esa fila.

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

*Apuntes de CPTS por [Arabot](https://github.com/Caan31) · HTB Academy · Módulo Footprinting · 2026*
