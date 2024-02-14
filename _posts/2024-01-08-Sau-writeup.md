---
title: Sau - Hack The Box
author: r4v3nC0d3
date: 2024-01-07 11:33:00 +0800
categories: [Hacking, HackTheBox]
tags: [request-baskets 1.2.1 Explotation (SSRF - Server Side Request Forgery), Maltrail 0.53 Explotation (RCE - Username Injection), Abusing sudoers privilege (systemctl) (Privilege Escalation)]
# pin: true
math: true
mermaid: true
---

En esta máquina, abordamos múltiples vulnerabilidades que abarcan desde la exposición de servicios mal configurados, pasando por vulnerabilidades de SSRF en aplicaciones web, hasta vulnerabilidades de ejecución remota de código (RCE).

## Reconocimiento (Reconnaissance)

Verificación de Conectividad con Ping:
Antes de iniciar cualquier exploración o escaneo, es fundamental asegurarse de que hay conectividad con el objetivo. Para ello, utilizaremos la herramienta ping.

```
ping -c1 10.10.11.224
```

El comando ping nos permitirá confirmar si la máquina objetivo, con la dirección IP 10.10.11.224, está activa y responde a solicitudes ICMP. Si recibes respuestas, significa que hay conectividad básica con el objetivo.

El parametro `-c1` es para indicarle que solo queremos enviar un paquete icmp.

![](/assets/img/HackTheBox/Sau/ping.png)

Identificación del Sistema Operativo con el TTL:
Una observación importante durante el proceso de ping es el valor del TTL (Time To Live) en las respuestas recibidas. El TTL puede proporcionar información valiosa sobre el sistema operativo del objetivo:

TTL = 64: Generalmente, un TTL de 64 indica que el sistema operativo es Linux.
TTL = 128: Un TTL de 128 suele ser característico de sistemas Windows.
Esto nos da una primera indicación sobre el sistema operativo en el que podría estar basado el objetivo, lo cual es útil para adaptar nuestras estrategias de enumeración y explotación.

### Nmap

