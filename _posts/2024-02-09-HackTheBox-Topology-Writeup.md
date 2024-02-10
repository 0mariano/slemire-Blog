---
layout: single
title: Topology - Hack The Box
excerpt: "Topology se trata de una máquina basada en el Sistema Operativo Linux, donde a través de Local File Inclusion (LFI) via LaTeX Injection se obtiene un archivo que contiene credenciales que luego se utilizan para entablar conexión por SSH. Finalmente, para la escalada de privilegios, se debe crear un archivo con extensión .plt para el programa gnuplot, con el fin de convertir la BASH en permisos SUID."
date: 2024-02-09
classes: wide
header:
  teaser: /assets/images/HTB/writeup-topology/topology.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Easy
  - CFTs
tags:
  - Pentesting
  - Linux
  - LaTeX
  - LaTeX Injection
  - Local FIle Inclusion
  - LFI
  - SSH
---

![Topology](assets/images/HTB/writeup-topology/topology.png)

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Topology.pdf)

---

# **Índice**

1. [Introducción](#introduccion)
  * [Alcance](#alcance)
  * [Metodologia Aplicada](#metodologia-aplicada)
2. [Reconocimiento - Enumeración](#reconocimiento-enumeracion)
  * [Uso de la Herramienta Nmap](#uso-de-la-herramienta-nmap)
  * [Aplicación Web](#aplicacion-web)
3. [ Análisis de Vulnerabilidades](#analisis-de-vulnerabilidades)
  * [Local File Inlcusion (LFI) via LaTeX Injection](#local-file-inlcusion-lfi-via-latex-injection)
4. [Explotación](#explotacion)
  * [Enumeración de Archivos del Sistema](#enumeracion-de-archivos-del-sistema)
  * [Uso de la Herramienta John the Ripper](#uso-de-la-herramienta-john-the-ripper)
  * [Acceso al Sistema via SSH](#acceso-al-sistema-via-ssh)
  * [Listamiento de Directorios Interesantes](#listamiento-de-irectorios-interesantes)
5. [Escalada de Privilegios](#escalada-de-privilegios)
  * [Uso de la Herramienta pspy](#uso-de-la-herramienta-pspy)
  * [Creación del Exploit](#creacion-del-exploit) 
6. [Conclusión Final](#conclusion-final)
7. [Apéndice I Links de Referencia](#apendice-i-links-de-referencia)
  * [Herramientas Utilizadas en la Auditoria](#herramientas-utilizadas-en-la-auditoria)
  * [Documentación](#documentacion) 

---

# Introducción 📄 [#](#introduccion) {#introduccion}
En el presente Write Up explicare los pasos para resolver la máquina <a href="https://app.hackthebox.com/machines/Topology" style="color:blue"><strong>**Topology**</strong></a> de la plataforma [HackTheBox](https://hackthebox.com).
Topology se trata de una máquina basada en el Sistema Operativo Linux, donde a través de Local File Inclusion (LFI) via LaTeX Injection se obtiene un archivo que contiene credenciales que luego se utilizan para entablar conexión por SSH. Finalmente, para la escalada de privilegios, se debe crear un archivo con extensión .plt para el programa gnuplot, con el fin de convertir la BASH en permisos SUID.

## Alcance 🎯 [#](#alcance) {#alcance}
El alcance de esta máquina fue definida como la siguiente.

| **Servidor Web / Direcciónes IPs / Hosts / URLs** |           **Descripción**           | **Subdominios** |
|:-------------------------------------------------:|:-----------------------------------:|:---------------:|
|                   10.129.16.121                   | Dirección IP de la máquina Topology |      Todos      |

## Metodologia Aplicada 📦​​📚​​ [#](#metodologia-aplicada) {#metodologia-aplicada}
* Enfoque de prueba: En el proceso de pruebas de seguridad, se optó por un enfoque gray-box, lo que significó que se tenía un nivel de acceso parcial a la infraestructura y el sistema objetivo.
* Las etapas aplicadas para esta auditoria fueron las siguientes:

![](/assets/images/HTB/writeup-topology/Etapas-pentest.png)

# Reconocimiento - Enumeración 🔍 [#](#reconocimiento-enumeracion) {#reconocimiento-enumeracion}
## Uso de la Herramienta Nmap 👁️ [#](#uso-de-la-herramienta-nmap) {#uso-de-la-herramienta-nmap}
Primeramente realizamos un escaneo con ayuda de la herramienta nmap en búsqueda de puertos abiertos.

```bash
# Primer escaneo.
--------------------------------------------------------------
nmap -p- --open -sV --min-rate 5000 10.129.16.121
```

|    **Parámetro**    |                                   **Descripción**                                   |
|:-------------------:|:-----------------------------------------------------------------------------------:|
|       **-p-**       |                              Escanea los 65535 puertos.                             |
|      **\--open**     |                          Muestra solo los puertos abiertos.                         |
|       **-sV**       | Determina la versiones de los servicios que se ejecutan en los puertos encontrados. |
| **\--min-rate 5000** |   Establece la velocidad mínima de envío de paquetes a 5000 paquetes por segundo.   |

El resultado que nos arrojó este primer escaneo fue que la máquina tiene el puerto **22** que pertenece al servicio *SSH* y el puerto **80** que pertenece al protocolo *HTTP*.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/nmap_1.png" alt="Primer escaneo.">
</div>

Se procedió a realizar otro escaneo con los scripts default de nmap, también especificando la versión nuevamente.

```bash
# Segundo escaneo.
--------------------------------------------------------------
nmap -sC -sV -p22,80 10.129.16.121
```

| **Parámetro** |                                   **Descripción**                                   |
|:-------------:|:-----------------------------------------------------------------------------------:|
|    **-sC**    |                   Realiza un escaneo con los scripts por defecto.                   |
|    **-sV**    | Determina la versiones de los servicios que se ejecutan en los puertos encontrados. |
|     **-p**    |                      Especifica los puertos que se escanearán.                      |

Lo único interesante que obtenemos es el título de la página web **Miskatonic University | Topology Group**.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/nmap_2.png" alt="Resultado del segundo escaneo.">
</div>

## Aplicación Web 🌐​ [#](#aplicacion-web) {#aplicacion-web}
Luego del segundo escaneo, se ingresó a la aplicación web, donde en el código fuente se encontró un subdominio **http://latex.topology.htb/equation.php**

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/código-fuente.png" alt="Código fuente.">
</div>

Antes de ingresar al subdominio, se agregó el mismo al archivo **/etc/hosts**

```bash
# Agregando subdominio al archivo /etc/hosts.
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.16.121 latex.topology.htb
```

# Análisis de Vulnerabilidades ​🔬​ [#](#analisis-de-vulnerabilidades) {#analisis-de-vulnerabilidades}
Al ingresar al subdominio, vemos que cosiste en una generador de ecuaciones mediante sintaxis en LaTeX.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/Latex-equation.png" alt="Aplicación web.">
</div>

Realizamos un **PoC**, para ver más en detalle la funcionalidad de la aplicación web.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/PoC_1.png" alt="Realizando PoC.">
</div>

Una vez que enviamos el comando de LaTeX para generar la ecuación, nos envia a una ruta con la ecuación generada en formaato de imagen.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/PoC_resultado.png" alt="Ecuación generada.">
</div>

Muy interesante el funcionamiento de la aplicación y de como se genera la ecuación.

## Local File Inlcusion (LFI) via LaTeX Injection 📄💉​ [#](#local-file-inlcusion-lfi-via-latex-injection) {#local-file-inlcusion-lfi-via-latex-injection}
Al entender como funciona la aplicación, se me ocurrió realizar una prueba que consiste en incluir el archivo **/etc/passwd**, ingresando código de LaTeX arbitrario.

Para realizar esta prueba se utilizó en siguiente recurso [https://book.hacktricks.xyz/v/es/pentesting-web/formula-csv-doc-latex-ghostscript-injection](https://book.hacktricks.xyz/v/es/pentesting-web/formula-csv-doc-latex-ghostscript-injection) de [HackTricks](https://book.hacktricks.xyz)

Realicé una prueba para incluir el archivo **/etc/passwd** con el siguiente comando.

```latex
# Comando utilizado.
--------------------------------------------------------------
$input/etc/passwd$
```

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/Latex-Injection.png" alt="Comando ingresado.">
</div>

Como resultado la apliación detecta que se están ingresando comandos arbitrarios.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/Latex-Injection-Resultado.png" alt="Imagen generada.">
</div>

Después de varios intentos, el comando que me funcionó para obtener lectura del archivo **/etc/passwd**, fue el siguiente:

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/Latex-Injection_2.png" alt="Comando útil.">
</div>

Como resultado genera la imagen del archivo **/etc/passwd**

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/Latex-Injection_2_Resultado.png" alt="Archivo /etc/passwd">
</div>

## Explotación 💣​ [#](#explotacion) {#explotacion}
## Enumeración de Archivos del Sistema 📌​ [#](#enumeracion-de-archivos-del-sistema) {#enumeracion-de-archivos-del-sistema}
Al obtener lectura del archivo, se confirma que la aplicación web es vulnerable a **Local File Inclusion** via **LaTeX Injection**.

Utilizaremos esta vulnerabilidad para conseguir posibles datos que nos interesen o sean útiles. 

Como la app web tiene un servidor web apache, procedemos a leer el archivo de configuración predeterinado del servidor web, que se encuentra en esta ruta **/etc/apache2/sites-available/000-default.conf**

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/comando-archivo-config.png" alt="Comando para obtener lectura del archivo de configuración.">
</div>

Obtenemos el resultado, donde se aprecia la ruta de la landing page de la universidad y las demas rutas de los aplicativos.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/archivo-cofig_1.png" alt="Aplicativos con sus respectivas rutas.">
</div>

Además se encuentran otros aplicativos con sus respectivas rutas.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/otro-aplicativo.png" alt="Nuevos aplicativos con sus respectivas rutas.">
</div>

Para ingresar a los subdominios encontrados, debemos nuevamente agregarlos al archivo **/etc/hosts**

```bash
# Agregando subdominios al archivo /etc/hosts.
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.16.121 latex.topology.htb dev.topology.htb stats.topology.htb
```

Lo único interesante es el subdominio **dev.topology.htb**, que tiene un formulario de login, pero no podemos ingresar por falta de credenciales.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/dev-form-login.png" alt="Fomrulario de inicio de sesión.">
</div>

Nuevamente utilizaremos el LFI para acceder al archivo **.htpasswd**, donde se supone que se guardan las credenciales de autenticación del servidor HTTP Apache.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/lfi-htpasswd.png" alt="LFI para el archivo .htpasswd">
</div>

Como resultado obtenemos un usuario y una contraseña cifrada, aparentemente con el **algoritmo** de **hashing** que usa **Apache** por defecto, que es **APR1**.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/user-passwd.png" alt="Usuario y contraseña cifrada.">
</div>

## Uso de la Herramienta John the Ripper 🕵️‍♂️​ [#](#uso-de-la-herramienta-john-the-ripper) {#uso-de-la-herramienta-john-the-ripper}
Con el uso de la herramienta [John the Ripper](https://www.kali.org/tools/john) desciframos la contraseña.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/passwd-deshasheda.png" alt="Contraseña descifrada.">
</div>

La contraseña obtenida es **calculus20** y la utilizaremos en el formulario de login junto al usuario **vdaisley** que obtuvimos anteriormente.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/creds-form-login.png" alt="Iniciando sesión.">
</div>

Al entrar no encontramos nada interesante, solo un software desarrollado por el personal de la **universidad Miskatonic**.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/dev-pag.png" alt="Software desarrollado por el personal de la universidad.">
</div>

## Acceso al Sistema via SSH 🖥️​ [#](#acceso-al-sistema-via-ssh) {#acceso-al-sistema-via-ssh}
Al haber escaneado los puertos anteriormente y obtener información de que el puerto 22 que corresponde al servicio SSH esta abierto, procedemos a conectarnos por dicho servicio, utilizando las credenciales obtenidas.

Al conectarnos exitosamente, podemos leer la <span style="color:blue"> **flag** </span> del **user**. 

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/user-flag.png" alt="Conexión por SSH.">
</div>

## Listamiento de Directorios Interesantes 🤔​ [#](#listamiento-de-irectorios-interesantes) {#listamiento-de-irectorios-interesantes}
Una vez dentro, vemos en el directorio **opt** que se encuentra un directorio llamado **gnuplot**, el cual tiene permisos de **escritura** y **ejecución**, y cuyo propietario es **root**.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/gnuplot.png" alt="Directorio opt.">
</div>

Investigando un poco encontre que **gnuplot** es un programa de interfaz de línea de comandos para generar gráficas de dos y tres dimensiones de funciones, datos y ajustes de datos.

# Escalada de Privilegios 👨‍💻​ [#](#escalada-de-privilegios) {#escalada-de-privilegios}
## Uso de la Herramienta pspy ⚙️​ [#](#uso-de-la-herramienta-pspy) {#uso-de-la-herramienta-pspy}
Para seguir enumernado la máquina victima utilizaré [pspy](https://github.com/DominicBreuker/pspy), que es una herramienta de monitoreo de procesos para sistemas Linux. Esta herramienta permitira enumerar procesos de la máquina victima.

Antes debo saber cuál es la arquitectura y la cantidad de bits del sistema Linux de la máquina víctima, para poder descargar el ejecutable de pspy para la arquitectura correspondiente.

Para eso ejecuto el comando **uname -m**, en el cual obtengo que la arquitectura de la máquina victima es de 64 bits.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/64-bits.png" alt="Arquitectura de 64 Bits.">
</div>

Una vez teniendo este dato me descargo el ejecutable de pspy y creo en el servidor python en el puerto 80, para luego desde la máquina victima realizar una petición wget y pasarme el archivo de la herramienta.

Realizamos una petición wget en la máquina victima.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/wget.png" alt="Petición wget.">
</div>

Le asignamos permisos de ejecución al ejecutable de pspy.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/chmod.png" alt="Asignando permisos de ejecución.">
</div>
 
Una vez asignados los permisos de ejecución, procedemos a ejecutarlo y observamos que el usuario con **UID=0**, o sea el usuario **root**, ejecuta el comando **find** sobre el directorio **/opt/gnuplot**, donde busca todos los archivos con extensión **.plt** (que es la extensión que corresponde al programa gnuplot) y luego los ejecuta.

Luego ejecuta una serie de scripts y por ultimo ejecuta un archivo **networkplot.plt**

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/resultado-pspy.png" alt="Procesos.">
</div>

Nos damos cuenta en el directorio **/opt/gnuplot** tenemos permisos de escritura pero no de listamiento.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/permisos.png" alt="Permisos.">
</div>

## Creación del Exploit 👾​ [#](#creacion-del-exploit) {#creacion-del-exploit}

Lo que se me ocurre es crear un archivo asignando permisos **SUID** para convertir la **BASH** del sistema, de modo que luego root ejecute el script habilitando el **Bit SUID** y enlace una BASH con altos privilegios.

```bash
# Exploit.
--------------------------------------------------------------
nano root.plt
          
system "chmod u+s /bin/bash"
```

Una vez creado el archivo, ejecutamos de vuelta pspy para saber cuando root ejecuto el archivo.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/pspy-script.png" alt="Archivo ejecutado.">
</div>

Para confirmar hacemos un **ls -la** de la bash y vemos que tiene el Bit SUID activado.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/ls-l-bash.png" alt="Bit SUID activado.">
</div>

Simplemente ahora hacemos **bash -p** y conseguimos escalar privilegios y obtener la **flag** de **root**.

<div align="center">
  <img src="/assets/images/HTB/writeup-topology/whoami-flag.png" alt="Flag.">
</div>

# Conclusión Final 💬​ [#](#conclusion-final) {#conclusion-final}
Esta máquina resultó muy entretenida, ideal para aquellos que recién comienzan en el pentesting resolviendo máquinas. La verdad es que fue bastante sencilla, en mi caso nunca había explorado ni explotado un Local File Inclusion (LFI) a través LaTeX Injection. Si no fuera por eso, la habría terminado antes. Después de eso, la parte de explotación y priv-esc no me dio problemas.

# Apéndice I Links de Referencia 📎​ [#](#apendice-i-links-de-referencia) {#apendice-i-links-de-referencia}
## Herramientas Utilizadas en la Auditoria 🛠️​ [#](#herramientas-utilizadas-en-la-auditoria) {#herramientas-utilizadas-en-la-auditoria}
* [Nmap:](https://nmap.org) - [https://nmap.org](https://nmap.org) - [https://www.kali.org/tools/nmap](https://www.kali.org/tools/nmap) ---> Uso de nmap para el escaneo de puertos.
* [John the Ripper:](https://www.openwall.com/john) - [https://www.openwall.com/john](https://www.openwall.com/john) - [https://www.kali.org/tools/john](https://www.kali.org/tools/john) - [https://github.com/openwall/john](https://github.com/openwall/john) ---> Uso de John the Ripper para descifrar contraseña.
* [pspy - unprivileged Linux process snooping:](https://github.com/DominicBreuker/pspy) - [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy) ---> Uso de pspy para monitorear procesos.

## Documentación ​📰​​ [#](#documentacion) {#documentacion}
* [HackTricks: LaTeX Injection](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection) - [https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection) 
* [Gnuplot Privilege Escalation:](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation) - [https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation)
