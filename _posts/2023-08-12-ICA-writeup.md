---
title: ICA VunlHub Walthrough
author: r4v3nC0d3
date: 2023-08-12 11:33:00 +0800
categories: [Hacking, HackTheBox]
tags: [Reconfiguring machine interfaces for correct IP assignment via dhcp (Small bypass to circumvent the password), Abusing qdPM 9.2 - Password Exposure (Unauthenticated), Remote connection to the MYSQL service and obtaining user credentials, SSH brute force with Hydra, Abusing relative paths in a SUID binary - Path Hijacking (Privilege Escalation)]
# pin: true
math: true
mermaid: true
---

Hoy vamos a estar resolviendo la maquina **ICA** de la plataforma de vulnhub


Esta maquina tiene un problema y es que solo se probo en **virtualBox**, por lo tanto si estas usando **vmware** tienes que hacer unos cambios.

![](/assets/img/VulnHub/ICA/ica.png)

Si haces un escaneo en la red con `arp-scan` o `netdiscover` te daras cuenta que no vez la **ip** de la maquina.

![](/assets/img/VulnHub/ICA/arp.png)

Por lo tanto lo que hay que hacer es lo siguiente:

Tenemos que reiniciar la maquina y entrar al **GNU GRUB** tecleando la letra `e` para editar.

![](/assets/img/VulnHub/ICA/e.png)

Vamos a dirijirte a la parte donde dice `ro quiet`

![](/assets/img/VulnHub/ICA/ro.png)

Vamos a borrar `ro quiet` y vas a poner `rw init=/bin/bash`
Presionamos `CTRL=X` o `F10` para arrancar el sistema

![](/assets/img/VulnHub/ICA/init.png)

Se nos cargara una `shell` con privilegios `root`

![](/assets/img/VulnHub/ICA/shell.png)

Ahora vamos `ip a` o `ifconfig` vamos a ver las interfaces, en mi caso la mia se llama `ens33`

![](/assets/img/VulnHub/ICA/ip.png)

Ahora vamos a ver en configuraciones de red las interfaces

```
cat /etc/network/interfaces
```

Y aqui esta el problema, el nombre de la interfaz aqui esta como `enp0s3` y mi interfaz se llama `ens33` por lo tanto solo cambiamos el nombre en `nano`

![](/assets/img/VulnHub/ICA/interface1.png)

![](/assets/img/VulnHub/ICA/interface2.png)

Una vez hecho esto, reiniciamos y listo

# Escaneo de red

Vamos hacer un escaneo a la red para identificar la ip de la maquina usando herramientas como `arp-scan` o `netdiscover` en mi caso lo hare con `arp-scan` 

![](/assets/img/VulnHub/ICA/arp2.png)

# Reconocimiento

Una vez tenemos la ip vamos hacer un escaneo para ver los puertos y servicios expuestos en la maquina usando `nmap`
### Nmap

```
nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.0.0.7 -oG allPorts
```

- `-p-`: Escanea todos los puertos en el rango.
- `--open`: Muestra solo los puertos abiertos.
- `--min-rate 5000`: Establece la tasa mínima de envío de paquetes a 5000 por segundo.
- `-vvv`: Proporciona una salida muy detallada y verbosa.
- `-n`: Desactiva la resolución de nombres DNS.
- `-Pn`: Trata a todas las direcciones IP como activas, omite la detección de hosts.
- `10.0.0.7`: La dirección IP objetivo del escaneo.
- `-oG allPorts`: Genera una salida en formato Greppable (Grep) que muestra todos los puertos y su estado.

![](/assets/img/VulnHub/ICA/nmap.png)

Una vez tenemos los puertos abiertos identificados lo siguiente seria identificar servicios y versiones

```
nmap -p22,80,3306,33060 -sCV 10.0.0.7 -oN targeted  
```

- `-p22,80,3306,33060`: Especifica los puertos a escanear, en este caso, los puertos 22 (SSH), 80 (HTTP), 3306 (MySQL) y 33060 (MySQL alternativo).
- `-sCV`: Realiza un escaneo de versión (identifica versiones de servicios) utilizando varios scripts y detección de servicios.
- `10.0.0.7`: La dirección IP objetivo del escaneo.
- `-oN targeted`: Guarda la salida en un archivo llamado "targeted" en formato normal.

![](/assets/img/VulnHub/ICA/nmap2.png)

Tiene el puerto 22 corriendo ssh pero no tenemos credenciales

Tiene el puerto 80 y esta corriendo apache vamos a darle un vistazo

Tiene el puerto 3306 corriendo mysql

### Puerto 80:HTTP

Tenemos un panel de login con correo que no tenemos, pero veo algo llamado `qdPM 9.2`  vamos a buscarlo en google

![](/assets/img/VulnHub/ICA/http.png)

Bueno de primera ya vemos que esa version es vulnerable

![](/assets/img/VulnHub/ICA/google.png)

![](/assets/img/VulnHub/ICA/exploit.png)

Al parecer en esta ruta `/core/config` nos encontramos con un archivo llamado `databases.yml`