Voy a comenzar haciendo un escaneo con nmap para revisar si hay puertos expuestos.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.224 -oG allPorts
```
#### Descripción de las Opciones Utilizadas:

```bash
-p-: Escanea todos los puertos (del 1 al 65535).
--open: Muestra solo los puertos que están abiertos.
-sS: Realiza un escaneo de tipo SYN (half-open) para identificar puertos abiertos.
--min-rate 5000: Establece una velocidad mínima de envío de paquetes de 5000 paquetes por segundo, lo que acelera el proceso de escaneo.
-vvv: Aumenta el nivel de verbosidad para obtener una salida más detallada y comprensible.
-n: Evita la resolución de DNS inverso para las direcciones IP y muestra las direcciones en formato numérico.
-Pn: Ignora el descubrimiento de hosts y trata la máquina objetivo como si estuviera en línea.
10.10.11.224: Dirección IP de la máquina objetivo que se va a escanear.
-oG allPorts: Guarda los resultados del escaneo en formato "grepable" en un archivo llamado "allPorts" para futuros análisis.
```

Tiene dos puertos abiertos `22` y `55555`.

![](/assets/img/HackTheBox/Sau/nmap1.png)

Ahora vamos a escanear especificamente esos puertos con una serie de scripts de reconocimeinto de nmap y checar las versiones.

```bash
nmap -p22,55555 -sCV 10.10.11.224 -oN targeted
```

![](/assets/img/HackTheBox/Sau/nmap2.png)

Tras realizar el escaneo, identificamos que el puerto 22 está abierto y ejecuta el servicio ssh. Sin embargo, debido a las siguientes consideraciones, decidimos descartar este servicio como punto de entrada:

**Falta de Credenciales Válidas:** Aunque el servicio SSH está activo, no contamos con credenciales válidas para autenticarnos en el sistema. La falta de credenciales adecuadas nos impide avanzar en una posible explotación o enumeración a través de SSH.

**Versión del Servicio SSH:** La versión del servicio SSH detectada no es la 7.7, que es conocida por su enfoque en la validación de usuarios. Las versiones anteriores a 7.7 pueden presentar vulnerabilidades que facilitan la explotación, pero en este caso, la versión no proporciona una ruta clara para la validación de usuarios.

Debido a estas razones, hemos decidido descartar el puerto 22 y el servicio SSH como área de enfoque en este momento.

Por otro lado, el puerto `55555` está abierto, pero no pudimos identificar el servicio específico en ejecución. Sin embargo, observando los headers HTTP, es posible inferir que este puerto podría estar alojando un sitio web.

Para explorar más a fondo, podemos abrir un navegador web y acceder a la dirección IP de la máquina objetivo seguida del puerto 55555. Por ejemplo, http://10.10.11.224:55555.

![](/assets/img/HackTheBox/Sau/web1.png)

### Wappalyzer

Estas son las tecnologias que esta corriendo por detras, no veo que sea un gestor de contenido no te detecta tipo cms wordpress, drupal.

![](/assets/img/HackTheBox/Sau/wappalyzer.png)

### Web

En la parte de abajo de la pagina web podemos ver `Powered by request-baskets | Version: 1.2.1`.

![](/assets/img/HackTheBox/Sau/web2.png)

Solo recordar que nosotros como atacante el ver esto, lo que se esta usando y la version esto para un atacante es oro por que si esta desactualizado pueden haber vulnerabilidades y todo esto.

Al pinchar en `request-baskets` me lleva a un proyecto de github.

![](/assets/img/HackTheBox/Sau/github.png)

En resumen, Request Baskets es una herramienta útil para el análisis y la depuración de solicitudes HTTP, ofreciendo capacidades de recolección, visualización y prueba que pueden ser aprovechadas en una variedad de escenarios, incluidos el desarrollo de aplicaciones, las pruebas de penetración y la seguridad de las aplicaciones web.

Tambien vemos que la ultima version es la `1.2.3` y la version que se esta utilizando en el sitio web es la `1.2.1`, esta desactualizada.

Asi que hay que buscar vulnerabilidades en esa version.

## Request-Baskets 1.2.1 Server-Side Request Forgery (CVE-2023–27163)

![](/assets/img/HackTheBox/Sau/google.png)

Tenemos un recurso de medium que explica como funciona la vulnerabilidad.

![](/assets/img/HackTheBox/Sau/medium.png)

Por lo que comenta el articulo este es un caso en el cual me interesa saber que puertos estan filtrados, y no abiertos.

Voy a realizar un escaneo con nmap pero sin el parametro `--open`.

Si estan abiertos y tengo un SSRF igual y puedo llegar a ver puertos internos.

![](/assets/img/HackTheBox/Sau/filtrados.png)

Puerto `80` y `8338` filtrados, nmap no sabe del todo si esta abierto o cerrado y no tengo forma de verlo, si por ejemplo voy al navegador y coloco esta ruta `http://10.10.11.224:80` no me va cargar nada, pero claro a travez de un SSRF tu explotas esta vulnerabilidad y desde el panel web si que podrias intentar llegar a este.

Lo que vamos hacer es crear un basket.

![](/assets/img/HackTheBox/Sau/basket.png)

![](/assets/img/HackTheBox/Sau/basket2.png)

![](/assets/img/HackTheBox/Sau/basket3.png)

Vamos a enviar una solicitud.

![](/assets/img/HackTheBox/Sau/basket4.png)

Ahora vemos la solicitud que hicimos.

![](/assets/img/HackTheBox/Sau/basket5.png)

Al hacer click en configuracion nos muestra un campo `Forward URL`.

![](/assets/img/HackTheBox/Sau/forward.png)

Aqui podemos poner una URL, (por ejemplo yo me voy montar un servidor http con python por el puerto 80)

![](/assets/img/HackTheBox/Sau/python.png)

Coloco mi ip en el campo `Forward URL`.

![](/assets/img/HackTheBox/Sau/forward2.png)

![](/assets/img/HackTheBox/Sau/forward3.png)

Ahora hacemos otra solicitud.

![](/assets/img/HackTheBox/Sau/test.png)

![](/assets/img/HackTheBox/Sau/python2.png)

Recibimos la solicitud por lo tanto el Forwarding esta funcionando, procesa internamente la solicitud y luego me lo muestra.

