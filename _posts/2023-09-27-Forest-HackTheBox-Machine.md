---
title: Forest - Write Up (HTB) ❤
author: sha16
categories: [HackTheBox, Active Directory]
tags: [AS-REP Roasting, Abusing Account Operators, Abusing DCSync, Abusing Exchange Windows Permissions, Abusing WriteAcl Permissions, Active Directory, User Enum via RPC, HTB]
---

Buenas a tod@s, el día de hoy les traigo un `write-up` de la máquina `Forest` de `HackTheBox`.  

![Untitled](/assets/img/htb/machines/Forest/Untitled.png)

Como en cada `write-up`, lo primero hago es lanzar un escaneo de puertos con `nmap` con el objetivo de identificar potenciales vías de ataque. 

Como podrán ver a continuación, específico el parámetro `-sS` con el objetivo de que el escaneo sea vía `TCP SYNC`, lo que de cierto modo me permitirá agilizarlo, debido a que no hay `Three-Way-HandShake` como en `TCP`.

Lo otro es que configuré el `--min-rate` en `3000` con la idea de no enviar una cantidad menor a `3000` paquetes por segundo.

*Advertencia:* No aconsejo lanzar este escaneo en entornos productivos debido a que es ultra ruidoso y agresivo. En este caso lo estoy lanzando en un entorno controlado de `HackTheBox`.

```bash
nmap -v -sS --min-rate 3000 -n -Pn -p- 10.10.10.161 -oG tcp_ports.txt
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%201.png)

Posterior a ello, agarramos todos los puertos identificados, y lanzo un escaneo de servicios, con el objetivo de detectar nombres y versiones que puedan tener vulnerabilidades conocidas publicamente.

```bash
map -v -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49706,49969 10.10.10.161 -oN port_scan.txt
```

En el resultado del escaneo, identifico varios servicios de interés, entre ellos: WinRPC, SMB, LDAP, WinRM y Kerberos. Sin embargo, ya con sólo ver al servicio `Kerberos`, tengo claro de que estoy ante un entorno de *Active Directory*, por lo que ya sé más o menos cómo enforcar mi estrategia de ataque.

![Untitled](/assets/img/htb/machines/Forest/Untitled%202.png)

Al enumerar el servicio WinRPC, detectó que este posee habilitado el *null session*, lo que me da la posibilidad de acceder y recopilar información importante del dominio.

![Untitled](/assets/img/htb/machines/Forest/Untitled%203.png)

En este caso, ejecutando el siguiente `one-liner` extraigo a todos los usuarios listados en `WinRPC`, con la herramienta `rpcclient`.  

```bash
rpcclient -N -U '' 10.10.10.161 -c 'enumdomusers' | grep -oP '\[.*?\]' | grep -v '0x' | tr -d ']['
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%204.png)

Posterior a esto, con la herramienta `kerbrute` valido si estos usuarios son válidos y se encuentran correctamente registrados en el dominio.

```bash
kerbrute userenum --dc 10.10.10.161 -d htb.local users.txt
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%205.png)

Sabiendo esto, con el listado de usuarios en mano, ejecuto un ataque `AS-REPRoasting`, donde se identificó que el usuario `svc-alfresco` tiene habilitado el privilegio `UF_DONT_REQUIRE_PREAUTH`.

Este privilegio permite efectuar una solicitud del tipo `KRB_AS_REQ` al servicio `Kerberos`, sin previa autenticación del usuario, la cual respondida con `KRB_AS_REP`. Esta respuesta como tal, entrega el `Ticket Granting Ticket (TGT)` del usuario, el cual posteriormente puede ser crackeado. De ahí surge el nombre del ataque.

En este caso, logramos identificar que el ataque se efectúa exitosamente sobre el usuario `svc-alfresco`.

```bash
impacket-GetNPUsers -no-pass -usersfile users.txt htb.local/
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%206.png)

Un atacante puede obtener la contraseña de un usuario crackeando un `Ticket Granting Ticket (TGT)`, debido a que parte de este se encuentra cifrado con la misma contraseña.

En base a esto, crackeo el `TGT` obtenido, lo cual me permite tener credenciales del entorno en mano y hacer pruebas con ello.

![Untitled](/assets/img/htb/machines/Forest/Untitled%207.png)