![](/assets/img/VulnHub/ICA/config.png)

Nos descargamos el archivo y tenemos unas credenciales para una base de datos `mysql` 

Como vimos en el escaneo con `nmap` la maquina tiene abierto y corriendo el puerto `3306` con `myslq`

![](/assets/img/VulnHub/ICA/databases.png)

### mysql

Vamos a intentar conectarnos a `mysql` con estas credenciales

```
mysql -u qdpmadmin -p -h 10.0.0.7
```

Estamos dentro de la base de datos

![](/assets/img/VulnHub/ICA/mysql1.png)

```
show databases;
```
![](/assets/img/VulnHub/ICA/mysql2.png)

Vamos a ver si la base de datos `qdpm` tiene informacion util

```
use qdpm;
show tables;
```

Hechemos un vistazo a `users`

![](/assets/img/VulnHub/ICA/mysql3.png)

Esta vacio

![](/assets/img/VulnHub/ICA/mysql4.png)

Chequemos la base de datos `staff`

![](/assets/img/VulnHub/ICA/mysql5.png)

Bueno tenemos usuarios y passwords en base64

![](/assets/img/VulnHub/ICA/mysql6.png)

# Fuerza Bruta (Hydra:SSH)

Lo que podemos hacer es usar `hydra` para hacer hacer fuerza bruta con esas credenciales al servicio `ssh` 

Primero guardamos en un archivo los ussers y en otro las passwords

Puedes aplicar este comando para automatizar el proceso de base64 en las passwords

```
for pass in c3VSSkFkR3dMcDhkeTNyRg== N1p3VjRxdGc0MmNtVVhHWA== WDdNUWtQM1cyOWZld0hkQw== REpjZVZ5OThXMjhZN3dMZw== Y3FObkJXQ0J5UzJEdUpTeQ==; do echo $pass | base64 -d; echo; done > passwords
```

>Asegurate de los nombres de los usuarios esten en minusculas

![](/assets/img/VulnHub/ICA/creds.png)

Uso de Hydra

```
hydra -L users -P passwords ssh://10.0.0.7
```

- `-L users`: Especifica un archivo llamado "users" que contiene una lista de nombres de usuario para probar en el ataque.
- `-P passwords`: Especifica un archivo llamado "passwords" que contiene una lista de contraseñas para probar en el ataque.
- `ssh://10.0.0.7`: Indica el protocolo SSH y la dirección IP objetivo del ataque.

Tenemos user y password validos

![](/assets/img/VulnHub/ICA/hydra.png)

# SSH

Nos conectamos por `ssh` ya que tenemos credenciales validas

```
ssh <user>@<host>
```
![](/assets/img/VulnHub/ICA/ssh.png)

![](/assets/img/VulnHub/ICA/ssh2.png)

### Permisos SUID

Una vez dentro vemos que archivos tiene permisos `SUID`

```
find / -perm -4000 2>/dev/null
```

Bueno ya vemos algo extraño en un archivo `get_access` en el directorio `/opt`

![](/assets/img/VulnHub/ICA/suid.png)

Si hacemos un `file` vemos que es un ejecutable

![](/assets/img/VulnHub/ICA/file.png)

Lo ejecutamos para ver que hace

![](/assets/img/VulnHub/ICA/get_access.png)

Probamos con `strings` para ver los caracteres legibles

Analizando un poco vemos que hay un `cat /root/system.info`

![](/assets/img/VulnHub/ICA/string.png)

esta haciendo uso de el binario `cat` pero no esta usando la ruta absoluta del binario lo que lo hace vulnerable un [[PATH Hijacking]]

# Privilege Escalation

Entonces lo que vamos hacer es irnos a un directorio con capacidad de escritura como `/tmp`

Una vez dentro vamos a crear un archivo que se llame `cat` y dentro de ese archivo vamos a ponder lo siguiente

```
chmod u+s /bin/bash
```

Esta es una manera de darle permisos [[SUID]] a la `/bin/bash`

![](/assets/img/VulnHub/ICA/priv1.png)

Ahora le damos permisos de ejecucion al archivo `cat` con `chmod +x cat`

Lo siguiente seria modificar el PATH para que empiece a buscar primero en la ruta `/tmp` donde tenemos el archivo `cat` 

![](/assets/img/VulnHub/ICA/priv2.png)

Ahora nos vamos al directorio `/opt` donde esta el ejecutable `get_access` y lo ejecutamos

![](/assets/img/VulnHub/ICA/priv3.png)

Ahora hacemos un `ls -l` para ver los permisos de la `/bin/bash`

![](/assets/img/VulnHub/ICA/priv4.png)

Efectivamente ya cuenta con permisos [[SUID]] por lo tanto basta con hacer un `bash -p` para que nos de una shell con privilegios root

![](/assets/img/VulnHub/ICA/root.png)

Para volver a usar el `cat` simplemente cambiamos el PATH a como estaba antes

```
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/games
```
![](/assets/img/VulnHub/ICA/flag.png)
