---
title: Mi experiencia con el eWPTXv2 🏴‍☠️
author: sha16
categories: [reviews, certifications, web, elearnsecurity]
tags: [reviews, certifications, web, elearnsecurity] 
---

Hola a tod@s! Muchas gracias por darse el tiempo de pasarse por mi blog.

Por aquí les traigo una review acerca de mi experiencia rindiendo el **eWPTXv2** de **eLearnSecurity**.

![Untitled](/assets/img/reviews/review-ewptx/Untitled.png)

## Introducción 🥂

Bueno, de ante mano para quienes no sepan qué es el eWPTXv2, esta es la versión 2 ("*eXtreme*") de la certificación de pentesting web de **eLearnSecurity**, desglosando sus siglas en *eLearnSecurity WebApp Penetration Tester eXtreme version 2*.

Como ya muchos sabrán, el examen es totalmente práctico y tiene un valor de $400 USD, aunque para abaratar costes siempre les recomiendo mirar en la web si es que hay algún código de descuento *leaked* (*dinero es dinero, ya saben eh*).

Al igual que en otros exámenes, eLearnSecurity te da 14 días para rendir el examen: 7 días para explotar el laboratorio, donde tendrás que cumplir con los **3 requisitos mínimos pero no suficientes para aprobar**. Por otro lado, se te darán 7 días más para escribir un reporte profesional de vulnerabilidades, el cual tendrás que subir a la plataforma para que te evalúen. 

## Requerimientos mínimos pero no suficientes 🗿

A la hora de arrancar el examen, eLearnSecurity te una carta de engagement, donde se define el scope (3 aplicaciones web) sobre el que deberás explotar y encadenar “Xs” vulnerabilidades, con las cuales podrás lograr los 3 requisitos mínimos pero no suficientes para aprobar:

1. Leer un archivo interno del servidor.
2. Lograr ejecución remota de comandos mediante la explotación de un servicio interno en la máquina.
3. Lograr ejecución remota de comandos mediante la explotación de un servicio interno en la máquina (debes explotar otra vulnerabilidad).

Es importante que tengas en cuenta que son **no suficientes** para aprobar, por lo que te recomiendo ser bien busquilla al rendir el examen.

## Requisitos 🐱‍👤

Bueno, los requerimientos para esta certificación examen son bastante amplios. Así que iré detallándolos uno por uno.

### Entender cómo funciona una aplicación web 📌

Puede parecer algo tonto mencionar esto, pero créanme que no lo es. Para este examen es necesario, más bien dicho es crucial, entender bien cómo funciona una aplicación web, no pueden existir dudas básicas del tipo: *cómo funcionan los procesos de autenticación*, *cifrado de cookies*, *llamadas a la base de datos*, etc. 

Entender cómo está construida una aplicación web, tanto en el frontend como el backend, te permitirá encarar mucho mejor la certificación de cara a identificar vectores de ataque, enumeración, explotación, etc.

En síntesis, es muy importante que estés pulido con los conceptos básicos de la web y el funcionamiento de las aplicaciones. Para reforzar todo esto, les recomiendo encarecidamente que se monten una aplicación web simple, ya sea en PHP, Python, o lo que quieran, y que procuren que esta cuente con mecanismos de autenticación y autorización básicos, para que logren entender por sobre todo como se conecta el backend con el frontend, cómo se consulta una base de datos, cómo funciona un procesos de autenticación, cómo se cifran las cookies, etc.

### Técnicas de enumeración 📌

De cara al examen, es super importante que sepan emplear técnicas de enumeración ofensivas: fuzzing, volcar directorios `/.git/`, etc., Lo otro importante es usar el típico guessing (o pensamiento lateral) de los CTFs, en el que cualquier cosa que encuentres *aquí*, probablemente también la encuentres en otra ruta o en la raíz del servidor web.

Por experiencia les digo que esto último se adquiere jugando con CTFs, o simplemente probando *a lo loco*, pero para hacerlo sencillo siempre mantengan en mente lo siguiente: *lo que encuentres aquí también podría estar en este otro lugar*.

