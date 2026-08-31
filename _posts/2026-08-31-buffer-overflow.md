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

Para poder entender mejor una configuración vulnerable de este binario. Vamos a darle permisos SUID y grupo le asignaremos como `root`.

```sh
chown root:root wvverez
chmod u+s wvverez
```

Cuando hacemos esto lo que estamos indicando es que cada vez que lo ejecute un usuario físico cualquiera, lo va a estar ejecutando como el `propietario` de dicho archivo, es decir `root`.

