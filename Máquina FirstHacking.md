# Máquina FirstHacking

Dificultad -> Muy fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
```

--------------
## Explotación

Tan solo tenemos el **ftp** y la versión. Buscamos algún exploit para esa versión.

![Pasted image 20240512012359](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/db8c21d4-3e49-48d5-8050-006c3662b102)

Tenemos uno justo para esta versión.

Lo descargamos:

```shell
searchsploit -m unix/remote/49757.py
```

Lo ejecutamos:

![Pasted image 20240512012452](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/b563a6a1-878f-40bb-bbfd-8a649a66fa24)

Somos **root** XD
Hemos alcanzado el nivel de privilegios máximos en el sistema !
