---
layout: single
title: Buffer Overflow ret2libc bypasseando ASLR
excerpt: "Cómo abusar de un buffer overflow para retornar a ejecución remota de código saltandose la protección ASLR"
date: 2026-08-31
classes: wide
header:
  teaser: /assets/images/buffer.jpeg
  teaser_home_page: true
categories:
  - Ret2libc
  - Buffer Overflow
  - Linux
  - ASLR 
  - Libc
  - strcpy
tags:
  - Ret2libc
  - Buffer Overflow
  - Linux
  - ASLR
  - Libc
  - strcpy
  - SUID
  - C code
---

<p align="center">
  <img src="/assets/images/buffer.jpeg" alt="buffer" width="500">
</p>

---

<div class="notice" style="border-left: 4px solid #ff4444; background-color: #2a1515; color: #ffdddd;">
  <strong style="color: #ff4444;">[+] DISCLAIMER</strong><br><br>
  Todo el contenido, información, ejemplos o conocimientos presentados en este sitio tienen <strong>exclusivamente fines educativos, formativos y éticos.</strong><br><br>
  El objetivo es promover el aprendizaje y la comprensión responsable de los conceptos explicados. <strong>No se pretende fomentar, facilitar ni promover actividades ilegales, dañinas o no autorizadas.</strong>
</div>


# Definición de la vulnerabilidad

`Ret2libc` o (Return to libc) es una técnica de las más complejas a la hora de explotar `buffer overflows` que se utiliza para saltarse protecciones en un programa para ejecutar código malicioso de forma remota. Este tipo de ataque suele concatenarse una vez tenemos un buffer overflow en algún binario cuando no podemos ejecutar shellcodes debido a que estéa protegido con protecciones como `NX`.

Lo que pasa en esta vuln es que en vez de meter nuestra `shellcode` podemos aprovecharnos de las llamadas que hacen a las funciones de la biblioteca estándar **libc** que ya están cargadas en la memoria de el binario.

Para los que no sepáis para que sirve **libc**, es una parte fundamental de la programación en lenguaje `C` que proporciona muchas funciones que son muy usadas.

Y por último entender que hace la protección `ASLR`. 

`ASLR` es una protección que consiste en aleatorizar la ubicación de las áreas clave de memoria en un binario una vez en ejecución. Lo cual complica mucho abusar de el binario.

Para cualquier usuario que quiera activarlo para hacer pruebas necesitas `sudo` pero se activaría así:

```sh
echo 2 > /proc/sys/kernel/randomize_va_space
```

Una vez ejecutes esto tendrás aleatoridad en las direcciones de `memoria` que es una capa extra de protección para evitar **buffer overflows**.

De hecho si nos fijamos en el `binario`, que vamos a crear más adelante nos podemos fijar en la segunda línea la dirección de memoria de `libc` y que cada vez va a cambiar.

```sh
# ldd wvverez
        linux-gate.so.1 (0xf7ed4000)
        libc.so.6 => /usr/lib/i386-linux-gnu/libc.so.6 (0xf7c7f000)
        /lib/ld-linux.so.2 (0xf7ed6000)

```

# Creando binario vulnerable

Para explicarlo mejor vamos a crear el siguiente archivo en C

```sh
#include <stdio.h>

void vuln(char *buff) {
    char buffer[64];
    strcpy(buffer, buff);
}

void main(int argc, char **argv) {
    vuln(argv[1]);
}
```

Este sería el código al estar en C deberemos compilarlo y después veremos como abusar de está técnica. Para poder compilar el binario lo hacemos de la siguiente forma:

```sh
gcc -fno-stack-protector -m32 vuln.c -o wvverez
```

Bien una vez tenemos todo listo, antes de nada vamos a entender como funciona el flujo de el binario. Primeramente va a esperar que el usuario introduzca datos como `parámetro` en el que lo que hace es copiar lo que le pasamos como argumento como una variable que la hemos llamado `buffer` con una función que por ahora desconocereis llamada "strcpy". 


Bueno esta función `strcpy` en C y C++ es bastante **vulnerable**. Lo que pasa con esta función es que no realiza ninguna verificación de los tamaños que se pasan de los buffers. Y esto concatena con `buffer overflows`.

Para poder entender mejor una configuración vulnerable de este binario. Vamos a darle permisos SUID y propietario le asignaremos como `root`.

```sh
chown root:root wvverez
chmod u+s wvverez
```

Cuando hacemos esto lo que estamos indicando es que cada vez que lo ejecute un usuario físico cualquiera, lo va a estar ejecutando como el `propietario` de dicho archivo, es decir `root`.

Una vez listo podemos probar el propio **binario**. Aquí comparto alguna prueba de su flujo de ejecución y de la propia vulnerabilidad.

```sh
# ./wvverez "Hola"

# ./wvverez $(python3 -c 'print("A"*1000)')
Segmentation fault         ./wvverez $(python3 -c 'print("A"*1000)')
```

En la primera vez que lo hemos ejecutado lo que ha pasado es que en la variable **buffer** que le dimos un tamaño de `64` bytes, almacena lo que le pasemos como argumento, en nuestro caso "hola".

