# Popcorn

<p align="center">
  <img width="300" height="300" src="https://raw.githubusercontent.com/iS3g/boxes/master/images/popcorn/popcorn.png">
</p>


## Resumen

Popcorn es una máquina Linux creada por el usuario [Ch4t1](https://www.hackthebox.eu/home/users/profile/1), lanzada el 15 de marzo de 2017. El nivel de seguridad es Medium, pero, en las estadísticas, la mayoría de usuarios la califican como Fácil. En la fase de reconocimiento, se identifica los puertos 21 y 80 abiertos, en donde el puerto 80 tiene configurado apache, luego en la etapa de reconocimiento web, se identificó una aplicación de nombre Torrnet Hoster, que es vulnerable a File Upload. En la fase de explotación se modficaron las cabeceras HTTP para realizar un tampering del archivo a subir (PHP), y finalmente se logró escalar privilegios explotando una vulnerabilidad del sistema basada en PAM MOTD. 

IP 10.10.10.6.


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
### Identificación de Vulnerabilidades

### Escalación de Privilegios
