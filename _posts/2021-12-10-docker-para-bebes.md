---
title: Docker para bebes 👻
author: sha-16
date: 2021-12-10 00:20:00 -03000 
categories: [Docker]
tags: [docker, guide]
math: true
mermaid: true
---

## ¿Qué es? 🎈

Es un software que asegura que una aplicación (con sus respectivas dependencias) corra en un entorno dedicado para sí (corre de forma contanerizada).



## ¿Para qué nos sirve? 🧨

Permite que una aplicación, junto con sus dependencias, esté contenida en una especie de SandBox. Un entorno dedicado para la App. 



## ¿Por qué Docker? ⚔

Evita que hayan conflictos entre dependencias y versiones de un App con otra que se encuentran en un mismo servidor. Por otro lado Docker hace mucho más facil la tarea de mover una aplicación de una máquina a otra sin problemas compatibilidad.

Con esto se acabó lo de: 
> En mi máquina funciona...

Por otro lado, si un atacante consiguiera ejecución remota de comandos a través de una App, sólo lograría acceder al contenedor y no al servidor como tal. Es un plus de seguridad que no viene nada mal.



## ¿Cómo es la estructura de Docker? ⛓

* **Infraestructura**: El servidor que posee al OS. 
* **Sistema Operativo**: Está corriendo a Docker.
* **Docker**: Programa encargado de mantener a los contenedores.
* **Contenedores**: Entornos dedicados para aplicaciones.

> Los contenedores de Docker lo único que comparten es el mismo Kernel del Sistema Operativo. Por eso es que un contendor puede estar corriendo con un Ubuntu, un Debian, CentOS, etc (comparten el Kernel de Linux).  

Docker utiliza una imágen para correr un contendor. Esta imagen se compone de: 

* **Sistema Operativo**
* **Aplicación**
* **Software (dependencias, software de servicios, librerías, etc)**

> Para crear una imagen de Docker debemos hacer y preescribir las instrucciones de esto en un Dockerfile. 



## Dockerfile 📄

```bash
From debian
RUN apt-get install apache2 
CMD [ "apache2" ]
``` 



## Nota importante 📌

Es importante que antes de crear nuestra propia imagen, miremos en los repositorios oficiales de Docker Hub si alguien más ya creo¿ó una imagen con las dependencias que necesitamos para correr nuestra App. 
Por un lado esto nos ahorraría trabajo, y por otro podemos evitar acarrear malas practicas a la hora de crear el Dockerfile para la imagen. 

> Es una buena prática utilizar imagenes oficiales que hayan en Docker Hub y que sean de última versión.

**Tags**: Etiquetas que se le asignan a cada imagen con distintas versiones de tecnologías. 

**Imágenes de contendores**: [Docker Hub Container Image Library](https://hub.docker.com/)



## Montando nuestro primer contenedor 😎

* Lo primero es crear un directorio donde vamos a montar nuestra App:

```bash
mkdir main
```

* Lo segundo es irnos al sitio web de Docker Hub y seleccionar la Tag de una imagen que se adapte a lo que nosotros necesitamos para montar nuestra propia App en un contendor con sus respectivas dependencias, versiones, librerías, etc. 

* Lo tercero es escribir el Dockerfile para hacer arrancar y construir nuestro contenedor, en este caso usé la Tag de Python:
> 3.11.0a2-bullseye

```bash
FROM python:3.11.0a2-bullseye
COPY main/ var/www/html
EXPOSE 80
``` 
* **FROM**: Nos permite establecer la imagen que vamos a utilizar para nuestro contenedor.
* **COPY**: La declaración señala que va a hacer una copia de nuestro directorio ```main``` completo en el directorio ```/var/www/html``` del contenedor.
* **EXPOSE**: Señala que el contenedor, cuando este corriendo, va tener abierto el puerto 80 abierto. 

#### Instalación de la Imagen en nuestra máquina

```bash 
docker build -t <container-name> .
```
**-t**: Nos permite asignarle un nombre al contenedor. La **t** viene de Tag.

* Arranque del contendorr con nuestra imagen ya instalada:
```bash
docker run -p 80:80 <container-name>
```

* **-p**: Nos permite redireccionar el puerto expuesto del contenedor a nuestro puerto local del servidor. Dicho esto, si nos vamos al navegador y buscamos http://localhost/ veremos que allí esta el servidor de la aplicación del contenedor corriendo. 

* Si queremos empezar a hacer cambios, crear directorios, hacer configuraciones, construir una App, etc., debemos montar un volumen de nuestro servidor local dentro del contenedor. El volumen es basicamente un directorio de nuestro servidor. 

```bash
docker run -p 80:80 -v /root/docker/main:/var/www/html <name-container>
```

* **-v**: Nos permite montar un directorio de nuestro servidor en el contenedor. De esta forma cualquier cambio que hagamos en el volumen de nuestra App se hará en los del contenedor:

 


**Instalación de Docker**: [Install Docker Engine](https://docs.docker.com/engine/install/) 
