# HTB / CTF Write-up

**Autor:** Santiago Fonseca  
**Máquina:** Niblles  
**IP (objetivo):** 10.129.200.170

**Fecha:** 10-10-2025  

---

## Objetivo
 * Gain a foothold on the target and submit the user.txt flag
 * Escalate privileges and submit the root.txt flag.

---

## Pasos a seguir

 1) Reconocimiento inicial

El objetivo del reconocimiento inicial es identificar superficies de ataque expuestas por el host.
Para ello se realiza primero un escaneo completo de puertos para obtener una visión global del sistema y luego un análisis más profundo de los servicios detectados.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.129.200.170 -oN puertos
```
* -p- Incluye todos los puertos TCP (1–65535).
* --open Solo muestra los puertos abiertos.
* -sS SYN scan (half-open): envía SYN y observa respuestas (SYN/ACK → abierto; RST → cerrado).
  Es rápido y evita completar conexiones completas con el objetivo. Requiere privilegios de root.
* --min-rate 5000 intenta enviar al menos 5000 paquetes por segundo (acelera mucho el escaneo).
* -n Acelera el escaneo y evita ruido de consultas DNS. No aplica resolucion DNS.
* -Pn No hacer ping previo; trata al host como “up”. Útil si el objetivo bloquea pings/ICMP o responde mal a sondas.
* -oN Guarda la salida en formato legible (puertos) para evidencias y posterior análisis.

```bash
┌─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $cat puertos 
# Nmap 7.94SVN scan initiated Sat Nov 15 15:21:56 2025 as: nmap -p- --open -sS --min-rate 5000 -n -Pn -oN puertos 10.129.200.170
Nmap scan report for 10.129.200.170
Host is up (3.6s latency).
Not shown: 59501 closed tcp ports (reset), 6032 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

# Nmap done at Sat Nov 15 15:22:28 2025 -- 1 IP address (1 host up) scanned in 32.47 seconds
```

El escaneo revela únicamente dos servicios accesibles: SSH (22) y HTTP (80).
Esto sugiere que el vector inicial probablemente se encuentre en el servicio web, ya que SSH requiere credenciales válidas.

 2) Utilizando la informacion anterior realizamos un escaneo mas profundo

Una vez identificados los puertos abiertos, se realiza un escaneo enfocado a enumerar versiones y scripts NSE para obtener información específica de los servicios detectados.

```bash
nmap -sCV -p22,80 10.129.200.170 -oN objetivos
```

* -sC: Ejecuta scripts NSE por defecto (detección estándar).

* -sV: Identifica versiones de los servicios.

* -p22,80: Limita el escaneo a los puertos ya descubiertos.

* -oN: Guarda la salida para analisis posterior.

```bash
┌─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $cat objetivos 
# Nmap 7.94SVN scan initiated Sat Nov 15 15:36:15 2025 as: nmap -sCV -p22,80 -oN objetivos 10.129.200.170
Nmap scan report for 10.129.200.170 (10.129.200.170)
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Nov 15 15:36:46 2025 -- 1 IP address (1 host up) scanned in 30.64 seconds
```
El servicio HTTP en el puerto 80 es la única superficie expuesta con potencial de explotación directa.
La versión de Apache y el contenido web deben analizarse para identificar posibles vectores de entrada.

 3) Analizamos el contenido de la web

```
┌─[✗]─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $curl http://10.129.200.170/
<b>Hello world!</b>





<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Al inspeccionar la página principal, se observa únicamente un mensaje básico y un comentario HTML que revela el directorio /nibbleblog/.
Lo cual sugiere que el sitio utiliza Nibbleblog, un sistema de gestión de contenido (CMS) gratuito y ligero, diseñado específicamente para la creación de blogs de forma sencilla y rápida. Esto orienta el ataque hacia ese subdirectorio.

Para identificar posibles fallos de seguridad en Nibbleblog, se consulta el repositorio de exploits públicos mediante:

```bash
searchsploit nibbleblog
```

El resultado es el siguiente:

```bash
┌─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $searchsploit nibbleblog
-------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                              |  Path
-------------------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                                      | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                                       | php/remote/38489.rb
-------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

El exploit más relevante es el de Arbitrary File Upload, presente en Nibbleblog 4.0.3, el cual permite la carga de archivos maliciosos, esto puede conducir a un RCE (Remote Code Execution).

 4) Descubrimiento de directorios con Wfuzz

Para enumerar directorios dentro del subdirectorio /nibbleblog/, utilicé Wfuzz, una herramienta de fuerza bruta para análisis web. El objetivo es identificar rutas ocultas que puedan exponer paneles de administración, módulos vulnerables o archivos sensibles.

```bash
┌─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $wfuzz -c --hc=404 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt  http://10.129.200.170/nibbleblog/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.129.200.170/nibbleblog/FUZZ
Total requests: 220559

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                      
=====================================================================

