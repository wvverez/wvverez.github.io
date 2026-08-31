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
---

<p align="center">
  <img src="/assets/images/buffer.jpeg" alt="buffer" width="500">
</p>

---

# Definición de la vulnerabilidad

`Ret2libc` o (Return to libc) es una técnica de las más complejas a la hora de explotar `buffer overflows` que se utiliza para saltarse protecciones en un programa para ejecutar código malicioso de forma remota. Este tipo de ataque suele concatenarse una vez tenemos un buffer overflow en algún binario cuando no podemos ejecutar shellcodes debido a que estéa protegido con protecciones como `NX`.

Lo que pasa en esta vuln es que en vez de meter nuestra `shellcode` podemos aprovecharnos de las llamadas que hacen a las funciones de la biblioteca estándar **libc** que ya están cargadas en la memoria de el binario.



