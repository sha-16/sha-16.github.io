---
title: Sauna - Write Up (HTB) ❤
author: sha16
categories: [HackTheBox, Active Directory]
tags: [AS-REP Roasting, Abusing DCSync, ActiveDirectory, Pass-the-hash (PtH), Windows]
---

Buenas a tod@s, el día de hoy les traigo un `write-up` de la máquina `Sauna` de `HackTheBox`.  

![Untitled](/assets/img/htb/machines/Sauna/Untitled.png)

Como en cada `write-up`, lo primero hacemos es lanzar un escaneo de puertos con `nmap` con el objetivo de identificar potenciales vías de ataque. 

Como podrán ver a continuación, se específica el parámetro `-sS` con el objetivo de que el escaneo sea vía `TCP SYNC`, lo que de cierto modo me permitirá agilizarlo, debido a que no hay `Three-Way-HandShake` como en `TCP`.

Lo otro es que se configura el `--min-rate` en `3000` con la idea de no enviar una cantidad menor a `3000` paquetes por segundo.

**Advertencia:** No aconsejo lanzar este escaneo en entornos productivos debido a que es ultra ruidoso y agresivo. En este caso lo estoy lanzando en un entorno controlado de `HackTheBox`.

```bash
nmap -v -sS --min-rate 3000 -n -Pn -p- 10.10.10.175 -oG tcp_ports.txt
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%201.png)

En el resultado del escaneo, se identifican varios servicios de interés, entre ellos: WinRPC, SMB, LDAP, WinRM y Kerberos. Sin embargo, ya con sólo ver al servicio `Kerberos`, podemos deducir que estamos ante un entorno de *Active Directory*. Por lo que ya podemos saber más o menos cómo enforcar nuestra estrategia de ataque y enumeración.

```bash
nmap -v -sCV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49677,49689,49696 10.10.10.175 -v port_scan.txt
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%202.png)

Una vez entendemos a rasgos generales, ante qué nos enfrentamos, revisamos la aplicación web que se encuentra corriendo en el puerto `80/http`.

En dicho servicio existe un *landing page*, en el endpoint `about.html` donde se listan los nombres y apellidos de los miembros del equipo. 

![Untitled](/assets/img/htb/machines/Sauna/Untitled%203.png)

En base a esto, creamos el siguiente diccionario con potenciales nombres de usuarios que se encuentran registrados en el dominio.

![Untitled](/assets/img/htb/machines/Sauna/Untitled%204.png)

Hecho esto, con la herramienta `kerbrute` validamos si es que alguno de estos usuarios se encuentra registrado en el dominio. 

```bash
kerbrute userenum --dc 10.10.10.175 -d egotistical-bank.local users.txt
```

Como se muestra en la imagen, la herramienta `kerbrute` permite identificar al usuario `FSmith` registrado en el dominio.

![Untitled](/assets/img/htb/machines/Sauna/Untitled%205.png)

Teniendo dicho nombre de usuario, ejecutamos un ataque `AS-REPRoasting` con la herramienta `impacket-GetNPUsers`.

En la imagen a continuación, se puede cómo es posible solicitar el `Ticket Granting Ticket (TGT)` del usuario, enviando una solicitud `KRB_AS_REQ` al `Key Distribution Center (KDC)` de `Kerberos`. Este servicio responde correctamente al atacante con un mensaje `KRB_AS_REP`, cual contiene el `TGT`.

Esto ocurre debido a que la cuenta del usuario tiene configurado el privilegio `UF_DONT_REQUIRE_PREAUTH`, lo cual permite solicitar el `TGT` sin previa autenticación.

```bash
impacket-GetNPUsers -no-pass -usersfile user.txt egotistical-bank.local/
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%206.png)

Como mencionamos en el post anterior, es posible crackear un `Ticket Granting Ticket` debido a que parte de este se encuentra cifrado con la contraseña del usuario. 

Probando un ataque de fuerza bruta con la herramienta `john`, logramos obtener la contraseña del usuario `FSmith`.

![Untitled](/assets/img/htb/machines/Sauna/Untitled%207.png)

Como se muestra a continuación, dicha credencial es validada con `crackmapexec` sobre el servicio `WinRM`, con lo cual comprobamos que el usuario pertenece al grupo `Remote Management Users`. Esto significa que dicho usuario se puede conectar a la terminal del servidor y ejecutar comandos. 

Para aprovecharnos de dicho grupo utilizamos la herramienta `evil-winrm`, la cual nos permite emular el comportamiento de una terminal `PowerShell` y acceder al servidor de manera directa.

![Untitled](/assets/img/htb/machines/Sauna/Untitled%208.png)

Enumerando a los usuarios del dominio, identificamos a `svc_loanmgr`.

![Untitled](/assets/img/htb/machines/Sauna/Untitled%209.png)

Dado esto, revisando las credenciales almacenadas en el `Winlogon` se identifica una contraseña por defecto.

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2010.png)

Esta contraseña es probada sobre el servicio SMB con el usuario `svc_loanmgr` la cual resulta ser valida. 

```bash
crackmapexec smb 'egotistical-bank.local' -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2011.png)

Posterior a esto, lanzamos la herramienta `bloodhound-python` con el objetivo de enumerar gran parte de los privilegios y configuraciones habilitadas entre los objetos del *Active Directory*.

```bash
bloodhound-python -u 'fsmith' -p 'Thestrokes23' -d egotistical-bank.local -ns 10.10.10.175 -c all
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2012.png)

Recopilando información sobre los privilegios de cada cuenta, identificamos que el usuario `svc_loanmgr` tiene capacidad `DCSync` sobre el dominio, lo cual significa que este puede solicitar los hashes `NTLM` de los usuarios. 

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2013.png)

Sabiendo esto, con la herramienta `impacket-secretsdump` solicitamos los hashes `NTLM` de los usuarios del dominio, a partir de las credenciales de `svc_loanmgr`. 

```bash
impacket-secretsdump 'egotistical-bank.local/svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2014.png)

Posterior a esto, tenemos dos alternativas: 

1. Ejecutar un *Pass-the-hash attack* y engañar al sistema de autenticación con la herramienta `impacket-psexec`, o `wmiexec`. 
2. Abusar de `WinRM` con la herramienta `evil-winrm` y conectarnos directamente a una terminal del servidor. Ambas opciones son válidas para obtener una terminal con el usuario `Administrator`. 

En este caso, el primero que probamos es el *Pass-the-hash*, con la herramienta `impacket-psexec`, el cual funciona exitosamente como se muestra en imagen.

```bash
impacket-psexec -hashes ':823452073d75b9d1cf70ebdf86c7f98e' 'egotistical-bank.local/Administrator@10.10.10.175'
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2015.png)

La segunda opción de igual forma funciona y permite obtener control total del servidor.

```bash
evil-winrm -i egotistical-bank.local -u 'Administrator' -H '823452073d75b9d1cf70ebdf86c7f98e'
```

![Untitled](/assets/img/htb/machines/Sauna/Untitled%2016.png)