---
title: Blackfield - Write Up (HTB) ❤
author: sha16
categories: [HackTheBox, Active Directory]
tags: [AS-REP Roasting, Abusing Backup Operators, Abusing ForceChangePassword, ActiveDirectory, Dumping LSASS through a dump file (Pypykatz)]
---

Buenas a tod@s, el día de hoy les traigo un `write-up` de la máquina `Blackfield` de `HackTheBox`.  

![Untitled](/assets/img/htb/machines/Blackfield/Untitled.png)

Como en cada `write-up`, lo primero hacemos es lanzar un escaneo de puertos con `nmap` con el objetivo de identificar potenciales vías de ataque. 

Como podrán ver a continuación, se específica el parámetro `-sS` con el objetivo de que el escaneo sea vía `TCP SYNC`, lo que de cierto modo me permitirá agilizarlo, debido a que no hay `Three-Way-HandShake` como en `TCP`.

Lo otro es que se configura el `--min-rate` en `3000` con la idea de no enviar una cantidad menor a `3000` paquetes por segundo.

**Advertencia:** No aconsejo lanzar este escaneo en entornos productivos debido a que es ultra ruidoso y agresivo. En este caso lo estoy lanzando en un entorno controlado de `HackTheBox`.

```bash
nmap -v -sS --min-rate 3000 -n -Pn -p- 10.10.10.192 -oG tcp_ports.txt
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%201.png)

En el resultado del escaneo, se identifican varios servicios de interés, entre ellos: WinRPC, SMB, LDAP, WinRM y Kerberos. Sin embargo, ya con sólo ver al servicio `Kerberos`, podemos deducir que estamos ante un entorno de *Active Directory*. Por lo que ya podemos saber más o menos cómo enforcar nuestra estrategia de ataque y enumeración.

```bash
nmap -v -sCV -p53,88,135,389,445,593,3268,5985 10.10.10.192 -oN scan_ports.txt
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%202.png)

Posterior a esto, enumeramos los recursos compartidos a través el servicio SMB, aprovechando el *null session*.

```bash
crackmapexec smb blackfield.local -u 'null' -p '' --shares
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%203.png)

Como se puede ver en imagen, sobre el recurso `$IPC` directamente no encontramos archivos o directorios importantes.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%204.png)

Sin embargo, en el recurso `profiles$` identificamos un listado extenso de carpetas con nombres de usuarios.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%205.png)

Dado que estamos ante un entorno AD, agarramos todos los nombres de usuarios y, con la herramienta `kerbrute` validamos si efectivamente estos se encuentran registrados en el dominio.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%206.png)

Una vez identificados los usuarios del dominio, lanzamos un *ASREP-Roasting Attack* con el cual logramos solicitar el `Ticket Granting Ticket (TGT)` del usuario `support`, debido a que este tiene habilitado el privilegio `UF_DONT_REQUIRE_PREAUTH`.

```bash
impacket-GetNPUsers -no-pass -usersfile domain_users.txt blackfield.local/
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%207.png)

Posterior a ello, teniendo en cuenta que el `TGT` se encuentra cifrado con la contraseña del usuario, y haciendo uso de la herramienta `john`, logramos crackear dicho hash exitosamente.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%208.png)

Considerando que tenemos a más de un usuario en mano, lanzamos un *Password Sprying* con el objetivo de verificar que si la contraseña pertenece a más de un usuario en el dominio.

```bash
crackmapexec smb blackfield.local -u domain_users.txt -p '#00^BlackKnight' --continue-on-success
```

En este caso, al lanzar el ataque, validamos que dicha contraseña pertenece al usuario `support`, únicamente.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%209.png)

Como se puede observar en imagen, también verificamos que la contraseña del usuario es válida sólo sobre el servicio SMB. 

En este caso no lo es sobre WinRM, debido a que `support` no es parte del grupo `Remote Management Users`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2010.png)

Como se observa en la imagen, al enumerar los recursos compartidos con las credenciales del usuario `support` logramos identificar que este tiene privilegios de lectura sobre el `NETLOGON` y el `SYSVOL`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2011.png)

Al acceder al `NETLOGON` no se encuentran directorios ni archivos. Sin embargo, en el `SYSVOL` si, lo cual nos podría permitir obtener credenciales a través del archivo `Groups.xml` del *Group Policy*. 

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2012.png)

Para esto, hacemos una montura sobre nuestro sistema del recurso `SYSVOL`, sin embargo, posterior a esto no encontramos el archivo `Groups.xml`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2013.png)

Dado que en el servicio SMB no existen archivos de completo interés, lanzamos la herramienta `BloodHound-Python` con la cual, enumeramos ampliamente la estructura de permisos y accesos configurados en el directorio activo.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2014.png)

Al analizar el resultado obtenido observamos que el usuario `support` tiene el permiso `ForceChangePassword` sobre el usuario `audit2020`, lo cual significa que `support` le puede cambiar la contraseña a `audit2020`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2015.png)

Para ejecutar dicha acción desde Linux, hacemos uso de la herramienta `net rpc`, con la cual logramos cambiarle la contraseña al usuario exitosamente.  

