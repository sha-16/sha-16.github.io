---
title: Docker - Cómo ser un pro siendo noob 💣
author: sha-16
date: 2021-12-13 17:30:00 -03000 
categories: [Docker]
tags: [docker, guide]
image: []
math: true
mermaid: true
image:
  src: https://babel.es/WebBabel/media/BlogImages/Docker_BABEL_2.png 
  width: 800
  height: 500
---

Docker nos sirve principamente para encapsular aplicaciones con sus respectivas dependencias. Lo cual significa que, estas pueden correr en cualquier máquina.

Esto a su vez permite que una app se puede exportar de PC a PC sin problemas de compatibilidad.

Con Docker basicamente encapsulamos a una aplicación en un entorno dedicado para sí, hecho completamente a su medida para correr.

Como Docker nos permite aislar entornos, significa que podemos tener multiples aplicaciones en un mismo servidor con distintas dependencias y diferentes versiones de estas.

Antiguamente se utilizaban máquinas virtuales para aislar entornos, lo que hoy en día es algo nefasto e ineficiente (**¿Por qué?** básicamente porque es un execesivo e innecesario consumo recursos). 


## Máquinas Virtuales VS Contenedores 💕


##### Máquinas Virtuales
*  Es un sistema operativo dentro de otro que posee un hardware basado en el del sistema operativo nativo. 
* Su peso es mucho mayor que el de un contenedor. Ocupan una gran porción del disco duro. 
* Su configuración e instalación es mucho más lenta.
* Es relativamente más seguro, pero mucho menos eficiente.


##### Contenedores
* Son virtualizaciones o instancias de un sistema operativo, que no contemplan al hardware en ningún aspecto. 
* Son mucho más ligeros y no ocupan mucho espacio en el disco. 
* Su instalación es practicamente instantánea. 
* Son mucho más eficientes a diferencia de las máquinas virtuales. 

## Docker Image 💿

* Es una plantilla que contendrá las instrucciones para armar al contenedor. 
* De cierta forma, es similar a un instalador.
* Puede contener todos, o casi todos, los elementos necesarios para montar una app contenerizada (esto dependerá de las necesidades del software).
* Puede ser compartida para que otros desarrolladores las ocupen. 
* Estas imagenes son creadas usando Dockerfile (para entenderlo de una: son las instrucciones para armar cada contenedor). 

## Contenedor 📦
 * Un contenedor es una instancia de una imagen ya instalada y arrancada. 
 * Poseen un rendimiento similar al del sistema operativo nativo (por lo que no perjudican a la aplicación como tal).
 * Pueden haber multiples de estos en un solo servidor. 
 * Cada contenedor es un proceso. 

## Docker Hub 🌎
Es un repositorio centralizado de Docker donde se comparten diferentes imágenes que nos pueden ser utilles de acuerdo a los proyectos que estemos llevando a cabo. 

