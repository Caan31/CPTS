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
[![Idioma](https://img.shields.io/badge/Idioma-Espa%C3%B1ol-9fef00?style=flat-square)](#)

</div>

---

## `$ whoami`

```bash
> Repo      :  CPTS — Cheatsheets y laboratorios personales
> Autor     :  Arabot
> Objetivo  :  Preparar y aprobar el HTB Certified Penetration Testing Specialist (CPTS)
> Formato   :  Notas Obsidian → Markdown limpio para GitHub + writeups de labs con capturas
> Idioma    :  Español 🇪🇸  (contenido 100% en castellano)
> Estado    :  [ Aprendiendo en público · Siempre en progreso ]
```

> Notas y laboratorios organizados por módulo de HTB Academy, pensados para:
> - **Repaso rápido** antes y durante la práctica de cada módulo.
> - **Material público** para cualquiera preparando el CPTS en español.
> - **Referencia personal** — documentando la metodología completa, no solo comandos sueltos.

---

## `$ cat sobre_el_cpts.txt`

El **HTB Certified Penetration Testing Specialist (CPTS)** de Hack The Box Academy es una certificación práctica centrada en pentesting de infraestructura y aplicaciones web. Se prepara siguiendo el *job-role path* "Penetration Tester" de HTB Academy, módulo a módulo, cada uno con sus propios laboratorios guiados y un *Skills Assessment* final sin pistas.

```
🎯  Modalidad   →  100% práctico (examen de 10 días + informe profesional)
🧰  Entorno     →  Kali Linux + labs de HTB Academy (Pwnbox o VPN propia)
📚  Temario     →  Redes, enumeración, web, AD, pivoting, informática forense básica
🏆  Aprobado    →  Compromiso completo del entorno de examen + informe aceptado
```

---

## `$ ls modulos/`

> Cada módulo completado incluye su carpeta con el writeup del *Skills Assessment* (capturas + metodología explicada paso a paso, igual que mis writeups de máquinas). Marcados con `✅` los terminados y `⏳` los pendientes.

| Módulo | Estado | Contenido | Enlace |
|--------|:-:|-----------|:------:|
| Footprinting | ✅ | 3 laboratorios (Fácil/Medio/Difícil): DNS/FTP/SSH · NFS/RDP/MSSQL · SNMP/IMAP/MySQL | [📄 Ver](./Footprinting/Footprinting_Writeup.md) |

> 📌 El resto de módulos del path (Network Enumeration with Nmap, Information Gathering, Vulnerability Assessment, Web Attacks, Active Directory, Pivoting...) se irán añadiendo aquí conforme los vaya completando — prefiero no listar módulos que aún no he hecho para no prometer un temario que no he verificado yo mismo.

---

## `$ tree .`

```
CPTS/
├── README.md                              ← este índice
└── Footprinting/
    ├── Footprinting_Writeup.md             ← los 3 laboratorios del módulo
    └── Imagenes/                           ← capturas de los 3 laboratorios
```

---

## `$ cat metodologia.txt`

Cada writeup de módulo sigue la misma estructura que mis writeups de máquinas de HTB/DockerLabs:

```
1.  📡  Reconocimiento     →  Nmap (TCP completo + versión/scripts, UDP si aplica)
2.  🔬  Enumeración        →  por servicio expuesto (FTP, NFS, SMB, SNMP, IMAP...)
3.  🔑  Correlación        →  cruzar credenciales/pistas encontradas entre servicios
4.  🪜  Acceso              →  con las credenciales o claves obtenidas
5.  🎒  Post-explotación   →  localizar y acreditar la prueba pedida (flag/credencial)
6.  📋  Lección aprendida  →  qué falló y cómo se mitigaría en un entorno real
```

> ⚠️ Los laboratorios de HTB Academy exigen explícitamente **no explotar agresivamente** los servicios en los ejercicios de footprinting — el objetivo es enumerar y correlacionar información, no lanzar exploits.

---

## `$ cat aviso.txt`

> ⚠️ **Material de estudio personal.** Las técnicas descritas solo deben usarse contra los laboratorios oficiales de **HTB Academy** o sistemas para los que se tenga autorización explícita por escrito. El uso contra sistemas de terceros sin permiso es **ilegal**.

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