Una vez obtenidas las credenciales, utilizando `crackmapexec`, valido que el usuario `svc-alfresco` es parte del grupo `Remote Management Users` lo cual me permite conectarme y ejecutar comandos de manera directa en el servidor, a través del servicio `WinRM`.

```bash
crackmapexec winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%208.png)

Como se muestra en imagen, con la herramienta `evil-winrm` logro acceder correctamente a la terminal del usuario `svc-alfesco`, mediante `WinRM`.

```bash
evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%209.png)

Posterior a esto, ejecuto la herramienta `bloodhound-python` con la cual enumero gran parte de la configuración existente en el directorio activo, a partir de las credenciales obtenidas:

```bash
bloodhound-python -u 'svc-alfresco' -p 's3rvice' -d htb.local --auth-method ntlm -ns 10.10.10.161 -c all
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%2010.png)

Como se observa en la imagen, el usuario `svc-alfresco` pertenece al grupo `Service Accounts`, el cual es miembro del grupo `Privileged IT Accounts`, el que a su vez es miembro del grupo `Account Operators`:

El grupo `Account Operators` tiene dos cualidades particulares de las cuales me puedo aprovechar como atacante:

- Puede crear y administrar cuentas de usuarios y grupos *no administradores* en el dominio.
- Pueden iniciar sesión en el `Domain Controller (DC)` localmente.

![Untitled](/assets/img/htb/machines/Forest/Untitled%2011.png)

Dado que el usuario `svc-alfresco` forma parte de `Account Operators`, significa que tengo la capacidad de gestionar cuentas y grupos en el dominio. 

Para ello, previamente, listo todos los grupos que puedan ser de utilidad para posteriormente escalar privilegios.

Entre estos identificó a `Exchange Windows Permissions`, el cual corresponde a un grupo de *Active Directory* que, de manera predeterminada posee habilitado el permiso `WriteDACL`, lo cual me permitiría asignar ciertos permisos al resto de usuarios en el dominio, entre ellos `DCSync`.

- *“`DCSync` es una técnica que se utiliza para pedir las claves de cualquier usuario a un controlador del dominio, a través del protocolo de replicación `DRSUAPI`.” - Tarlogic*

Por lo general, en entornos de AD donde se encuentra se instalado `Microsoft Exchange`, suelen haber una cantidad mayor de usuarios en el grupo `Exchange Windows Permissions`. 

![Untitled](/assets/img/htb/machines/Forest/Untitled%2012.png)

Sabiendo esto, creo la cuenta de usuario `sha16` en el dominio y la añado al grupo local `Remote Management Users`, con el objetivo de que este puede acceder al servidor mediante el servicio `WinRM`. 

Por otro lado, también añado al usuario al grupo del dominio, `Exchange Windows Permissions`, el cual tiene la capacidad `WriteDacl`, lo cual me permitiría escalar privilegios, mediante la asignación de permisos críticos.

Para dar contexto, las `DACL` son las *Dictionary Access Control List* con las cuales es posible gestionar de manera efectiva los accesos en los entornos de directorio activo.  

![Untitled](/assets/img/htb/machines/Forest/Untitled%2013.png)

Como se muestra en imagen, dicho grupo posee el permisos `WriteDacl` sobre el dominio.

![Untitled](/assets/img/htb/machines/Forest/Untitled%2014.png)

Dado esto, con la siguiente instrucción en PowerShell, habiendo importado PowerView previamente, asigno el permiso `DCSync` a la cuenta del usuario `sha16`.

```powershell
# Add-DomainObjectAcl = Add-ObjectAcl
Add-ObjectAcl -PrincipalIdentity sha16 -Domain htb.local -Rights DCSync
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%2015.png)

Hecho esto, como se muestra en imagen, el usuario `sha16` tiene la capacidad de volcar todo el NTDS, pudiendo obtener todos los hashes `NTLM` de los usuarios del dominio.

```bash
impacket-secretsdump 'htb.local/sha16:Admin12345$@10.10.10.161'
```

![Untitled](/assets/img/htb/machines/Forest/Untitled%2016.png)

Ya con el hash `NTLM` del usuario `Administrador`, me puedo conectar a su cuenta a través del servicio `WinRM` y obtener una terminal interactiva directamente en el servidor.

![Untitled](/assets/img/htb/machines/Forest/Untitled%2017.png)