<div align="center">

```
 ██████╗██████╗ ████████╗███████╗
██╔════╝██╔══██╗╚══██╔══╝██╔════╝
██║     ██████╔╝   ██║   ███████╗
██║     ██╔═══╝    ██║   ╚════██║
╚██████╗██║        ██║   ███████║
 ╚═════╝╚═╝        ╚═╝   ╚══════╝
```

**`Certified Penetration Testing Specialist · Cheatsheets & Notas`**

[![Cert](https://img.shields.io/badge/Certificaci%C3%B3n-CPTS-ff6b35?style=flat-square&logo=hackthebox&logoColor=white)](https://academy.hackthebox.com/preview/certifications/htb-certified-penetration-testing-specialist)
[![Academy](https://img.shields.io/badge/HTB-Academy-9fef00?style=flat-square&logo=hackthebox&logoColor=black)](https://academy.hackthebox.com/)
[![Estado](https://img.shields.io/badge/Estado-En%20progreso-00ff88?style=flat-square)](#)
[![Idioma](https://img.shields.io/badge/Idioma-ES%20%7C%20EN-9fef00?style=flat-square)](#)

</div>

---

## `$ whoami`

```bash
> Repo      :  CPTS — Cheatsheets, laboratorios y máquinas de práctica
> Autor     :  Arabot
> Objetivo  :  Preparar y aprobar el HTB Certified Penetration Testing Specialist (CPTS)
> Formato   :  Notas Obsidian → Markdown limpio para GitHub + writeups con capturas
> Idioma    :  Español 🇪🇸 + English 🇬🇧 (cada documento, en dos ficheros separados)
> Estado    :  [ Aprendiendo en público · Siempre en progreso ]
```

> Notas, laboratorios de HTB Academy y máquinas de práctica adicionales, pensados para:
> - **Repaso rápido** antes y durante la práctica de cada módulo — y durante el propio examen.
> - **Material público** para cualquiera preparando el CPTS, en español o en inglés.
> - **Referencia personal** — documentando la metodología completa y el *porqué* de cada comando/herramienta, no solo el *qué*.

---

## `$ cat sobre_el_cpts.txt`

El **HTB Certified Penetration Testing Specialist (CPTS)** de Hack The Box Academy es una certificación práctica centrada en pentesting de infraestructura y aplicaciones web. Se prepara siguiendo el *job-role path* "Penetration Tester" de HTB Academy, módulo a módulo, cada uno con sus propios laboratorios guiados y un *Skills Assessment* final sin pistas — complementado con la resolución de máquinas reales de HTB para coger soltura fuera del entorno guiado de Academy.

```
🎯  Modalidad   →  100% práctico (examen de 10 días + informe profesional)
🧰  Entorno     →  Kali Linux + labs de HTB Academy (Pwnbox o VPN propia)
📚  Temario     →  Redes, enumeración, web, AD, pivoting, informática forense básica
🏆  Aprobado    →  Compromiso completo del entorno de examen + informe aceptado
```

---

## `$ ls modulos/`

> Cada módulo completado incluye su carpeta con el writeup del *Skills Assessment* **en español e inglés** (documentos separados), con capturas, metodología paso a paso y una breve explicación de para qué sirve cada comando/herramienta usado — pensado tanto para repasar antes del examen como para ayudar a otras personas cursando el path. Marcados con `✅` los terminados y `⏳` los pendientes.

| Módulo | Estado | Contenido | ES | EN |
|--------|:-:|-----------|:--:|:--:|
| Footprinting | ✅ | 3 laboratorios (Fácil/Medio/Difícil): DNS/FTP/SSH · NFS/RDP/MSSQL · SNMP/IMAP/MySQL | [📄 Ver](./Footprinting/Footprinting_Writeup_ES.md) | [📄 View](./Footprinting/Footprinting_Writeup_EN.md) |
| Shells & Payloads | ✅ | Foothold + 3 hosts: Tomcat WAR upload (Windows) · RCE autenticado en blog PHP (Linux, Metasploit) · EternalBlue/MS17-010 · pivoting con Chisel | [📄 Ver](./Shells%20%26%20Payloads/Shells_and_Payloads_Writeup_ES.md) | [📄 View](./Shells%20%26%20Payloads/Shells_and_Payloads_Writeup_EN.md) |

> 📌 El resto de módulos del path (Network Enumeration with Nmap, Information Gathering, Vulnerability Assessment, Web Attacks, Active Directory, Pivoting...) se irán añadiendo aquí conforme los vaya completando — prefiero no listar módulos que aún no he hecho para no prometer un temario que no he verificado yo mismo.
>
> 📌 Cada pregunta oficial del *Skills Assessment* se documenta junto a su sección correspondiente, pero las respuestas de tipo "flag" (contraseñas o contenidos de ficheros a enviar como prueba) se omiten a propósito — se indica siempre dónde encontrarlas, pero no el valor exacto, para no darle la respuesta hecha a quien esté cursando el módulo.

---

## `$ ls maquinas/`

> Máquinas reales de HTB resueltas como práctica adicional al margen de los módulos guiados de Academy — mismo formato de writeup (capturas, explicación de cada comando/herramienta, glosario), en español e inglés. Estas mismas máquinas también se publican, en versión solo-español y con el formato estándar de writeup, en el repositorio [HackTheBox-Writeups-by-Arabot](https://github.com/Caan31/-HackTheBox-Writeups-by-Arabot).

| Máquina | SO | Dificultad | Técnicas principales | ES | EN |
|---------|:--:|:----------:|-----------------------|:--:|:--:|
| Access | 🪟 | Fácil | FTP anónimo · BD Access (`mdbtools`) · ZIP con contraseña reutilizada · PST (`readpst`) · Telnet · `runas /savecred` cacheado → Administrator | [📄 Ver](./Maquinas/Access/Access_Writeup_ES.md) | [📄 View](./Maquinas/Access/Access_Writeup_EN.md) |
| Delivery | 🐧 | Fácil | osTicket · Mattermost · Verificación de email vía ticket de soporte (buzón cruzado) · Credenciales filtradas en chat interno · LinPEAS · `config.json` de Mattermost · MySQL · Hashcat (`best66`) + John → root | [📄 Ver](./Maquinas/Delivery/Delivery_Writeup_ES.md) | [📄 View](./Maquinas/Delivery/Delivery_Writeup_EN.md) |
| Driver | 🪟 | Fácil | Credenciales por defecto · Fichero `.scf` → captura de hash NTLMv2 con Responder · John · Evil-WinRM · WinPEAS · **PrintNightmare (CVE-2021-1675)** → Administrator | [📄 Ver](./Maquinas/Driver/Driver_Writeup_ES.md) | [📄 View](./Maquinas/Driver/Driver_Writeup_EN.md) |

> 📌 Igual que en los módulos, las respuestas de tipo flag (`user.txt`/`root.txt`) se omiten a propósito.

---

## `$ tree .`

```
CPTS/
├── README.md                                    ← este índice
├── Footprinting/
│   ├── Footprinting_Writeup_ES.md                 ← los 3 laboratorios del módulo (español)
│   ├── Footprinting_Writeup_EN.md                 ← los 3 laboratorios del módulo (English)
│   └── Imagenes/                                 ← capturas compartidas por ambas versiones
├── Shells & Payloads/
│   ├── Shells_and_Payloads_Writeup_ES.md          ← foothold + 3 hosts (español)
│   ├── Shells_and_Payloads_Writeup_EN.md          ← foothold + 3 hosts (English)
│   └── Imagenes/                                 ← capturas compartidas por ambas versiones
└── Maquinas/
    ├── Access/
    │   ├── Access_Writeup_ES.md                   ← máquina Access (español)
    │   ├── Access_Writeup_EN.md                   ← máquina Access (English)
    │   └── Imagenes/                              ← capturas compartidas por ambas versiones
    ├── Delivery/
    │   ├── Delivery_Writeup_ES.md                 ← máquina Delivery (español)
    │   ├── Delivery_Writeup_EN.md                 ← máquina Delivery (English)
    │   └── Imagenes/                              ← capturas compartidas por ambas versiones
    └── Driver/
        ├── Driver_Writeup_ES.md                   ← máquina Driver (español)
        ├── Driver_Writeup_EN.md                   ← máquina Driver (English)
        └── Imagenes/                              ← capturas compartidas por ambas versiones
```

---

## `$ cat metodologia.txt`

Cada writeup (módulo o máquina) sigue la misma estructura, con un extra pensado específicamente para el estudio del CPTS: **cada comando y cada herramienta llevan una breve explicación de para qué sirven**, con el símbolo 🛠️, para que el documento funcione también como cheatsheet de repaso y no solo como registro de lo que se hizo.

```
1.  📡  Reconocimiento     →  Nmap (TCP completo + versión/scripts, UDP si aplica)
2.  🔬  Enumeración        →  por servicio expuesto (FTP, NFS, SMB, SNMP, IMAP, HTTP...)
3.  🔑  Correlación        →  cruzar credenciales/pistas encontradas entre servicios
4.  🪜  Acceso              →  con las credenciales, exploit o payload correspondiente
5.  🎒  Post-explotación   →  localizar y acreditar la prueba pedida (flag/credencial)
6.  📋  Lección aprendida  →  qué falló y cómo se mitigaría en un entorno real
7.  🛠️  Glosario           →  tabla resumen de herramientas/comandos usados, para repaso rápido
```

> ⚠️ Algunos módulos de HTB Academy (como Footprinting) exigen explícitamente **no explotar agresivamente** los servicios — el objetivo es enumerar y correlacionar información. Otros, como Shells & Payloads o las máquinas reales de HTB, sí requieren explotación activa (esa distinción se indica en cada writeup).

---

## `$ cat aviso.txt`

> ⚠️ **Material de estudio personal.** Las técnicas descritas solo deben usarse contra los laboratorios oficiales de **HTB Academy**, máquinas **retiradas** de Hack The Box, o sistemas para los que se tenga autorización explícita por escrito. El uso contra sistemas de terceros sin permiso es **ilegal**.

---

## `$ cat contacto.txt`

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-Caan31-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Caan31)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Conectar-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/carlos-andres-aragon-nacimba)
[![HackTheBox](https://img.shields.io/badge/HackTheBox-Perfil-9fef00?style=for-the-badge&logo=hackthebox&logoColor=black)](https://profile.hackthebox.com/profile/019d926f-f2ef-7360-bbab-d3551b8aa9b5)

</div>

---

<div align="center">

```
[ Aprendiendo en público · Cheatsheet viva · Se actualiza tras cada módulo ]
```

*Si te sirve para preparar tu CPTS, dale una ⭐ al repo — significa mucho.*

</div>
