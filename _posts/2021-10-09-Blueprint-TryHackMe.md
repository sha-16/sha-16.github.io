---
title: Blueprint - Write up (TryHackMe)
author: sha-16
date: 2021-10-09 13:58:00 -0300 
categories: [TryHackMe]
tags: [write-up, windows]
math: true
mermaid: true
image:
  src: /commons/thm/blueprint/banner/blueprint-banner.png
  width: 800
  height: 500
---

Este será el primer Write up de mi blog. Partiré suave con una máquina Windows de dificultad fácil en TryHackMe. 
Su nombre es [Blueprint](https://tryhackme.com/room/blueprint).

> Como ya sabrán algunos, para auditar máquinas en TryHackMe debemos conectarnos a una VPN mediante OpenVPN. Esto nos permitirá hacer como si nuestro equipo y el de TryHackMe estuvieran en la misma red.

* Instalación: [VPN de TryHackMe](https://tryhackme.com/room/openvpn)

Bueno sin más preambulos, comencemos con la máquina...


## **Enumeración de puertos**

Comenzaré haciendo un *Port discovery* o bien, en Español, un *Escaneo de puertos*. Para ello haré uso de nmap:

```bash
nmap -vvv -sS --open -n -Pn -p- --min-rate 5000 10.10.126.105 -oG ports
```

* **-vvv**: verbose con máximo despliegue de información.
* **-sS**: indicamos que queremos hacer conexiones por TCP SYN. Esto significa que no se entablarán completamente las conexiones por TCP, esto se hace no completando el [Three Way HandShake](https://www.techopedia.com/definition/10339/three-way-handshake)
* **\--open**: filtramos por los puertos que se detecten con estado *open*, o sea que están abiertos (no filtrados, ni cerrados).
* **-n**: deshabilitamos la [resolución DNS](https://www.ionos.es/digitalguide/servidores/know-how/que-es-el-servidor-dns-y-como-funciona/).
* **-Pn**: deshabilitamos el *host discovery*, o *descubrimiento de equipos* por medio del [protocolo ARP](https://es.wikipedia.org/wiki/Protocolo_de_resoluci%C3%B3n_de_direcciones).
* **-p-**: indicamos que haga escaneo de los 65535 puertos de una máquina.
* **--min-rate**: señalamos la cantidad mínima de paquetes a enviar por segundo (en este caso indicamos 5000).
* **-oG**: solicitamos que output de la captura se almacene en formato grepeable en el fichero indicado (en mi caso es el archivo *ports*). 

![Imágen de los puertos abiertos](/commons/thm/blueprint/imgs/port-discovery.png)

**Ya con los puertos abiertos en mano, procederé a hacer un escaneo más exhaustivo de de estos:**

```bash
nmap -p80,135,139,443,445,3306,8080,49152,49154,49158,49159 -sV -sC 10.10.126.105 -oN nmap
```

* **-p**: me permite indicarle los puertos a escanear.
* **-sV**: detección de servicios y la versión de estos. 
* **-sC**: me permite utilizar los scripts por defecto que tiene la herramienta nmap para enumerar los servicios que estén corriendo en los puertos.
* **-oN**: solicitamos que el output del escaneo se almacene en un fichero en un formato normal (en mi caso lo almaceno en el fichero *nmap*).  

![Imágen del escaneo](/commons/thm/blueprint/imgs/enum-scan.png)

## **Enumeración de servicio SMB**

Como se puede notar en la captura de nmap, tenemos el puerto 139 y 445 corriendo el servicio SMB. Por esto mismo recurriremos a 
la herramienta [smbclient]() para enumerar los recursos compartidos que tenga la máquina (aunque también hay otras como 
[crackmapexec]() o [smbmap]()). 
Comenzaré listando las comparticiones sin hacer uso de credenciales (*null-session*):

```bash
smbclient -L 10.10.126.105 -N
```

* **-L**: solicitamos el listado de recursos compartidos.
* **-N**: indicamos el *null-session*   

Como podemos ver tenemos dos recursos asequibles; aquellos que no tienen el símbolo de dolar ($).

![Imágen del escaneo](/commons/thm/blueprint/imgs/smb-enum.png)

Con smbclient probaremos el acceso a ambos recursos y veremos si hay algo interesante por ver:

##### **1) Users:**

Al parecer este es el directorio por defecto de usuarios en el sistema. Al recorrer sus directorios, no di con nada interesante:

```bash
smbclient //10.10.126.105/Users -N
```
![Imágen del escaneo](/commons/thm/blueprint/imgs/smb-users.png)

##### **2) Windows:** 

Enumerando de forma rápida este recurso, veo que no permite descargas, ni carga, de ficheros/directorios. Por lo que no nos sirve de mucho.

```bash
smbclient //10.10.126.105/Windows -N
```

![Imágen del escaneo](/commons/thm/blueprint/imgs/smb-windows.png)


## **Enumeración de servicios HTTP**

Como puedo ver en la captura del escaneo de nmap, hay dos puertos corriendo servicios HTTP. Haré una enumeración de ambos 
con la herrmienta [whatweb](https://github.com/urbanadventurer/WhatWeb). Desde el navegador, podríamos utilzar la extensión [Wappalyzer](https://chrome.google.com/webstore/detail/wappalyzer/gppongmhjkpfnbhagpmjfkannfbllamg?hl=es). 
En este caso, con una me basta (lo ideal es probar con más de una si no estás obteniendo resultados de interés):


##### **1) Servicio HTTP en el puerto 80:**

Como se observa hay un Microsoft IIS de versión 7.5 y la página principal en el navegador responde con un código de estado 404. 

```bash
whatweb http://10.10.126.105/
```

![Imágen del escaneo](/commons/thm/blueprint/imgs/whatweb.png)

**Código de estado 404 en el navegador:** 

![Imágen del resultado](/commons/thm/blueprint/imgs/browser-404-capture.png)

Buscando algún exploit, para el servicio **Microsft IIS 7.5**, no di con nada que fuera útil (lo ideal hubiera sido dar
con algo que me permitierá hacer algún RCE, File Upload, Path Traversal, etc). 

* **Fuentes de busqueda (**para exploits**)**: [Github](https://github.com), [exploit-db](https://exploit-db.com) y [Google](https://google.com)


##### **2) Servicio HTTP en el puerto 8080:**

Analizando la captura, lo suyo sería chequear que estos servicios, y tecnologías, no fueran vulnerables (sobre todo si consideramos 
el hecho de que están desactualizadas). 

```bash
whatweb http://10.10.126.105:8080/
```

![Imágen del escaneo](/commons/thm/blueprint/imgs/whatweb2.png)

**Para ahondar más, continue la enumeración mediante el mismo navegador:**

![Imágen del resultado](/commons/thm/blueprint/imgs/browser-oscommerce.png)

![Imágen del resultado](/commons/thm/blueprint/imgs/browser-oscommerce-2.png)

Como vemos, pareciera ser que el servicio web se integra por directorios y ficheros de una especie de CMS llamado **OSCommerce (de versión 2.3.4)**.

Analizando y enterandome de qué es esto, di de forma acertada con que es un tipo de Gestor de contenidos enfocado al comercio electrónico. 

![Imágen de la busqueda](/commons/thm/blueprint/imgs/enum-oscommerce.png)

Ahondando un poco más, encontré con un exploit para la versión 2.3.4 que nos permite hacer ejecución remota de comandos:

![Imágen del exploit](/commons/thm/blueprint/imgs/oscommerce-exploit.png)


## **Explotación y escalación de privilegios**

Haciendo ejecución del exploit hice el Remote Command Execution (RCE) y para variar gane acceso al sistema como **NT Authority\System**, o sea practicamente 
como el usuario Administrator con control total del equipo.  

![Imágen del exploit ejecutado](/commons/thm/blueprint/imgs/rce-exploit.png)

Pero aquí no termina todo. Por ahora tenemos una shell medianamente interactiva, que corresponde unicamente a un exploit de RCE. Lo ideal sería montarnos una *Reverse Shell*.
Para ello vamos a tratar de meter el binario de netcat al Windows. Por eso, primero debemos tenerlo en nuestra máquina local y abrir un servidor HTTP en el directorio que esté
se encuentra (en mi caso abro el servicio con Python3):

![Imágen del binario en nuestra máquina y de la apertura del servicio HTTP](/commons/thm/blueprint/imgs/share-netcat.png)

Por otro lado, en la máquina remota hago uso de la utilidad nativa de Windows, **Certutil.exe**, para descargar el binario de mi servicio HTTP.

>Es bueno considerar que Certutil.exe no siempre está instalada en todas las máquinas Windows (por más que sea una utilidad nativa). Por lo que es importante tener a mano otras formas de mover archivos de una máquina a otra (no siempre es necesario hacer uso de servicios HTTP).

Ya con el netcat descargado, pondré a mi equipo en escucha por el puerto 443:

```bash
rlwrap nc -nlvp 443
```

>rlwrap es una utilidad que permite añadir historico de comandos, movimiento del cursor, etc., que la shell sea aún más interactiva sin necesidad de complicarse tanto.

Ahora desde el Windows, ejecutaré el netcat entablar una conexión y ejecutarme una CMD luego de hacerlo:

![Imágen ya dentro de la máquina completamente](/commons/thm/blueprint/imgs/reverse-shell.png)

De esta forma, ya estoy completamente dentro.

Si deseamos hacer **lectura de la flag** (root.txt) usamos el comando ```type```:

```
type C:\Users\Administrator\Desktop\root.txt
```


## **Volcando Hashes NTML** 

El desafío de TryHackMe nos solicita que obtengamos la contraseña del usuario Lab en texto claro a partir de su hash NTLM desencriptado. 

Para volcar estos hashes debemos de tener a mano lo que es la SAM y el SYSTEM (bases de datos en formato de registro que almacenan la información requerida para obtener las credenciales de los usuarios).

>Ojo que la SAM y el SYSTEM sólo se pueden obtener con privilegios del usuario Administrator.

Para mover estos binarios yo fui por lo práctico y cree una copia tanto de la SAM como del SYSTEM en el directorios que se compartía mediante el servicio SMB (a los cuales teniamos acceso mediante el *null-session*).

Ambas copias las metí en el directorio: ```C:\Users\Public\Documents\```

```bash
C:\Users\Public\Domcuments>reg save HKLM\SAM .\SAM.bak
C:\Users\Public\Domcuments>reg save HKLM\SYSTEM .\SYSTEM.bak
``` 

![SAM.bak y SYSTEM.bak](/commons/thm/blueprint/imgs/sam-system-bak.png)

**Acá podemos ver a SAM.bak y SYSTEM.bak en los recursos compartidos:**
 
![SAM y SYSTEM en el SMB](/commons/thm/blueprint/imgs/sam-system-smb.png)

Luego en mi equipo hice una mountra de tipo *cifs* con el objetivo de mover más rapidamente la SAM y el SYSTEM a mi sistema:

```bash 
mkdir /mnt/remote_dir
mount -t cifs //10.10.126.105/Users /mnt/remote_dir
```
![Transfiriendo SAM/SYSTEM](/commons/thm/blueprint/imgs/cp-sam-system.png)


Ya con la SAM y el SYSTEM hago uso de la utilidad [samdump2]() para volcar las credenciales: 

![Volcado de hashes](/commons/thm/blueprint/imgs/dump-hashes.png)

Como ven debemos extraer la parte que nos interesa del Hash de Lab.

Para desencriptarlo podemos tirar de John, Hashcat, etc., pero yo en este caso fui por [Crackstation]() (que hace algo parecido pero desde la web). Este me entrego el valor del hash en texto claro en cosa de segundos: 

![Imágen de crackstation con la contraseña crackeada](/commons/thm/blueprint/imgs/hash-cracked.png)

Y pues bueno, el Write up está terminado! Espero haberte sido de ayuda, tengo un canal de YouTube por si quieres estar más atento a las cosas que estoy haciendo, pronto 
haré alguna que otra pasada por Twitch, etc.

¡Hasta la próxima!  