**Sitio de Docker Hub**: [Docker Hub](https://hub.docker.com/)

## Glosario necesario 📌

* **ID**: cada imagen y contenedor tienen uno.
* **Tag**: es el nombre de una imagen. Esta puede contener dos puntos (:) para señalar la verisón de la misma, por ejemplo ```ubuntu```, ```ubuntu:latest``` o ```ubuntu:18.04```.

## Comandos básicos 👀⚡

**Instalación y arranque de un contenedor**: 
```bash
$ docker run <image/tag>
```

**Listar las imagenes instaladas en el servidor**:
```bash
$ docker images
```

**Buscar una imagen de Docker Hub por terminal**:
```bash
$ docker search <image/tag>
```

**Descargar una imagen**: 
```bash
$ docker pull <image/tag>
```

**Ejecutar un comando en el contenedor**:
```bash 
$ docker run <tag> <command>
```
**Ejecutar un inteprete bash, o sh, de forma interactiva en el contenedor**: 
```bash 
$ docker run -it <tag> bash 
$ docker run -it <tag> sh 
```

**Listar los contenedores que estén corriendo en el servidor**: 
```bash
$ docker ps
$ docker ps -a (muestra el historial de los contendores que estuvieron corriendo)
```

**Eliminar un contendor**:
```bash
$ docker rm <container-id/name>
```

**Arrancar un contenedor nuevamente**: 
```bash
$ docker start <container-id> (este puede estar detenido pero ser visualizado con docker ps -a)
```

**Detener un contenedor corriendo**:
```bash
$ docker stop <container-id>
```

**Visualizar un puerto interno del contenedor en uno de nuestro servidor local (tunelizamos su salida)**:
```bash
$ docker run -p <local-port>:<container-port> <tag>
```

**Visualizar un puerto interno del contenedor en varios puertos de nuestro servidor local (tunelizamos las salida)**:
```bash
$ docker run -p <local-port>:<container-port> -p <local-port>:<container-port> -p <local-port>:<container-port> <tag>
```

**Arrancar un contenedor en segundo plano (Detached Mode / Modo  Independiente)**:
```bash
$ docker run -p <local-port>:<container-port> -d <tag>
```

**Listar los IDs de todos los contendores**:
 ```bash
$ docker ps -q
$ docker ps -aq (considera a los contenedores que hace poco fueron detenidos)
```

**Asignar un nombre a contenedor al arrancarlo**:
```bash
$ docker run --name <container-name> <tag>
```

**Forzar opciones de Docker**: 
```bash
$ docker rm -f <container-id> (forzamos la eliminación de un contenedor)
$ docker rmi -f <image> (forzamos la eliminación de una imagen)
```

**Establecer variables de entorno en el contenedor**:
```bash
$ docker run -e <env-variable> <tag>
```

**Establecer un formato de salida al listado de contenedores**:
```bash
$ docker ps -a --format="format"
$ docker ps --format="format"
$ docker ps -a --format="\n[*] ID: {{.ID}}\n[*] Name: {{.Name}}"
```


**Montar un volumen del servidor local en el contenedor**: 
```bash
$ docker run -v <local-dir>:<container-dir> <tag>
```
* **Nota**: un volumen es basicamente el contenido de un directorio de nuestro servidor. Este contenido se copia en dentro del volumen del contenedor. Esto significa que cualquier cambio que se haga en los volumenes va a ocurrir tanto en el servidor como en el contenedor, lo que genera persistencia de los datos.

**Exponer un puerto del contenedor en una red de contenedores**:
```bash 
$ docker run --expose <port> <tag>
```
* **Nota**: algunas imágenes ya traen habilitada esta opción por defecto aunque, como buena práctica, nunca viene mal señalarlo. Sobre todo si más adelante vamos a llevar nuestro proyecto a un entorno de producción.

**Ejecutar comandos en un contenedor arrancado**:
```bash
$ docker exec <container-id> <command>
```

**Abrir una terminal interactiva en un contenedor arrancado**:
```bash
$ docker exec -it <container-id> bash (nos abre una terminal interactiva con bash en el contenedor arrancado)
$ docker exec -it <container-id> sh (nos abre una terminal interactiva con sh en el contenedor arrancado)
```

## Dockerfile 📄
 Como mencioné anteriormente los Dockerfile son las recetas para montar nuestras propias imagenes.
 
 ##### Contenido común de un Dockerfile: 
 * **FROM**: La imagen o tag que vamos a tomar como base para la nuestra.
 * **RUN / CMD / ENTRYPOINT**: Los comandos a ejecutar cuando se arranque en el contenedor (puede ser al construirlo o arrancarlo; dependerá meramente de la sentencia que usemos). 
 * **WORKDIR**: Nos permitirá establecer el directorio de trabajo del usuario root en el contenedor.
 * **COPY**: Nos permitirá copiar cosas de nuestro servidor al contenedor. 

### Dockerfile de ejemplo
```bash
FROM nginx:1.21.24

WORKDIR /var/www/html
COPY . .

RUN npm install -y serialize 
CMD ["node", "index.js"]
```

* En este caso vemos que se ocupará como base la imagen ```nginx``` de versión ```1.21.24```.
* Por otro lado, vamos a establecer al directorio ```/var/www/html``` como directorio de trabajo dentro del contenedor.
* También vemos que se copiará todo el contenido de nuestro directorio actual (```.```), en el directorio de trabajo del contenedor (eso es lo que significa cada punto).
* Con ```RUN``` vamos a correr un comando únicamente cuando construyamos la imagen con ```docker build```. 
* Con ```CMD``` vamos a ejecutar un comando cada vez que arranquemos un contenedor con ```docker run```.
* Aunque no está definido en el Dockerfile, ```ENTRYPOINT``` nos va a permitir ejecutar un comando siempre que arranquemos el contenedor; pero dicho comando estará a la espera de un argumento que puede ser un tanto un fichero como un parámetro, entre otras. Esa es su principal diferencia con ```CMD```.

## Construyendo la imagen 🛠
En el directorio donde tengamos el Dockerfile debemos ejecutar lo siguiente para construir nuestra imagen: 
```bash
$ docker build -t <tag> . 
```
* **-t**: Nos da la opción de asignarle un nombre, o etiqueta (tag), a nuestra imagen.  

## Subir una imagen a Docker Hub ✔

Una vez que tenemos nuestro Dockerfile podemos proceder a crearnos una cuenta en [Docker Hub](https://hub.docker.com/) y subir nuestra propia imagen a los repositorios públicos.

### Pasos
1. Tener un Dockerfile con lo que necesitamos para crear nuestra imagen. 
2. Construir la imagen, probarla, y asegurarnos de que todo va correctamente. 
3. Tener una cuenta en Docker Hub. 
4. Crear una nueva instancia de la imagen pero con un tag diferente siguiendo esta estructura: ```docker build -t <username>/<tag> .```
5. Iniciamos sesión en nuestra cuenta de Docker Hub desde la terminal ejecutando: 	
	```docker login```.
6. Subimos nuestra la instancia de nuestra imagen ejecutando: 
	```docker push <username/tag>```
	
	
## Creación de tags 🏷
Si llegaramos a hacerle cambios a nuestros contenedores podríamos crear tags para almacenarlos en diferentes imágenes.

1. **Para crear un tag sólo debemos ejecutar**: 
```bash
$ docker tag <image-id>:<version> <name>
$ docker tag <image-id>:<version> <username>/<name> (el ID de la imagen y la versión son lo que hacen al tag) 
```

2. **Subir el tag a Docker Hub**:
```bash
$ docker login
$ docker push <username>/<name>:<version>
```


## Redes con Docker 💻⛓

Tener esto en cuenta nos permitirá crear una red de contenedores y enlazarlos entre sí los unos con los otros. 

1. **Crear una red en Docker**:
```bash
$ docker network create <network-name>
``` 

2. **Listar las redes que tengamos con Docker**:
```bash
$ docker network ls 
```

3. **Eliminar una red de Docker**:
```bash
$ docker network rm <network-name> 
```

2. **Correr un contenedor en una red Docker**:
```bash
$ docker run --network <network-name> --network-alias <container-name> <tag>
```
**Nota**: la opción ```--network-alias``` nos permitirá asignarle un dominio al contenedor en la red Docker. 
