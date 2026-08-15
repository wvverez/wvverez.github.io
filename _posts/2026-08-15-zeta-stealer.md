---
layout: single
title: Zeta Stealer - Un stealer para robar tokens de discord
excerpt: "¿Cómo de complejo suele ser para un atacante conseguir tokens de tus cuentas para ganar acceso completo?"
date: 2026-08-15
classes: wide
header:
  teaser: /assets/images/zeta.png
  teaser_home_page: true
categories:
  - Malware
  - Stealer
  - Discord
  - Tokens
  - Python
tags:
  - Malware
  - Stealer
  - Discord
  - Tokens
  - Python
---

<p align="center">
  <img src="/assets/images/zeta.png" alt="zeta" width="500">
</p>

En este nuevo post vamos a estar creando un sencillo `stealer` para robar tokens de sesión de `discord` en un sistema comprometido. Pero antes de ello vamos a definir que es un `stealer`.


## ¿Qué es exactamente un Stealer?

La palabra `stealer` se refiere a un tipo de `malware` diseñado específicamente para encontrar información **sensible** en sistemas ya comprometidos.

Un stealer es un software malicioso que, una vez ejecutado en el sistema de la víctima, tiene como objetivo robar credenciales como contraseñas guardadas en navegadores, extraer tokens de sesión como cookies y tokens de autenticación, obtener información financiera como datos de `cc`...

## ¿Cómo funcionan los Stealers en Discord?

El objetivo principal de estos stealers son los `tokens de sesión`. Discord utiliza tokens de autenticación para mantener a los usuarios conectados. Estos tokens actúan como una "llave maestra" que permite acceder a la cuenta sin necesidad de **usuario y contraseña**. Un token de `Discord` sigue la estructura de ID en Base64, seguido de un `timestamp` y una `firma`.

El stealer lee estos archivos y busca patrones específicos que coincidan con la estructura de los tokens de Discord, utilizando técnicas de decodificación para extraer el token completo. Finalmente, el token robado se envía al servidor del atacante, generalmente mediante solicitudes HTTP a un webhook de Discord. En el caso vamos a hacerlo para que lo muestre directamente por pantalla. Pero los cibercriminales operan con `webhooks`.