000000001:   200        60 L     168 W      2985 Ch     "# directory-list-2.3-medium.txt"                            
000000003:   200        60 L     168 W      2985 Ch     "# Copyright 2007 James Fisher"                              
000000007:   200        60 L     168 W      2985 Ch     "# license, visit http://creativecommons.org/licenses/by-sa/3
                                                        .0/"                                                         
000000259:   301        9 L      28 W       327 Ch      "admin"                                                      
000000519:   301        9 L      28 W       329 Ch      "plugins"                                                    
000000013:   200        60 L     168 W      2985 Ch     "#"                                                          
000000012:   200        60 L     168 W      2985 Ch     "# on at least 2 different hosts"                            
000000014:   200        60 L     168 W      2985 Ch     "http://10.129.200.170/nibbleblog/"                          
000000009:   200        60 L     168 W      2985 Ch     "# Suite 300, San Francisco, California, 94105, USA."        
000000005:   200        60 L     168 W      2985 Ch     "# This work is licensed under the Creative Commons"         
000000004:   200        60 L     168 W      2985 Ch     "#"                                                          
000000008:   200        60 L     168 W      2985 Ch     "# or send a letter to Creative Commons, 171 Second Street," 
000000010:   200        60 L     168 W      2985 Ch     "#"                                                          
000000002:   200        60 L     168 W      2985 Ch     "#"                                                          
000000006:   200        60 L     168 W      2985 Ch     "# Attribution-Share Alike 3.0 License. To view a copy of thi
                                                        s"                                                           
000000011:   200        60 L     168 W      2985 Ch     "# Priority ordered case-sensitive list, where entries were f
                                                        ound"                                                        
000000897:   200        63 L     643 W      4624 Ch     "README"                                                     
000000935:   301        9 L      28 W       331 Ch      "languages"                                                  
000000075:   301        9 L      28 W       329 Ch      "content"                                                    
000045240:   200        60 L     168 W      2985 Ch     "http://10.129.200.170/nibbleblog/"                          
000000127:   301        9 L      28 W       328 Ch      "themes"                                                     
000068675:   404        9 L      32 W       298 Ch      "895929041"                                                  

Total time: 0
Processed Requests: 68572
Filtered Requests: 68551
Requests/sec.: 0
```

* -c: salida coloreada para mayor legibilidad.

* --hc=404: oculta las respuestas con código 404 (no encontrado).

* -t 200: usa 200 hilos para acelerar la enumeración.

* -w …directory-list-2.3-medium.txt: diccionario ampliamente utilizado en enumeración web.

* FUZZ: palabra clave que Wfuzz sustituye en cada petición.

También realicé fuzzing orientado a encontrar archivos PHP dentro del CMS

```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt  http://10.129.200.170/nibbleblog/FUZZ.php
```

El archivo admin.php confirma la presencia de un panel de login activo:

http://10.129.200.170/nibbleblog/admin.php

Este será un punto clave para la explotación.

 5) Analisis

Dentro del directorio /content/, específicamente en la ruta: /content/private/users.xml

Encontramos lo siguiente:

```xml
<users>
  <user username="admin">
    <id type="integer">0</id>
    <session_fail_count type="integer">0</session_fail_count>
    <session_date type="integer">1514544131</session_date>
  </user>
  <blacklist type="string" ip="10.10.10.1">
    <date type="integer">1512964659</date>
    <fail_count type="integer">1</fail_count>
  </blacklist>
</users>
```

Todo apunta a la existencia de un usuario llamado admin

 6) Explotacion

Una vez identificado el panel de login en /nibbleblog/admin.php, probé credenciales básicas para el usuario admin.
La contraseña correcta resultó ser: nibbles
Este tipo de credenciales débiles es común en máquinas de entrenamiento de HTB y refleja un escenario de mala configuración.

En sttings descubri que la version de nibbleblog que se esta utilizando es la 4.0.3, la cual, es vulnerable a Arbitrary File Upload. 

 Investigamos el código para ver como funciona la explotación 

 ```bash
┌─[✗]─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $searchsploit nibbleblog
-------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                              |  Path
-------------------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                                      | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                                       | php/remote/38489.rb
-------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
┌─[santiago@parrot]─[~/Desktop/Maquinas/nibbles/practica2]
└──╼ $searchsploit -x php/remote/38489.rb
```
La explotacion conciste en subir un archivo .php llamado image.php al directorio /content/private/plugins/my_image/

Fragmento clave del exploit:
```bash
php_fname = 'image.php'
    payload_url = normalize_uri(target_uri.path, 'content', 'private', 'plugins', 'my_image', php_fname)
    vprint_status("#{peer} - Parsed response.")
```

El plugin my_image no valida adecuadamente el tipo de archivo, por lo que puedo subir un archivo con codigo malicioso.

Se puede acceder directamente al archivo subido y ejecutarlo en el servidor, al abrir ese archivo mediante una petición HTTP, se ejecuta el payload y se obtiene ejecución remota de comandos (RCE).