Otro ejemplo es que si creo un archivo index.html y monto de nuevo el servidor con python vemos como me lo muestra.

![](/assets/img/HackTheBox/Sau/index.png)

![](/assets/img/HackTheBox/Sau/index2.png)

El recurso de medium comentaba que la version 1.2.1 era vulnerable a un SSRF, por lo tanto lo que hay que probar es colocar en Forward URL esta ruta `http://127.0.0.1`, como la propia maquina la que va hacer esta solicitud asi misma, claro la propia maquina asi misma si que puede ver los servicios internos que esta corriendo la maquina, tu publicamente desde afuera no lo ves pero desde aqui...

![](/assets/img/HackTheBox/Sau/SSRF.png)

Ahora volvemos a enviar la solicitud.

![](/assets/img/HackTheBox/Sau/SSRF2.png)

OJO...

![](/assets/img/HackTheBox/Sau/SSRF3.png)

Lo unico es que no se ve bien.

Si hago ctrl + u, veo que esta intentando cargar recursos.

![](/assets/img/HackTheBox/Sau/u.png)

Es solo de agregar un slash al final del nombre del basket para que pueda cargar correctamente los recursos.

Nos muestra un panel de Authentication, probamos password guessing pero nada.

## Maltrail-v0.53-Exploit

En la parte de abajo del panel muestra `Powered by Maltrail (v0.53)`

![](/assets/img/HackTheBox/Sau/maltrail.png)

Buscamos vulnerabilidades en esa version.

![](/assets/img/HackTheBox/Sau/rce.png)

Y OJO... tenemos un RCE (Remote Code Execution) y un exploit.

![](/assets/img/HackTheBox/Sau/exploit.png)

La vulnerabilidad recide en el panel de Authentication y puede ser explotado via el parametro `username`.

No voy hacer uso del exploit, lo que voy hacer es analizar el exploit y ver que hace por detras para hacerlo de forma manual.

![](/assets/img/HackTheBox/Sau/ex1.png)

A la URL objetivo se le agrega una ruta `/login`

![](/assets/img/HackTheBox/Sau/ex2.png)

Y hace una peticion POST a la URL con curl enviando el payload por el parametro `username` haciendo uso de `;` y `backsticks`.

![](/assets/img/HackTheBox/Sau/ex3.png)

Ahora le voy hacer es ponerme en escucha por la interfaz tun0 en espera de trazas icmp y probar enviar un ping a mi maquina desde la maquina victima.

![](/assets/img/HackTheBox/Sau/ex4.png)

La maquina victima me a enviado un ping, por lo tanto eso significa que tenemos ejecucion remota de comandos.

Osea de un panel de autenticacion el campo de usuario vulnerable a injeccion de comandos... nunca olvides de probar estas cosas en paneles de autenticacion.

![](/assets/img/HackTheBox/Sau/nc.png)

![](/assets/img/HackTheBox/Sau/curles.png)

Como la maquina cuenta con la herramienta `curl` lo que podemos hacer es crear un archivo `index.html` que contenga una reverse shell en bash, para lenvantar un servidor con python y con curl ejecutar el archivo en la maquina victima y enviarme una reverse shell a nc en el puerto 443.

Ahora ya estamos dentro de la maquina como el usuario `puma`

![](/assets/img/HackTheBox/Sau/rev.png)

## User FLag

![](/assets/img/HackTheBox/Sau/userf.png)

## Vulnerabilidad subprocess.check_output()

![](/assets/img/HackTheBox/Sau/vuln.png)

## Privilege Escalation

`sudo -l` para ver si tengo algun privilegio a nivel de sudoers.

![](/assets/img/HackTheBox/Sau/sudo.png)

Puedo ejecutar como quien quiera sin proporcionar password este comando `/usr/bin/systemctl status trail.service`

![](/assets/img/HackTheBox/Sau/sys.png)

Normalmente con `systemctl` cuando estas viendo un servicio o estas consultando algo, te lo pone en formato **paginate** como `less`.

Ahora en este formato reduciendo y manipulando el total de columnas y filas en la terminal hay una forma en ese formato de ejecutar un comando presionando el caracter `!` y seguido el comando.

![](/assets/img/HackTheBox/Sau/rooted.png)

## R00T3D by ch4r0niv