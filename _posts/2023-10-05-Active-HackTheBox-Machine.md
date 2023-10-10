---
title: Active - Write Up (HTB) ❤
author: sha16
categories: [HackTheBox, Active Directory]
tags: [Kerberoasting, Decrypting cpassword (Groups.xml), Windows]
---

Buenas a tod@s, el día de hoy les traigo un `write-up` de la máquina `Active` de `HackTheBox`.  

![Untitled](/assets/img/htb/machines/Active/Untitled.png)

Como en cada `write-up`, lo primero hacemos es lanzar un escaneo de puertos con `nmap` con el objetivo de identificar potenciales vías de ataque. 

Como podrán ver a continuación, se específica el parámetro `-sS` con el objetivo de que el escaneo sea vía `TCP SYNC`, lo que de cierto modo me permitirá agilizarlo, debido a que no hay `Three-Way-HandShake` como en `TCP`.

Lo otro es que se configura el `--min-rate` en `3000` con la idea de no enviar una cantidad menor a `3000` paquetes por segundo.

**Advertencia:** No aconsejo lanzar este escaneo en entornos productivos debido a que es ultra ruidoso y agresivo. En este caso lo estoy lanzando en un entorno controlado de `HackTheBox`.

```bash
nmap -v -sS --min-rate 3000 -n -Pn -p- 10.10.10.100 -oG tcp_ports.txt
```

![Untitled](/assets/img/htb/machines/Active/Untitled%201.png)

En el resultado del escaneo, se identifican varios servicios de interés, entre ellos: WinRPC, SMB, LDAP, WinRM y Kerberos. Sin embargo, ya con sólo ver al servicio `Kerberos`, podemos deducir que estamos ante un entorno de *Active Directory*. Por lo que ya podemos saber más o menos cómo enforcar nuestra estrategia de ataque y enumeración.

```bash
nmap -v -sCV -p53,135,139,389,445,464,593,636,3268,3269,5722,49152,49153,49158,49169,49171 10.10.10.100 -oN scan_ports.t
```

![Untitled](/assets/img/htb/machines/Active/Untitled%202.png)

Al enumerar el servicio SMB, detectamos que este posee habilitado el *null session*, lo que nos da la posibilidad de acceder y recopilar información importante que se almacene en los recursos compartidos.

![Untitled](/assets/img/htb/machines/Active/Untitled%203.png)

En este caso, al primer recurso compartido que accedemos es a `Replication` donde identificamos que contiene un directorio con el nombre del dominio, de manera similar a lo que comúnmente se encuentra en `SYSVOL`.

Esto es importante, ya que si esta carpeta compartida es lo mismo que el `SYSVOL`, significa que existe una alta posibilidad de que podamos acceder a la directiva de grupo del directorio activo.

El `SYSVOL` es un recurso compartido en *Active Directory* sobre el cual, por defecto, todo los usuarios tienen acceso de lectura. Este recurso se encuentra sincronizado con el *Domain Controller* y almacena distintas cosas entre estas: *logon scripts*, información de políticas de grupo y otros datos que deben estar disponibles en todo momento para el *Domain Controller*.

La política de grupo (también conocida como *Group Policy*) es una tecnología en Windows Server que permite administrar la configuración de usuarios y computadoras en un entorno de directorio activo.  

![Untitled](/assets/img/htb/machines/Active/Untitled%204.png)

Como se muestra en la siguiente imagen, validamos correctamente que esta carpeta nos permite acceder al contenido del *Group Policy*, en el cual identificamos una `cpassword` configurada para el usuario `SVC_TGS`, mediante el archivo `Groups.xml`. 

![Untitled](/assets/img/htb/machines/Active/Untitled%205.png)

Esto como tal, representa un problema de seguridad, debido a que la clave de cifrado del `cpassword` fue revelada por Microsoft hace unos años atrás, lo cual le permitió que cualquier tipo de usuario con acceso al `SYSVOL`, leyera dicha clave y descifrara el valor real de la contraseña.

Para obtener la contraseña utilizamos la herramienta `gpp-decrypt` la cual hace uso de esta clave revelada por Microsoft para descifrar el valor de las `cpasswords`.

![Untitled](/assets/img/htb/machines/Active/Untitled%206.png)

Como se puede ver en imagen, la clave es descifrada correctamente.

![Untitled](/assets/img/htb/machines/Active/Untitled%207.png)

En base a esto, teniendo credenciales válidas en el directorio activo, lanzamos un *Kerberoasting Attack* con la herramienta `impacket-GetUserSPNs` la cual nos permite solicitar un `Ticket Granting Servicd (TGS)`.

Para dar contexto, el *Kerberoasting* es un ataque que consiste en solicitar un `Ticket Granting Service (TGS)` de un usuario en un entorno de *Active Directory* que cuenta con autorización en uno o varios servicios (que comúnmente se les referencia por el atributo `Service Principal Name (SPN)`). Es importante mencionar que quien solicita dicho ticket, debe de ser un usuario autenticado en el directorio activo.

En este caso, con `GetUserSPNs` listamos y solicitamos los `Ticket Granting Service` de los usuarios *kerberoastables*, donde logramos ver que el usuario `Administrator` es uno de ellos. Por lo que el ticket obtenido es el del usuario con máximos privilegios en el directorio activo. 

```bash
impacket-GetUserSPNs 'active.htb/SVC_TGS:GPPstillStandingStrong2k18' -request
```

![Untitled](/assets/img/htb/machines/Active/Untitled%208.png)

Una vez obtenido el `TGS` del usuario `Administrator`, lo almacenamos en un archivo `svc_tgs-tgs.crack` y ejecutamos un ataque de fuerza bruta local utilizando la herramienta `john`, con el objetivo de crackearlo.

Como se puede ver en imagen, dicho objetivo es logrado por lo que obtenemos la contraseña del usuario `Administrator`. 

![Untitled](/assets/img/htb/machines/Active/Untitled%209.png)

Ya con la contraseña del usuario `Administrator` verificamos que está es válida sobre el servicio SMB, por lo cual podemos autenticarlo directamente en el servidor.

![Untitled](/assets/img/htb/machines/Active/Untitled%2010.png)

En este caso, a través del siguiente comando con la herramienta `impacket-psexec` logramos autenticar al usuario con éxito en el servidor, obteniendo una terminal de máximos privilegios como `NT Authority\System`.

```bash
impacket-psexec 'active.htb/Administrator:Ticketmaster1968@10.10.10.100'
```

![Untitled](/assets/img/htb/machines/Active/Untitled%2011.png)

### Referencias

- Exploiting GPP SYSVOL (Groups.xml): [https://vk9-sec.com/exploiting-gpp-sysvol-groups-xml/](https://vk9-sec.com/exploiting-gpp-sysvol-groups-xml/)
- Kerberoasting Attacks: [https://www.crowdstrike.com/cybersecurity-101/kerberoasting/](https://www.crowdstrike.com/cybersecurity-101/kerberoasting/)
- Kerberoasting: [https://en.hackndo.com/kerberoasting/](https://en.hackndo.com/kerberoasting/)