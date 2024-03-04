---
layout: single
title: Waldo - Hack The Box
excerpt: "Para obtener acceso a la máquina, debemos obtener las claves privadas de SSH jugando con dos peticiones mediante BurpSuite, en las cuales cada una es vulnerable a Insufficient Input Validation, lo que permite explotar un Local File Inclusion - LFI via filter bypass y un Lirectory Listing via filter bypass, teniendo la posibilidad de obtener las claves privadas de SSH. Una vez dentro del sistema, debemos bypasear una rbash (restricted bash) para luego escalar privilegios mediante el abuso capabilities."
date: 2024-03-04
classes: wide
header:
  teaser: /assets/images/HTB/writeup-waldo/Waldo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Medium
  - CFTs
tags:
  - Pentesting
  - Linux
  - Insufficient Input Validation
  - Local FIle Inclusion
  - Diretory Listing
  - SSH Key
  - Priesc Capabilities
---

![](/assets/images/HTB/writeup-waldo/Waldo.png)

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Waldo.pdf)

---

# **Índice**

1. [Introducción](#introduccion)
  * [Scope](#scope)
  * [Metodologia Aplicada](#metodologia-aplicada)
2. [Reconocimiento - Enumeración](#reconocimiento-enumeracion)
  * [Uso de la Herramienta Nmap](#uso-de-la-herramienta-nmap)
  * [Reconocimiento de la web con proxy de Burp encendido](#reconocimiento-de-la-web-con-proxy-de-burp-encendido)
3. [Análisis de Vulnerabilidades](#analisis-de-vulnerabilidades)
  * [Directory Listing via Filter Bypass](#directory-listing-via-filter-bypass)
  * [Local File Inlcusion (LFI) via Filter Bypass](#local-file-inclusion-lfi-via-filter-bypass)
4. [Explotación](#explotacion)
  * [SSH Private key](#ssh-private-key)
  * [Conexion a la Máquina via SSH](#conexion-a-la-maquina-via-ssh) 
  * [Pivoting: nobody ---> monitor (escapando del contenedor)](#pivoting-nobody-monitor-escapando-del-contenedo) 
  * [Bypass Restricted Bash](#bypass-restricted-bash) 
5. [Escalada de Privilegios](#escalada-de-privilegios)
  * [Privesc Mediante el Abuso Capabilities](#privesc-mediante-el-abuso-capabilities)  
6. [Conclusión Final](#conclusion-final)
7. [Apéndice I Links de Referencia](#apendice-i-links-de-referencia)
  * [Herramientas Utilizadas en la Auditoria](#herramientas-utilizadas-en-la-auditoria)
  * [Documentación](#documentacion) 

---

# Introducción 📄 [#](#introduccion) {#introduccion}
En el presente Write-Up explicare los pasos para resolver la máquina <a href="https://app.hackthebox.com/machines/Waldo"><span style="color: #01a0cf">**W**</span><span style="color: #01a0cf">**a**</span><span style="color: gray">**l**</span><span style="color: red">**d**</span><span style="color: red">**o**</span></a> de la plataforma [HackTheBox](https://hackthebox.com).
Para obtener acceso a la máquina, debemos obtener las claves privadas de SSH jugando con dos peticiones mediante BurpSuite, en las cuales cada una es vulnerable a **Insufficient Input Validation**, lo que permite explotar un **Directory Listing via Filter Bypass** y un **Local File Inclusion - LFI via Filter Bypass**, teniendo la posibilidad de obtener las claves privadas de SSH. Una vez dentro del sistema, debemos bypasear una rbash (restricted bash) para luego escalar privilegios mediante el **abuso capabilities**.

## Scope 🎯 [#](#scope) {#scope}
El scope de esta máquina fue definida como la siguiente.

| **Servidor Web / Direcciónes IPs / Hosts / URLs** |           **Descripción**           | **Subdominios** |
|:-------------------------------------------------:|:-----------------------------------:|:---------------:|
|                   10.129.229.141                  | Dirección IP de la máquina Waldo |      Todos      |

## Metodologia Aplicada 📦​​📚​​ [#](#metodologia-aplicada) {#metodologia-aplicada}
* Enfoque de prueba: En el proceso de pruebas de seguridad, se optó por un enfoque gray-box, lo que significó que se tenía un nivel de acceso parcial a la infraestructura y el sistema objetivo.
* Las etapas aplicadas para esta auditoria fueron las siguientes:

![](/assets/images/HTB/writeup-waldo/Etapas-pentest.png)

# Reconocimiento - Enumeración 🔍 [#](#reconocimiento-enumeracion) {#reconocimiento-enumeracion}
## Uso de la Herramienta Nmap 👁️ [#](#uso-de-la-herramienta-nmap) {#uso-de-la-herramienta-nmap}
Primeramente realizamos un escaneo con ayuda de la herramienta nmap en búsqueda de puertos abiertos.

```bash
# Primer escaneo con nmap.
--------------------------------------------------------------
nmap -p- --open --min-rate 5000 -n 10.129.229.141
```

|    **Parámetro**    |                                   **Descripción**                                   |
|:-------------------:|:-----------------------------------------------------------------------------------:|
|       **-p-**       |                              Escanea los 65535 puertos.                             |
|      **\--open**     |                          Muestra solo los puertos abiertos.                         |
| **\--min-rate 5000** |   Establece la velocidad mínima de envío de paquetes a 5000 paquetes por segundo.   |
| **-n** |   Indica a Nmap que no realice la resolución inversa de DNS. Por lo tanto, el escaneo no incluirá la resolución de nombres de host para las direcciones IP encontradas durante el escaneo, esto acelera el proceso de escaneo.   |

El resultado que nos arrojó este primer escaneo fue que la máquina tiene el puerto **22** que pertenece al servicio *SSH* y el puerto **80** que pertenece al protocolo *HTTP*.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/nmap_1.png" alt="Primer escaneo.">
</div>

Se procedió a realizar otro escaneo con los scripts default de nmap, también especificando la versión nuevamente.

```bash
# Segundo escaneo.
--------------------------------------------------------------
nmap -p22,80 -sC -sV 10.129.229.141
```

| **Parámetro** |                                   **Descripción**                                   |
|:-------------:|:-----------------------------------------------------------------------------------:|
|     **-p**    |                      Especifica los puertos que se escanearán.                      |
|    **-sC**    |                   Realiza un escaneo con los scripts por defecto.                   |
|    **-sV**    | Determina la versiones de los servicios que se ejecutan en los puertos encontrados. |

Lo único interesante que obtenemos es el título **List Manager** de la página web.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/nmap_2.png" alt="Resultado del segundo escaneo.">
</div>

## Reconocimiento de la web con proxy de Burp encendido 🌐🐙​ [#](#reconocimiento-de-la-web-con-proxy-de-burp-encendido) {#reconocimiento-de-la-web-con-proxy-de-burp-encendido}
Vamos a ver la página web para ver qué contiene, pero con el proxy de <span style="color:orange"> Burp </span> encendido, para ver qué recolectamos.

Jajaja, al entrar vemos un fondo de Wally, jajaja. Alguien ya lo encontró, yo sí. No se vale hacer trampa...

Bueno, sigamos.

Al entrar en la página web, vemos que hay un administrador de listas, donde podemos agregar listas o eliminar las dos por defecto.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/pagina-web-1.png" alt="Web.">
</div>

También podemos ingresar a las listas. Hagamos una prueba e ingresemos para comprobar qué hay en la list1.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/pagina-web-2.png" alt="Contenido de la list1.">
</div>

Más de lo mismo, vemos que adentro tiene algunos items, donde se pueden borrar y agregar más.

También podemos cambiar el nombre de cada item.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/pagina-web-3.png" alt="Editando nombre de item.">
</div>

Pero bueno, nada interesante. Lo único divertido es el fondo de Wally de una de sus historietas. Vamos a ver el HTTP History para ver qué encontramos.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/Burp-1.png" alt="HTTP History.">
</div>

Bueno, vemos varias cosas, pero las que me llaman la atención son **/dirRead.php** y **/fileRead.php**. Porque primero, dirRead, para mí hace referencia a directory Read, es decir, para leer el directorio, mientras que fileRead justamente hace referencia a leer archivo. Pero bueno, vamos a enviar las dos peticiones al repeater y vamos a renombrarlas de la misma manera para diferenciarlas.

# Análisis de Vulnerabilidades ​🔬​ [#](#analisis-de-vulnerabilidades) {#analisis-de-vulnerabilidades}
Una vez que tenemos ambas peticiones en el Repeater y podemos manipularlas, vamos a centrarnos primeramente en la petición <span style="color:blue"> /dirRead.php </span>

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/Burp-2.png" alt="Petición /dirRead.php">
</div>

JAJAJA, vemos algo lindo que es la variable **path**, pero también algo curioso. Si vemos la respuesta, notamos que lista el directorio actual, el que contiene las dos listas apenas entras a la aplicación web, y también vemos que esos puntos de ahí representan algo.

Para aquellos que no lo sepan, en Linux, el primer punto "**.**" representa el directorio actual de trabajo, mientras que los dos puntos "**..**" representan el directorio anterior del directorio actual de trabajo.

Por lo tanto, algo me dice que está aconteciendo un Directory Listing, así que vamos a comprobarlo.

## Directory Listing via Filter Bypass 🗂️​🛡️​ [#](#directory-listing-via-filter-bypass) {#directory-listing-via-filter-bypass}
Vamos a ir hacia atrás un directorio. Para eso, vamos a hacer **path traversal** y, si en la respuesta nos muestra los directorios del directorio al que nos dirigimos, es porque está ocurriendo un directory listing.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/filter-directory-listing.png" alt="Comprobando directory listing.">
</div>

Vemos que no pasa nada, vamos a corroborar si se está aplicando un filtro. Para eso, agregamos lo siguiente:

> ....//

Esto hará que, en caso de aplicar un **filtro** de **primera pasada** y **verificar** que **haya ../**, **elimine el primer** ../ y quede el **siguiente ../**

Si no entendiste nada no pasa nada, te dejo esta info de la biblia del hacking <a href="https://book.hacktricks.xyz/pentesting-web/file-inclusion" style="color:green"><strong>**Hacktricks: File Inclusion/Path traversal**</strong></a>

Bien si lo aplicamos y le damos send vemos que funciona, es decir se aplica el filtro y a la vez funciona el path traversal y en la respuesta se listan los directorios.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/directory-listing.png" alt="Confirmación de directory listing.">
</div>

Bueno, sabemos que ocurre un directory listing. Como vimos anteriormente, se listó el contenido del directorio anterior. Por ende, el componente **http://10.129.229.141/dirRead.php** es vulnerable a directory listing.

## Local File Inlcusion (LFI) via Filter Bypass 📑​🛡️​ [#](#local-file-inclusion-lfi-via-filter-bypass) {#local-file-inclusion-lfi-via-filter-bypass}
Ahora que sabemos que la página es vulnerable a directory listing, vamos a por la otra petición <span style="color:red"> /fileRead.php </span>

Jujuju vemos lo mismo pero en vez de la variable path vemos la variable **file**. Lo curioso y, por sobre todo importante ocurre en la respuesta, se ve el contenido de la list1 y lo que me lleva a sospechar que está ocurriendo un Local File Inlcusion (LFI). 

Sí o sí vamos a ver si no está sanitizado como el directory listing anterior.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/Burp-3.png" alt="Petición /fileRead.php">
</div>

Para eso, vamos a intentar obtener la lectura del archivo **/etc/passwd** para verificar si hay un LFI.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/filter-lfi.png" alt="Filtro detectado.">
</div>

No, tampoco funciona, se debe aplicar otro filtro. Probemos con el mismo bypass que utilizamos para el directory listing, a ver si funciona.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/lfi-etc-passwd.png" alt="Bypass del filtro.">
</div>

Vemos que funcionó correctamente y obtuvimos la lectura del archivo **/etc/passwd**, pero se ve un poco desordenado.

Vamos a separar mejor el salto de línea.

```ruby
root:x:0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
operator:x:11:0:operator:/root:/bin/sh
man:x:13:15:man:/usr/man:/sbin/nologin
postmaster:x:14:12:postmaster:/var/spool/mail:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
at:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin
squid:x:31:31:Squid:/var/cache/squid:/sbin/nologin
xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
postgres:x:70:70::/var/lib/postgresql:/bin/sh
cyrus:x:85:12::/usr/cyrus:/sbin/nologin
vpopmail:x:89:89::/var/vpopmail:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/home/nobody:/bin/sh
nginx:x:100:101:nginx:/var/lib/nginx:/sbin/nologin
```

Vemos que hay dos usuarios: el usuario <span style="color:orange"> operator </span>, que reside en la ruta personal **/root**, y el usuario <span style="color:violet"> nobody </span>, que reside en la ruta personal **/home/nobody**, quedemosno con este ultimo usuario y recordemos su ruta.

> Bien, esto me hace pensar que debemos jugar con el directory listing para listar los directorios clave y el LFI para leer los archivos en esos directorios.

# Explotación 💣​ [#](#explotacion) {#explotacion}
Ok, lo siguiente que haremos es probar si en el parámetro /dirRead.php lista el directorio raíz para **corroborar por última vez** si tenemos la posibilidad de listar directorios **desde** la **raíz**.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/directory-listing-raiz.png" alt="Listando directorios desde la raiz.">
</div>

Bueno, funcionó otra vez. Veo la estructura de los directorios de Linux, pero hay un directorio llamado **.dockerenv** que suena interesante. Pero lo que haremos ahora será listar el contenido del directorio **/home** donde se encuentra el usuario **nobody** y si hay algún archivo interesante, lo miramos con el LFI.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/directory-listing-home.png" alt="Listando directorio home.">
</div>

## SSH Private key 🔑 [#](#ssh-private-key) {#ssh-private-key}
Bueno vemos que si esta el usuario nobody, vamos a ver que contiene adentro.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/directory-listing-home-nobody.png" alt="Listando del directorio /home/nobody">
</div>

Bien vemos varias cosas, por un lado el archivo **user.txt** y por otro lado un archivo oculto **.ssh** , pero vamos por partes, primero miramos la flag , despues el arhcivo .ssh

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/home-nobody-user-txt.png" alt="Flag de user no visible.">
</div>

No podemos ver la flag del usuario, pero probemos ahora con el directorio oculto **.ssh**. Vamos a listar su contenido para ver qué contiene.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/home-nobody-ssh.png" alt="Listando directorio .ssh">
</div>

Contiene tres archivos, pero me parece extraño el archivo **.monitor**. Vamos a ver qué contiene.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/lfi-monitor.png" alt="Clave privada.">
</div>

Oaaaa, lindo eh! , tenemos una llave privada de SSH, pero nuevamente se ve desordenado.

Vamos a separar mejor el salto de línea.

```bash
-----BEGIN RSA PRIVATE KEY----- MIIEogIBAAKCAQEAs7sytDE++NHaWB9e+NN3V5t1DP1TYHc+4o8D362l5Nwf6Cpl mR4JH6n4Nccdm1ZU+qB77li8ZOvymBtIEY4Fm07X4Pqt4zeNBfqKWkOcyV1TLW6f 87s0FZBhYAizGrNNeLLhB1IZIjpDVJUbSXG6s2cxAle14cj+pnEiRTsyMiq1nJCS dGCc/gNpW/AANIN4vW9KslLqiAEDJfchY55sCJ5162Y9+I1xzqF8e9b12wVXirvN o8PLGnFJVw6SHhmPJsue9vjAIeH+n+5Xkbc8/6pceowqs9ujRkNzH9T1lJq4Fx1V vi93Daq3bZ3dhIIWaWafmqzg+jSThSWOIwR73wIDAQABAoIBADHwl/wdmuPEW6kU vmzhRU3gcjuzwBET0TNejbL/KxNWXr9B2I0dHWfg8Ijw1Lcu29nv8b+ehGp+bR/6 pKHMFp66350xylNSQishHIRMOSpydgQvst4kbCp5vbTTdgC7RZF+EqzYEQfDrKW5 8KUNptTmnWWLPYyJLsjMsrsN4bqyT3vrkTykJ9iGU2RrKGxrndCAC9exgruevj3q 1h+7o8kGEpmKnEOgUgEJrN69hxYHfbeJ0Wlll8Wort9yummox/05qoOBL4kQxUM7 VxI2Ywu46+QTzTMeOKJoyLCGLyxDkg5ONdfDPBW3w8O6UlVfkv467M3ZB5ye8GeS dVa3yLECgYEA7jk51MvUGSIFF6GkXsNb/w2cZGe9TiXBWUqWEEig0bmQQVx2ZWWO v0og0X/iROXAcp6Z9WGpIc6FhVgJd/4bNlTR+A/lWQwFt1b6l03xdsyaIyIWi9xr xsb2sLNWP56A/5TWTpOkfDbGCQrqHvukWSHlYFOzgQa0ZtMnV71ykH0CgYEAwSSY qFfdAWrvVZjp26Yf/jnZavLCAC5hmho7eX5isCVcX86MHqpEYAFCecZN2dFFoPqI yzHzgb9N6Z01YUEKqrknO3tA6JYJ9ojaMF8GZWvUtPzN41ksnD4MwETBEd4bUaH1 /pAcw/+/oYsh4BwkKnVHkNw36c+WmNoaX1FWqIsCgYBYw/IMnLa3drm3CIAa32iU LRotP4qGaAMXpncsMiPage6CrFVhiuoZ1SFNbv189q8zBm4PxQgklLOj8B33HDQ/ lnN2n1WyTIyEuGA/qMdkoPB+TuFf1A5EzzZ0uR5WLlWa5nbEaLdNoYtBK1P5n4Kp w7uYnRex6DGobt2mD+10cQKBgGVQlyune20k9QsHvZTU3e9z1RL+6LlDmztFC3G9 1HLmBkDTjjj/xAJAZuiOF4Rs/INnKJ6+QygKfApRxxCPF9NacLQJAZGAMxW50AqT rj1BhUCzZCUgQABtpC6vYj/HLLlzpiC05AIEhDdvToPK/0WuY64fds0VccAYmMDr X/PlAoGAS6UhbCm5TWZhtL/hdprOfar3QkXwZ5xvaykB90XgIps5CwUGCCsvwQf2 DvVny8gKbM/OenwHnTlwRTEj5qdeAM40oj/mwCDc6kpV1lJXrW2R5mCH9zgbNFla W0iKCBUAm5xZgU/YskMsCBMNmA8A5ndRWGFEFE+VGDVPaRie0ro= 
-----END RSA PRIVATE KEY-----
```

Por favor, nunca compartan su clave privada de SSH, la clave privada tiene que ser conocida solamente por ustedes.

## Conexion a la Máquina via SSH 🖥️🔒 [#](#conexion-a-la-maquina-via-ssh) {#conexion-a-la-maquina-via-ssh}
Ya que el puerto 22 está abierto, utilizaremos esta clave privada para conectarnos por **SSH** como usuario **nobody**.

Para utilizar esta clave privada, crearé un archivo en mi máquina atacante y lo llamaré **key_private**. Le daré permisos para que el propietario tenga permisos de lectura y escritura (valor **"6"**), mientras que el grupo al que pertenece el archivo, así como cualquier otro usuario en el sistema, no tendrán ningún permiso sobre el archivo (valor **"0"**). Por último, nos conectamos por SSH y ya tenemos acceso a la primera flag, la flag del usuario.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/flag-user-censured.png" alt="Flag de user.">
</div>

Si miramos las conexiones activas de la red y los puertos en escucha en el sistema con el siguiente comando "**netstat -lant**"

| **Parámetro** | **Descripción** |
|:-------------:|:----------------:|
| **netstat** | Este es el comando que se utiliza para mostrar estadísticas de la red. |
| **-l** | Indica que solo se deben mostrar las conexiones que están en modo "escucha" (listen), es decir, los servicios que están esperando conexiones entrantes. |
| **-a** | Muestra todas las conexiones y puertos, tanto los que están en escucha como los que están en uso. |
| **-n** | Muestra las direcciones IP y los números de puerto en formato numérico en lugar de intentar resolverlos a nombres de host o servicios. |
| **-t** | Esta opción muestra solo las conexiones TCP. |

Vemos lo siguiente:

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/netstat-lant.png" alt="Conexiones activas de la red y los puertos en escucha.">
</div>

JJAJAJ gente, miren, no nos conectamos por SSH al puerto 22, sino al puerto 8888. Para sacarnos la duda, miremos el archivo de **configuración** del servicio SSH que es este **/etc/ssh/sshd_config** 

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/cat-etc-ssh-sshd_config.png" alt="Archivo de configuración del servicio ssh.">
</div>

## Pivoting: nobody ---> monitor (Escapando del Contenedor) 🏃‍♂️🖥️ [#](#pivoting-nobody-monitor-escapando-del-contenedo) {#pivoting-nobody-monitor-escapando-del-contenedo}
Listo, confirmamos que el servicio SSH está escuchando en el puerto 8888. Entonces, supongo que hay que migrar y salir de este usuario para conectarnos por SSH. Cómo lo hacemos?

Se acuerdan de este directorio?

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/ssh-ls-la.png" alt="Directorio de clave publica.">
</div>

Bueno, nunca miramos qué hay dentro, pero supongo que habrá una clave pública.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/clave-publica-ssh.png" alt="Clave pública ssh.">
</div>

Es una clave pública para el usuario monitor, pero cómo es posible?, si habíamos visto dos usuarios, que son: **operator** y **nobody**.

A lo mejor tenemos que conectarnos a por ssh al puerto 22, que debe estar escuchando localmente con el usuario monitor, pero como?, tal vez reutilzando la clave privada.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/ssh-conect-monitor.png" alt="Acceso al usuario monitor.">
</div>

Bueno, nos dejó. Nos conectamos con éxito y nos da una linda bienvenida, pero hay un problema y es que tenemos una restricted bash (rbash), o sea, una bash restringida y no tenemos capacidad de ejecutar ningún comando.

Si queremos ejecutar el comando whoami, no nos dejará.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/rbash.png" alt="Restricted Bash.">
</div>

## Bypass Restricted Bash 🚫🔓 [#](#bypass-restricted-bash) {#bypass-restricted-bash}
Bueno, hay dos formas de bypassear esto en SSH. La primera es aplicando este comando `ssh -i .monitor monitor@localhost bash`, pero no es recomendable ya que no asigna una pseudo terminal. La sesión de Bash no tiene todas las funcionalidades interactivas, por lo tanto, tendremos que estabilizar la shell. La segunda opción es `ssh -i .monitor monitor@localhost -t bash`, donde no tendremos que estabilizar la shell, porque con el comando **-t** se asigna una terminal plenamente interactiva y útil.

Por lo tanto, usaremos la segunda opción.

Bueno, ejecutamos el comando pero tenemos otro problema, y es que no podemos ejecutar los comandos del sistema como `whoami` o `id`

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/bypass-rbash.png" alt="Bypass de la restricted Bash.">
</div>

Esto sucede debido a que el sistema no encuentra un binario en todas las rutas del PATH que esté asociado a ese comando. Para solucionar esto, debemos modificar el valor de la variable PATH.

Usaremos nuestra variable de entorno de nuestra máquina. Para eso, hacemos un `echo $PATH` y lo exportamos en la máquina víctima.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/export-path.png" alt="Exportando la variable $PATH.">
</div>

Listo, ya funcionan los comandos.

# Escalada de Privilegios 🎢 [#](#escalada-de-privilegios) {#escalada-de-privilegios}
## Privesc Mediante el Abuso Capabilities 🔄⚙️​ [#](#privesc-mediante-el-abuso-capabilities) {#privesc-mediante-el-abuso-capabilities}
Para escalar privilegios, enumeré capabilities, donde encontré dos cosas interesantes:

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/enumeracion-capabilities.png" alt="Enumerando capabilities.">
</div>

Vemos las siguientes capabilities:

La más interesante es esta: **/usr/bin/tac**, donde el binario **tac** tiene la cacapabilitie **cap_dac_read_search**. Para aquellos que no lo sepan, esta capacidad permite **leer archivos**. Con respecto a **tac**, es **similar** a **cat** pero al **revés**, lo que significa que podemos leer archivos con tac, pero la **salida mostrará** la **última** **línea** del **archivo original** y luego la **penúltima línea**, y **así sucesivamente**.

Dejo un ejemplo aquí:

```ruby
monitor@waldo:~$ cat test.txt 
Hola
esto
es
un
ejemplo 
de 
tac
monitor@waldo:~$ tac test.txt 
tac 
de 
ejemplo
un
es
esto
Hola
```

Utilizaremos tac para leer la flag de root.

<div align="center">
  <img src="/assets/images/HTB/writeup-waldo/flag-root-censured.png" alt="Flag de root.">
</div>

En esta máquina no se puede rootear, ya que está configurada de tal manera que no se pueda reutilizar la clave privada de SSH para conectarse vía SSH como usuario root.

# Conclusión Final 💬​ [#](#conclusion-final) {#conclusion-final}
Bueno, mi opinión es la siguiente: Buena máquina al principio, estuvo bueno jugar con Burp y las dos peticiones para encontrar el directorio mediante Directory Listing y leerlo vía el Local File Inclusion y el filtro que se le aplicó fue bastante fácil. Luego, al momento de bypassear la rbash, me pareció muy buena, nunca había realizado un bypass de ese estilo. Por último, para leer la flag mediante esa capability, me pareció muy divertida, pero lamentablemente no se puede hacer root en la máquina.

Entonces, es recomendable por las vulnerabilidades que se tocan, que son muy básicas, y también por el privesc que es mediante el abuso de capabilities.

# Apéndice I Links de Referencia 📎​ [#](#apendice-i-links-de-referencia) {#apendice-i-links-de-referencia}
## Herramientas Utilizadas en la Auditoria 🛠️​ [#](#herramientas-utilizadas-en-la-auditoria) {#herramientas-utilizadas-en-la-auditoria}
* [Nmap:](https://nmap.org) [https://nmap.org](https://nmap.org) - [https://www.kali.org/tools/nmap](https://www.kali.org/tools/nmap) → Uso de nmap para el escaneo de puertos.
* [BurpSuite Community Edition:](https://portswigger.net/burp/communitydownload) [https://portswigger.net/burp/communitydownload](https://portswigger.net/burp/communitydownload) → Uso de BurpSuite para interceptar peticiones.

## Documentación ​📰​​ [#](#documentacion) {#documentacion}
* [Hacktricks: File Inclusion/Path traversal](https://book.hacktricks.xyz/pentesting-web/file-inclusion) [https://book.hacktricks.xyz/pentesting-web/file-inclusion](https://book.hacktricks.xyz/pentesting-web/file-inclusion) 
* [Técnicas para escapar de shells restringidas (restricted shells bypassing)](https://www.hackplayers.com/2018/05/tecnicas-para-escapar-de-restricted--shells.html) : [https://www.hackplayers.com/2018/05/tecnicas-para-escapar-de-restricted--shells.html](https://www.hackplayers.com/2018/05/tecnicas-para-escapar-de-restricted--shells.html)
