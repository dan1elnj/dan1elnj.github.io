---
title: "TryHackMe - Neighbour"
date: 2025-10-15
lastmod: 2025-10-15
categories: [TryHackMe]
tags: [Web, TryHackMe, CTF, Linux, IDOR]
toc: true
math: false
mermaid: true
image:
  path: assets/img/TryHackMe/Neighbour/logo.png
---

## Target
- **Host:** `http://neighbour.thm`  
- **Plataforma:** TryHackMe   
- **Nivel:** <span style="color: #66bb6a;">Easy</span>
- **Objetivo:** Encontrar la flag.

---

## Introduccion

Es una maquina muy sencilla, donde se filtran en comentarios unas credenciales.

## Enumeración

No hace falta enumerar puertos ya que esta maquina se accede a traves del servicio http, es una maquina web.

### Enumeración puerto 80

Web Principal

Nos muestra un panel de autenticacion.
![Web principal](assets/img/TryHackMe/Neighbour/website.png)

Revisamos el codigo fuente de la pagina con Ctrl+U y muestra unas credenciales.
![sourcecode](assets/img/TryHackMe/Neighbour/sourcecode.png)

```html
<!-- use guest:guest credentials until registration is fixed. "admin" user account is off limits!!!!! -->
```
- Username: `guest`
- Password: `guest`

Aparte aparece un usuario llamado `admin`.

Al iniciar sesión con guest:guest, el sistema nos da la bienvenida.
![guest](assets/img/TryHackMe/Neighbour/guest.png)

Tras iniciar sesión como guest, notamos que la URL contiene un parámetro que referencia directamente nuestra identidad, lo que indica un posible Insecure Direct Object Reference (IDOR).
El parámetro user se utiliza para cargar el contenido de la página. Dado que este parámetro se alimenta directamente de la entrada del usuario sin una validación de autorización adecuada, intentamos manipularlo.
Cambiamos el valor del parámetro user de guest al usuario que encontramos en el código fuente: `admin`.

Al cargar la nueva URL, el servidor devuelve el perfil del usuario admin, que incluye la flag.
![admin](assets/img/TryHackMe/Neighbour/admin.png)


Se encontró la flag aprovechando una vulnerabilidad de IDOR en el parámetro user tras una autenticación exitosa.
- `flag{66be95c478473d91a5358f2440c7af1f}`