Y en el segundo caso le estamos pasando 1000 letras A con `python` y de esa forma intenta almacenar más datos de los que tiene definidos esa variable. Y por ello corrompe por eso nos da un "Segmentation Fault".

Una vez confirmado la vulnerabilidad lo primero que suelo hacer es revisar las `protecciones` de el binario para poder entender y saber mejor por donde podemos tirar.

```sh
─# checksec --file=wvverez
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   No canary found   NX enabled    PIE enabled     No RPATH   No RUNPATH   41 Symbols        No    0               1               wvverez

```

Confirmamos que no está protegido por `canarios` y que tiene NX habilitado es decir que el stack no es ejecutable por lo cual no vamos a poder utilizar ningún `shellcode` y que también tiene PIE habilitado es decir que las direcciones van variando.

Con lo cual sabemos que no podemos ejecutar código directamente en la pila (esp). Es aquí donde podemos apoyarnos de **ret2libc** para abusar de las funciones de la biblioteca `libc`.

Antes de entrar a fondo quiero tocar un poco más en detalle lo que está pasando cuando sobre-escribimos `datos`. Ya que cuando desbordamos el buffer estamos sobre escribiendo `registros` para 32 bits que fue para lo que lo compilamos. Es decir arquitectura **x86**. 

Tenemos 3 que son de los primordiales para x86:

`ESP`: Que apunta a lo más alto de la pila

`EBP`: Marca el inicio de la pila

`EIP`: Es la siguiente dirección en memoria a la que va a retornar nuestro programa

<p align="center">
  <img src="/assets/images/pila.png" alt="buffer" width="500">
</p>

Ya digo que existen muchos más registros que no estamos teniendo en cuenta pero para este ejemplo con localizar estos 3 `registros` para abusar de el buffer overflow `ret2libc`.

Empieza por el `ESP` y continua sobre escribiendo el resto de registros.

Para poder ir viendo como se van sobre escribiendo estos `registros` existe `gdb`, que nos permite analizar ese binario a `bajo nivel` como funciona todo el binario.

Lo primero que se suele hacer para explotar un **buffer overflow** es calcular el número exacto de bytes que hay que pasar para llegar a el `EIP`.

Para esto podemos probar lo siguiente:

```sh
# gdb -q wvverez
Reading symbols from wvverez...
(No debugging symbols found in wvverez)
(gdb) r $(python3 -c 'print("A"*100)')
Starting program: /root/wvverez $(python3 -c 'print("A"*100)')
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
(gdb)
```

Como podemos ver `0x41414141` son basícamente nuestras "A" que sobre escribimos en el registro `EIP`. Esto es mucho más visual si le metemos al gdb alguna extensión como el `peda` o `gef`. Ya que así solo nos va a mostrar el EIP y si quisieramos ver más registros es recomendable meterle alguno de ellos.

Para este ejemplo os voy a enseñar a hacerlo con `gef` que personalmente es mi preferido primero para añadirlo necesitais ejecutar lo siguiente:

```sh
# bash -c "$(curl -fsSL https://gef.blah.cat/sh)"
[+] Added GEF source to /root/.gdbinit
```

Una vez añadido si ahora volvemos a probar a ejecutarlo y volver a pasarle como input las `A` deberíamos poder visualizar el resto de registros sobre escritos sin contar EIP (Ya que ese ya lo podíamos visualizar sin gef).

```sh
# gdb -q wvverez
[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0xffffd1c0  →  "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA[...]"
$ebx   : 0x41414141 ("AAAA"?)
$ecx   : 0xffffd500  →  0x48530041 ("A"?)
$edx   : 0xffffd223  →  0xffd20041 ("A"?)
$esp   : 0xffffd210  →  "AAAAAAAAAAAAAAAAAAAA"
$ebp   : 0x41414141 ("AAAA"?)
$esi   : 0x0
$edi   : 0xf7ffcc60  →  0x00000000
$eip   : 0x41414141 ("AAAA"?)
$eflags: [zero carry PARITY adjust SIGN trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x23 $ss: 0x2b $ds: 0x2b $es: 0x2b $fs: 0x00 $gs: 0x63
───────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0xffffd210│+0x0000: "AAAAAAAAAAAAAAAAAAAA"       ← $esp
0xffffd214│+0x0004: "AAAAAAAAAAAAAAAA"
0xffffd218│+0x0008: "AAAAAAAAAAAA"
0xffffd21c│+0x000c: "AAAAAAAA"
0xffffd220│+0x0010: "AAAA"
0xffffd224│+0x0014: 0xffffd200  →  "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
0xffffd228│+0x0018: 0x56558eec  →  0x56556130  →  <__do_global_dtors_aux+0000> endbr32
0xffffd22c│+0x001c: 0xf7d99f21  →   add esp, 0x10
─────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x41414141
─────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "wvverez", stopped 0x41414141 in ?? (), reason: SIGSEGV
───────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤ 
```

Personalmente creo que la diferencia es considerable. Con lo cual vemos que esta sobre escribiendo más registros como `EBP`, `ESP` Y `EIP` vemos que apunta `EIP` a 0x414141414141 que son basícamente nuestras "A".

# Calculando el offset de el binario

Bien en este punto es interesante que podamos conocer 
