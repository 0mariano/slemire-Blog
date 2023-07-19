---
layout: single
title: TwoMillion - Hack The Box
excerpt: "¡El especial dos millones! La plataforma HackTheBox presenta una máquina Linux de nivel fácil con el antiguo diseño de la plataforma, incluyendo el código de invitación.
Para conseguir acceso a la máquina, primero debemos completar el HTB Challenge al registrarnos. Luego de completarlo y crear la cuenta, la usaremos para enumerar la API y obtener permisos como admin, además de realizar una CMD injection en la generación de la VPN. Posteriormente, aprovecharemos una Reverse Shell mediante la enumeración de archivos de variables de entorno (el sistema tiene un archivo que suele contener credenciales de bases de datos o información de conexión a servicios externos). Por último, para obtener acceso de root, debemos explotar la vulnerabilidad en el kernel CVE-2023-0386."
date: 2023-07-19
classes: wide
header:
  teaser: /assets/images/HTB/writeup-twomillion/TwoMillion.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Easy
  - CFTs
  - CVE
tags:
  - Pentesting
  - Linux
  - CVE
  - Api
  - JavaScript
  - OverlayFS Vulnerability
---

![](/assets/images/HTB/writeup-twomillion/TwoMillion.png)

---

# **Índice**

1. [Introducción](#introduccion)
2. [Reconocimiento](#reconocimiento)
  * [Herramienta nmap](#herramienta-nmap)
3. [Enumeración](#enumeracion)
  * [Investigando el sitio web](#investigando-el-sitio-web)
  * [Api](#api)
4. [Explotación](#explotacion)
  * [Obteniendo acceso como admin](#obteniendo-acceso-como-admin)
  * [Shell como www-data](#shell-como-www-data)
  * [Escalada de privilegios](#escalada-de-privilegios)
5. [Agradecimiento y opinión](#agradecimiento-y-opinion)

---

# Introducción 📄 [#](#introduccion) {#introduccion}
El presente Write Up explica los pasos para resolver la máquina <span style="color:orange"> **TwoMillion** </span> de la plataforma [HackTheBox](https://hackthebox.com).
¡El especial dos millones! La plataforma HackTheBox presenta una máquina Linux de nivel fácil con el antiguo diseño de la plataforma, incluyendo el código de invitación.
Para conseguir acceso a la máquina, primero debemos completar el HTB Challenge al registrarnos. Luego de completarlo y crear la cuenta, la usaremos para enumerar la API y obtener permisos como admin, además de realizar una CMD injection en la generación de la VPN. Posteriormente, aprovecharemos una Reverse Shell mediante la enumeración de archivos de variables de entorno (el sistema tiene un archivo que suele contener credenciales de bases de datos o información de conexión a servicios externos). Por último, para obtener acceso de root, debemos explotar la vulnerabilidad en el kernel CVE-2023-0386.

# Reconocimiento 🔍 [#](#reconocimiento) {#reconocimiento}
## Herramienta nmap 👁️ [#](#herramienta-nmap) {#herramienta-nmap}
Lazamos la herramienta nmap para averiguar los puertos y servicios abiertos.

```bash
# Primer lanzamiento de la herramienta en nmap
--------------------------------------------------------------
nmap -p- --open -v 10.10.11.221 
```

![](/assets/images/HTB/writeup-twomillion/Nmap1.png)

Obtenemos dos puertos abiertos, el puerto <span style="color:blue"> 22 </span> que pertence al protocolo **ssh** y el puerto <span style="color:red"> 80 </span> que pertence al protocolo **http**.

Vamos a tirar nmap otra vez, pero ahora vamos a especificar la versión del servicio.

```bash
# Segundo lanzamiento de la herramienta en nmap
--------------------------------------------------------------
nmap -p 22,80 -sC -sV 10.10.11.221 -oN Information
```

![](/assets/images/HTB/writeup-twomillion/Nmap2.png)


Vemos que en el puerto 80 intenta redireccionar la conexión al dominio **2million.htb**, pero no tiene éxito.

# Enumeración 📌 [#](#enumeracion) {#enumeracion}
Vamos a tratar de entrar al dominio <span style="color:violet"> 2million.htb </span>, para eso hay que modificar el archivo de **/etc/hosts**

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el dominio que obtuvimos con el segundo escaneo
--------------------------------------------------------------
nano /etc/hosts

10.10.11.221 2million.htb
```

Ahora si queremos acceder al sitio web, podemos hacerlo.

## Investigando el sitio web 🕵️‍♂️ [#](#investigando-el-sitio-web) {#investigando-el-sitio-web}
Podemos ver la antigua plataforma de [HackTheBox](https://hackthebox.com).

![](/assets/images/HTB/writeup-twomillion/2million.htb.png)

Podemos ver el antiguo sitio de HackTheBox y los apartados de los mismo **abount, FAQ, join, commercial, labs, hall of fame ,login**.
Nos interesa loguearnos, pero tenemos un problema y es que no tenemos permiso para eso, tampoco no tenemos una cuenta, pero si vamos a <span style="color:green"> join </span> hay directorio <span style="color:pink"> /invite </span>, vamos a mirar por ahi.

![](/assets/images/HTB/writeup-twomillion/ir_a_join.png)

![](/assets/images/HTB/writeup-twomillion/-invite.png)

Claro como es la plataforma antigua se trata del CTF que teaniamos que hacer todos para poder registrarnos, mirando el codigo HTML encuentro este directorio **/js/inviteapi.min.js**

![](/assets/images/HTB/writeup-twomillion/-js-inviteapi-min-js.png)

Si miramos al directorio <span style="color:red"> 2million.htb/js/inviteapi.min.js </span>, encontramos algo interesante:

```bash
# Código ofuscado
--------------------------------------------------------------
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```
Es coódigo <span style="color:yellow"> JS </span> y esta ofuscado, asi que usare la herramienta  [beautifier.io](https://beautifier.io)

Obtenemos este código legible:

```bash
# Código legible
--------------------------------------------------------------
function verifyInviteCode(code) {
            var formData = {
                "code": code
            };
            
            $.ajax({
                type: "POST",
                dataType: "json",
                data: formData,
                url: '/api/v1/invite/verify',
                success: function(response) {
                    console.log(response);
                },
                error: function(response) {
                    console.log(response);
                }
            });
        }
        
        function makeInviteCode() {
            $.ajax({
                type: "POST",
                dataType: "json",
                url: '/api/v1/invite/how/to/generate',
                success: function(response) {
                    console.log(response);
                },
                error: function(response) {
                    console.log(response);
                }
            });
        }
```

Vemos una función **makeInviteCode**, si lo ejecutamos en la consola, obtenemos información cifrada con **ROT13**.

![](/assets/images/HTB/writeup-twomillion/makeinvitecodepng.png)

[rot13.com](https://rot13.com)

![](/assets/images/HTB/writeup-twomillion/decipher.png)

Dice que hagamos un **POST** a esa ruta **/api/v1/invite/generate**, para eso usamos curl.

```bash
# Resultado
--------------------------------------------------------------
  {
    "0": 200,
    "success": 1,
    "data": {
    "code": "UEJKWk4tVlFSVFAtS1dXSEwtOU1NTEE=",
    "format": "encoded"
            }
  }
```

Veo que esta en **base64**, para verlo mejor, lo deciframos y lo ingresamos en **/invite**

![](/assets/images/HTB/writeup-twomillion/insert-code.png)

Vemos que nos dirije a registrar un usuario.

![](/assets/images/HTB/writeup-twomillion/registercode.png) 

Registramos un usuario y accedemos a **/home** , donde solo tenemos acceso a Access, en url seria **/home/access**
    
En Access podemos descargar y regenerar VPN, tal y como lo hacemos siempre en HackTheBox. 
    
Lo mejor sera interceptar la petición con <span style="color:orange"> BurpSuite </span> de Connection Pack y Regenerate.
    
El <span style="color:red"> GET </span> de **Conection Pack** dirije a **/api/v1/user/vpn/generate**

![](/assets/images/HTB/writeup-twomillion/generate.png)

El <span style="color:red"> GET </span> de **Regenerate** dirije a **/api/v1/user/vpn/regenerate**

![](/assets/images/HTB/writeup-twomillion/regenerate.png)

## Api ⬜ [#](#api) {#api}
Podemos probar haciendo una petición a **/api** para ver que sucede.

![](/assets/images/HTB/writeup-twomillion/-api.png) 

Nos muestra la versión, <span style="color:pink"> v1 </span> y si le realizamos petición a **/api/v1**, obtenemos información de rutas y acciones de la api.

```bash
# Información sobre la API
--------------------------------------------------------------
{
        "v1":{ 
        "user": {
        "GET": {
                "/api/v1": "Route List",  
                "/api/v1/invite/how/to/generate": "Instructions on invite code generation", 
                "/api/v1/invite/generate": "Generate invite code",
                "/api/v1/invite/verify": "Verify invite code",
                "/api/v1/user/auth": "Check if user is authenticated",
                "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
                "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
                "/api/v1/user/vpn/download": "Download OVPN file"
                },

        "POST": {
                "/api/v1/user/register": "Register a new user",
                "/api/v1/user/login": "Login with existing user"
                }
                },
        "admin": {
        "GET": {
                "/api/v1/admin/auth": "Check if user is admin"
                },
        "POST": {
                "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
                },
        "PUT": {
                "/api/v1/admin/settings/update": "Update user settings"
                }
                }
                }
}
```
# Explotación 🔥 [#](#explotacion) {#explotacion}
## Obteniendo acceso como admin 👨‍💼 [#](#obteniendo-acceso-como-admin) {#obteniendo-acceso-como-admin}
Podemos averiguar si somos admin (spoiler no lo somos) haciendo un <span style="color:violet"> GET </span> a **/api/v1/admin/auth** , por lo tanto podemos jugar un poco cambiando la configuración haciendo un <span style="color:pink"> PUT </span> a **/api/v1/admin/settings/update**

Primero accedemos a la ruta y nos dice que el **Invalid content type**.

![](/assets/images/HTB/writeup-twomillion/put-update.png)

Al parecer para representar datos estructurados se nececita <span style="color:blue"> json </span>, colocamos **Content-Type: application/json**

![](/assets/images/HTB/writeup-twomillion/content-type.png)

Nos dice que indiquemos como parametro un email, usemos el que usamos para registrar nuestra cuenta.

Nos pide un nuevo parametro: **"is_admin"** ,  hay que indicar 0 o 1 si indicas otro parametro, va a decirte que indiques 0 o 1

![](/assets/images/HTB/writeup-twomillion/message-is_admin.png)

![](/assets/images/HTB/writeup-twomillion/is_admin_1.png)

Al parecer funciona.

![](/assets/images/HTB/writeup-twomillion/login-en-burpsuite.png)

Ahora verifiquemos si somos admin, haciendo <span style="color:orange"> GET </span> a **/api/v1/admin/auth**

![](/assets/images/HTB/writeup-twomillion/true-auth.png)

Nos reporta <span style="color:orange"> true </span>, por lo tanto somos admin.

Ahora probemos con un **POST** a **/api/v1/admin/vpn/generate**

![](/assets/images/HTB/writeup-twomillion/post-vpn.png)

Nos pide de vuelta el **username** para generar la VPN, tal y como lo hace HackTheBox.

![](/assets/images/HTB/writeup-twomillion/username-2twomillion.png)

## Shell como www-data ⌨️ [#](#shell-como-www-data) {#shell-como-www-data}

Probablemente este campo de username sea vulnerable, intentemos inyetando comandos, para eso hay que agregar una **;** para dividirlo con el usuario, ingresamos el comando **whoami**, despues hay que un **#** para comentar luego.

![](/assets/images/HTB/writeup-twomillion/inyect-code-malicious.png)

Vamos que funciona, ahora lo bueno es poder hacer una Reverse Shell.

```bash
# Nos ponemos en escucha por netcat desde nuestra terminal 
--------------------------------------------------------------
nc -nlvp 8002
```

![](/assets/images/HTB/writeup-twomillion/rshell.png)

Listo, antes de continuar hacemos un tratamiento a la **tty**.

```bash
# Tratamiento a la tty 
--------------------------------------------------------------
script /dev/null -c bash
Script started, output log file is '/dev/null'
^Z
[1]+  Stopped                 nc -nlvp 443
stty raw -echo;fg
              reset xterm
export TERM=xterm
export SHELL=bash
```

Si listamos detalladamente el contenido de un directorio, encontramos un archivo <span style="color:red"> .env </span>, que suele contener variables de entorno como claves de API, credenciales de base de datos, contraseñas, tokens de acceso, rutas de archivos.

![](/assets/images/HTB/writeup-twomillion/-env.png)

Entonces miremos por ahi.

```bash
# Visualizando en archivo .env
--------------------------------------------------------------
cat .env
```

![](/assets/images/HTB/writeup-twomillion/user-pass.png)

Vemos un user y una password, lo usaremos para conectarnos por **ssh**.

```bash
# Conexición por ssh usando usuario y password que encontramos en el archivo .env
--------------------------------------------------------------
shh admin@10.10.11.221

SuperDuperPass123
```

Listamos contenido del directorio y encontramos la **flag** de usuario.

![](/assets/images/HTB/writeup-twomillion/user-flag.png)

## Escalada de privilegios 👩‍💻 [#](#escalada-de-privilegios) {#escalada-de-privilegios}

Si buscamos directorios desde raiz, encontramos el directorio **/var**, miremos ahi.

![](/assets/images/HTB/writeup-twomillion/admin-var.png)

Si entramos vemos otro directorio interesante **/mail**.

![](/assets/images/HTB/writeup-twomillion/admin-email.png)

Vemos un correo que nos dice.

```text
# E-mail
--------------------------------------------------------------
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

En resumen nos dicen que si podemos actualizar el Sitema Operativo del servidor por hay nuevos CVE que tienen que ver con el Kernel de Linux, habla de un **CVE OverlayFS / FUSE**, investigando un poco,encontre información sobre:

-[Informacion de OverlayFS vulnerability DATADOG](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/)

-[CVE-2023-0386](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-0386)

No me pondre a explicar en que consiste la vuln pero si quieres profundizar el tema, te dejo de vuelta el enlace del **blog** de **DATADOG**, aqui que explica detalladamente la vuln [Informacion de OverlayFS vulnerability DATADOG](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/), pero dice que los sistemas de **Versión del Kernel inferior a 6.2 son vulnerables**, con las distribuciones de **Ubuntu, Debian, Amazon Linux, Red Hat**

Entonces comprobemos que versión y que distribución tiene la máquina victima.

Comando **uname -a** para proporcinar info del Kernel.

![](/assets/images/HTB/writeup-twomillion/uname-a.png)

Vemos que tiene una versión **5.15.70**, es inferior a la versión de 6.2, bien miremos ahora cual es la distro de la máquina.

Comando **lsb_release -a** para  mostrar info de la distribución de Linux.

![](/assets/images/HTB/writeup-twomillion/lsb-release-a.png)

La máquina tiene distribución **Ubuntu**, entonces es vulnerable.

Buscando como explotar la vuln, encontre este repositorio: [xkaneiki / CVE-2023-0386](https://github.com/xkaneiki/CVE-2023-0386)

Descargamos el repositorio, lo descomprimimos y seguimos los pasos.

![](/assets/images/HTB/writeup-twomillion/7z.png)

![](/assets/images/HTB/writeup-twomillion/ls-cve.png)

```bash
# Ejecutamos el comando make all
--------------------------------------------------------------
make all
```

Nos genera **tres binarios**, hay que pasarlos a la máquina victima, iniciamos un server en python3 por el puerto **80**.

![](/assets/images/HTB/writeup-twomillion/python3.png)

Damos permisos de ejecución a los archivos y luego ejecutamos lo siguiente **./fuse ./ovlcap/lower ./gc**

![](/assets/images/HTB/writeup-twomillion/chmod.png)

Ok ahora dejamos esa terminal y nos vamos a otra, nos conectamos de vuelta por ssh y ejecutamos el siguiente comando **./exp**

![](/assets/images/HTB/writeup-twomillion/-exp.png)

Y somos root.

![](/assets/images/HTB/writeup-twomillion/root.png)

Busquemos la **Flag**

![](/assets/images/HTB/writeup-twomillion/root-flag.png)

Hay otro archivo que se llama **thank_you.json**, vamos a ver que contiene.

![](/assets/images/HTB/writeup-twomillion/cat-thak_you-json.png)

Esta encoding con **"url"**, usamos [CyberChef](https://cyberchef.org).

![](/assets/images/HTB/writeup-twomillion/encoding-url.png)

El resultado dice que esta encoding en **"hex"**.

![](/assets/images/HTB/writeup-twomillion/encoding-hex.png)

Vemos que esta cifrado con **"xor"** y  tenemos la key **"HackTheBox"** y a la par esta encoding en **"base64"**

![](/assets/images/HTB/writeup-twomillion/message.png)

```text
# Mensaje de agradecimiento
--------------------------------------------------------------
Dear HackTheBox Community,

We are thrilled to announce a momentous milestone in our journey together. With immense joy and gratitude, we celebrate the achievement of reaching 2 million remarkable users! This incredible feat would not have been possible without each and every one of you.

From the very beginning, HackTheBox has been built upon the belief that knowledge sharing, collaboration, and hands-on experience are fundamental to personal and professional growth. Together, we have fostered an environment where innovation thrives and skills are honed. Each challenge completed, each machine conquered, and every skill learned has contributed to the collective intelligence that fuels this vibrant community.

To each and every member of the HackTheBox community, thank you for being a part of this incredible journey. Your contributions have shaped the very fabric of our platform and inspired us to continually innovate and evolve. We are immensely proud of what we have accomplished together, and we eagerly anticipate the countless milestones yet to come.

Here's to the next chapter, where we will continue to push the boundaries of cybersecurity, inspire the next generation of ethical hackers, and create a world where knowledge is accessible to all.

With deepest gratitude,

The HackTheBox Team
```

# Agradecimiento y Opinión 💬 [#](#agradecimiento-y-opinion) {#agradecimiento-y-opinion}
Esta máquina se toco un **CVE**, y mucho **BurpSuite**, tambien el concepto de la máquina me entretuvo y me gusto ya que me uni a [HackTheBox](https://hackthebox.com) este año 2023 y no tuve la posibilidad de realizar el CTF de invitación, tambien el mensaje de mail, para buscar info de como explotar el CVE y el mensaje de agradecimiento, que me gusto mucho y recalco la primera oración del segundo parrafo y el ultimo parrafo.

**"From the very beginning, HackTheBox has been built upon the belief that knowledge sharing, collaboration, and hands-on experience are fundamental to personal and professional growth"**

<span style="color:red"> Span </span>:

"Desde el principio, HackTheBox se ha basado en la creencia de que el intercambio de conocimientos, la colaboración y la experiencia práctica son fundamentales para el crecimiento personal y profesional"

**"We will continue to push the boundaries of cybersecurity, inspire the next generation of ethical hackers, and create a world where knowledge is accessible to all"**

<span style="color:red"> Span </span>:

"Seguiremos ampliando los límites de la ciberseguridad, inspirando a la próxima generación de hackers éticos y creando un mundo en el que el conocimiento sea accesible para todos"