### Entender las vulnerabilidades y saber cómo explotarlas 📌

A la hora de estudiar y practicar la explotación de vulnerabilidades web, es de carácter fundamental que tengan una idea más o menos clara de cómo se origina el hallazgo en el lado del backend. Por ejemplo, si encuentran una SQLi, es casi seguro que es porque: el input del usuario está mal sanitizado en el backend, las consultas a la base de datos no están parametrizadas, etc.

Con esto me refiero a que entiendan el hallazgo que tienen enfrente, cosa de saber qué tipo de técnica deben emplear o qué payload deben meter para hacer efectiva la explotación. Todo se resume en: *existen múltiples formas de explotar una vulnerabilidad, pero debes entenderla para tener éxito*.

Para el examen de la certificación, no sirve tener una idea vaga de vulnerabilidades web, o haberlas explotado sólo 3 veces en tu vida.

Para que se hagan un idea, recuerdo que ninguna de las *SQLi* que exploté, la encontré con el típico payload `" OR 1=1-- -`.

### Saber leer código fuente 📌

Desde mi punto de vista, es re importante que de cara al examen, sepas leer código fuente, mínimo: PHP, JavaScript y Python.

Esto te dará un comprensión mucho mejor acerca del panorama en general de una aplicación web. Si sabes cómo leer código serás capaz de detectar un vector de ataque mucho más rápido. Lo otro es que te será mucho más fácil explotarlo, o armarte una idea de cómo hacerlo.

Seguro pensarás que ChatGPT puede hacer esto, pero créeme que a veces no ☢ Lo otro, es que igual no veo mucho mérito en hacer el examen con la IA, se supone que la gracia de un examen es echar a andar la cabeza para completarlo. Con esto no digo que este mal usar IAs, pero si que tiene mucho más sentido que te la busques por tu cuenta antes que nada.

La verdad, no quiero explayarme más en este punto, te aconsejo por sobre todo que estudies PHP y JavaScript, ya que es lo que principalmente se ve en el examen.

### Vulnerabilidades Importantes 📌

No quiero dar spoilers, pero las vulnerabilidades que deben tener bien repasadas para el examen son las siguientes:

- SQL Injection
- Cross-Site Scripting (XSS)
- XML External Entity (XXE)
- Server-Side Request Forgery (SSRF)
- Cross-Site Request Forgery (CSRF)
- Server-Side Template Injection (SSTI)
- PHP Object Injection
- Java Deserialization
- WebApp Misconfigurations
- Entre otras.

## Recomendaciones 🏴‍☠

Buenos en base a mi experiencia mis recomendaciones de cara al examen son las siguientes:

- Estudia y ve tranquilo, no te apures para rendir la certificación, todo a su tiempo. El examen no es fácil, es bastante busquilla de hecho, por lo tanto ríndelo cuando verdaderamente te sientas listo. Con esto no digo que esperes toda tu vida, pero si digo que tengas la seguridad suficiente de lo que sabes como para lanzarte.
- Existen 3 requisitos mínimos pero no suficientes para aprobar. Por experiencia yo creo que deberían ser 4: `UmVwb3J0YSBtw6FzIGRlIDIgU1FMaSEgRGUgbG8gY29udHJhcmlvIHRlIHJlcHJvYmFyw6FuIDonYw==`
- *SQLi que encuentras, SQLi que reportas*. Da igual si no tienes data que volcar, repórtalo igual o te reprobarán ☠
- Aprende a explotar SQLi con SQLMap, usando tokens CSRF.
- No reportes sólo las vulnerabilidades altas y/o críticas, también contempla hallazgos de criticidad media y/o baja, que por lo común son: clickjacking, ausencia de certificado SSL, etc., no diré más jeje. Podría decirse que se relacionan con la ausencia de implementaciones de seguridad importantes, pero no “críticas”.
- Estudia la sintaxis de PHP, por lo menos en temáticas de criptografía y procesos de autenticación.
- No quiero dar spoilers, pero te aconsejo completar los laboratorios de PortSwigger relacionados con **PHP Object Injection**, lo que abordes en ellos te será de bastante ayuda de cara al examen.
- PortSwigger es tu mejor amigo de cara al estudio.
- Si te estancas en el examen, te aconsejo resetear el laboratorio ya que por inestabilidad suele arrojar falsos negativos 🙃
- Considerando el punto anterior, si estás seguro de que estás yendo por buen camino, confía en tu corazón y prueba múltiples formas de explotar una vulnerabilidad. Si es necesario, reinicia el laboratorio.
- ¡Practica! El examen no está regalado.