```bash
net rpc password "audit2020" "Admin12345$" -U "Blackfield.Local"/"support"%"#00^BlackKnight" -S "10.10.10.192
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2016.png)

Al enumerar los recursos compartidos, sobre los cuales tiene acceso `audit2020`, podemos ver que puede leer el contenido de `forensic`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2017.png)

En `forensic` se identifican tres directorios, los cuales parecen ser el resultado de un análisis forense ejecutado sobre el servidor.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2018.png)

En la carpeta `memory_analysis`, donde están los resultados de lo que debió ser el resultado de un análisis de la memoria en tiempo de ejecución, se identifica un comprimido interesante que corresponde al `lsass.zip`.

En principio esto es interesante, ya que el **Local Security Authority SubSystem (LSASS)** es un proceso que contiene a todos los **Security Service Providers (SSP),** los cuales son paquetes encargados específicamente de gestionar distintos de procesos de autenticación en el entorno.

Dependiendo del caso, el SSP que gestione la autenticación almacenará la contraseña del usuario y le permitirá no tener que autenticarse cada vez que necesite acceder al servicio, por le menos en un corto periodo de tiempo.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2019.png)

Dado esto, descargamos el comprimido `lsass.zip` en nuestro equipo local.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2020.png)

Al descomprimir dicho archivo, obtenemos un archivo de extensión `dmp`, lo cual corresponde a un fichero de volcado de memoria. 

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2021.png)

Para poder volcar el contenido de este archivo, en texto claro, usamos la herramienta `pypykatz`, utilizando el modo de volcado del LSA. 

```bash
pypykatz lsa minidump lsass.DMP
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2022.png)

Habiendo hecho esto, logramos obtener el hash NT de `svc_backup`, el cual nos permite autenticarnos en el servidor a través del servicio `WinRM`, debido a que dicho usuario forma parte del grupo `Remote Management Users`. 

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2023.png)

Al listar los grupos en los que está el usuario `svc_backup` vemos que pertenece a `Backup Operators`. 

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2024.png)

Por consecuencia también tiene habilitado el privilegio `SeBackupPrivilege`.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2025.png)

Un usuario perteneciente a dicho grupo puede crear una copia de seguridad del sistema de archivos, o restaurar ficheros, a pesar de las medidas de seguridad que se encuentren implementadas en el servidor. 

Considerando esto, podríamos pensar fácilmente en volcar la SAM, sin embargo, esto sólo aplica para un entorno local de Windows, en el que el equipo no está conectado a un directorio activo.

Para el caso AD, tenemos que usar `diskshadow` y crear un *snapshot* del sistema de archivos en otra partición del disco. En este caso los comandos se listan en un archivo de texto debido a que, con el parámetro `/s` estos serán ejecutados de igual forma, sin necesidad de acceder a la consola interactiva.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2026.png)

Como podemos ver en imagen, los comandos de `diskshadow` se ejecutan sin errores, por lo que la copia del sistema de archivos es realizada exitosamente.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2027.png)

Estando hecho el *backup*, utilizamos la herramienta `robocopy` para hacer una copia exacta del `NTDS.dit`, almacenado en la partición `E:`, hacía una ruta del servidor que se encuentre bajo nuestro control.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2028.png)

Teniendo el `NTDS.dit` y, habiendo hecho lo mismo con el `SYSTEM`, tenemos todo para volcar los hashes `NTLM` de todos los usuarios del dominio.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2029.png)

Como se muestra en la imagen, en este caso ejecutamos el siguiente comandos usando `impacket-secretsdump`, pasando como parámetros tanto el `NTDS.dit` como el `SYSTEM`, con lo que logramos volcar todos los hashes `NTLM` de los usuarios.

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
```

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2030.png)

Habiendo hecho esto, podemos usar el hash `NTLM` del usuario `Administrator` para autenticarnos en el servidor mediante WinRM y lograr el control total del *Domain Controller*.

![Untitled](/assets/img/htb/machines/Blackfield/Untitled%2031.png)

### Referencias

- Extract credentials from lsass remotely: [https://en.hackndo.com/remote-lsass-dump-passwords/](https://en.hackndo.com/remote-lsass-dump-passwords/)
- Privileged Groups (Hacktricks): [https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/privileged-groups-and-token-privileges](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/privileged-groups-and-token-privileges)
- Windows Authentication Architecture (Microsoft Learn): [https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-architecture](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-architecture)
- Credential Dumping: Local Security Authority (LSA|LSASS.EXE): [https://www.hackingarticles.in/credential-dumping-local-security-authority-lsalsass-exe/](https://www.hackingarticles.in/credential-dumping-local-security-authority-lsalsass-exe/)
- What is Backup Operators (WindowsTechno): [https://windowstechno.com/what-is-backup-operators/](https://windowstechno.com/what-is-backup-operators/)
- Copy / Xcopy / Robocopy: Which Is Better for Me? (EaseUS): [https://www.easeus.com/knowledge-center/copy-vs-xcopy-vs-robocopy.html](https://www.easeus.com/knowledge-center/copy-vs-xcopy-vs-robocopy.html)