## Experiencia Personal 🗿

Bueno, ya llegando al final de este post, puedo decirles que este examen no fue la mejor experiencia que he tenido, de hecho la pasé bastante mal ya que reprobé en mi primer intento.

#### Día 0: Me entran las ganas de rendir el examen

Todo empieza en Marzo 2023, cuando le digo a mi amigo [H4rr1z0n](https://github.com/Harrizzon): *“mano, voy a sacar el eWPTXv2 en un mes”*. Y así fue, sólo que lo saqué en un mes y medio.

Para empezar a prepararme agarré todos los recursos de PortSwigger que se relacionaban con las vulnerabilidades que entrarían en el examen.

Ya con esto, resolví cada laboratorio, procurando tomar nota de cada uno y entender qué es lo que hacía cada payload que fuese a meter.

Otra cosa que me sirvió mucho fue montarme mini aplicaciones vulnerables, a modo de entender cómo se originaban las vulnerabilidades y así darle un poco de más sentido a los payloads en mi cabeza.

Ya con esto, el resto lo dejé medio a la vida y me apoyé más de la experiencia previa que había ido adquiriendo en el trabajo y en otros laboratorios que había hecho en HackTheBox y TryHackme (más de 150 máquinas).

#### Día D: Experiencia y resultados del examen

En total, resolviendo el laboratorio en el primer intento me demoré 3 días. En este plazo ya tenía los 3 requisitos mínimos pero no suficientes para aprobar, junto con otro puñado de vulnerabilidades, medias y bajas, para reportar.

Ya con eso, me armé un reporte de estilo profesional de unas 100 páginas aprox, con todo lo que este requiere, lo más detallado posible. Hecho esto, lo mandé, esperé dos semanas aprox, miro el correo y me entero de que me habían reprobado 😐

En primera instancia no entendía por qué, me sentí super frustrado, confundido y me cuestionaba mucho qué es lo que había hecho mal, o qué me había faltado. Al mirar el feedback de eLearnSecurity, me indicaban que me habían faltado más casos de SQL Injection por reportar.

Entonces, arranqué el laboratorio de nuevo, encontré estas SQLi, las documenté y volví a mandar el reporte.

Habiendo hecho esto, me quedé un poco enojado, ya que los casos que me faltaron de SQLi, correspondían a casos de la vulnerabilidad en los que no se podía volcar ningún tipo de información del servidor, lo cual me hacía pensar que podían ser fácilmente descartables como falsos positivos, y/o también considerarse como *sin sentidos* dentro del examen. Pero estoy seguro de que lo digo más por lo emputado que quedé, que por lo que fue 🦆 

Pero bueno, a fin de cuentas lo medio entendí, hice las pases con eLearn, sobre todo considerando que las vulnerabilidades no eran fáciles de encontrar ni de explotar. 

Ya con esto, esperé otras 3 semanas y finalmente recibí el típico correo con el mensaje de felicitaciones de eLearnSecurity por haber aprobado el examen 😎 La verdad me sentí feliz y aliviado por finalmente haberlo sacado después de todo lo que había pasado, sin embargo, lamentablemente, igual me quedaron esas *bad-vibes*, con un sabor agrio en la boca, por el mal rato que pasé. Pero bueeeno, es lo de menos, se peleó, se aprobó y se celebró.

¡Muchaaas gracias por leer la review, cualquier cosa estamos al habla por LinkedIn, Twitter o Discord! ❤

Happy Hacking 🏴‍☠️