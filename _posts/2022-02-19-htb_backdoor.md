---
title: Backdoor
published: true
categories: [Linux, Easy]
tags: [Web explotation, Wordpress, LFI, File upload, SUID abuse]
author: Kripteria
image:
  path: https://user-images.githubusercontent.com/55555187/154721648-0688b1f2-e0f0-4bc6-8e4c-92d6a0686284.PNG
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Backdoor es una <font color="#98E256">máquina Easy</font> con temática <font color="#D35400">Linux</font> en la que se toca enumeración y fuzzing de un servicio web, explotación del plugin ebook download, local file inclusion, abuso de local file inclusion para listar directorios cmdline de /proc, explotación de gdbserver, abuso de screen para hacer attach a una sesión privilegiada.

## Reconocimiento

### Nmap

Una vez que la máquina está activa empezamos la fase de reconocimiento de puertos con nmap.

Identificamos los puertos abiertos de la máquina.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.11.125 -oN portScan
```

Obteniendo:

![recon1](https://user-images.githubusercontent.com/55555187/154721508-42def219-86de-4d6f-a6d3-9c1cbb1b17f4.PNG)

Identificamos la versión y el servicio corriendo en los puertos reportados en el escaneo anterior.

```
nmap -sVC -p22,80 10.10.11.125 -oN portSV 
```

Obteniendo:

![recon6](https://user-images.githubusercontent.com/55555187/154722142-b40a905d-5e69-495b-bca6-3bdb30d4f400.PNG)

### Whatweb

Whatweb es una utilidad que nos permite identificar las tecnologías web utilizadas en una página web.

Sabiendo que la máquina tiene un servicio web alojado en el puerto 80 (HTTP) utilizaremos whatweb para obtener información de las tecnologías web que utilizan.

#### Puerto 80

```
whatweb 10.10.11.125:80
```

Obteniendo:

![recon3](https://user-images.githubusercontent.com/55555187/154721512-13e75775-37c5-48ca-9857-3a39f8778f05.PNG)

Como podemos ver, nos encontramos ante un **Wordpress 5.8.1**.

Si buscamos en searchsploit veremos que nos aparecen 3 scripts que no aplican para este caso.

![recon7](https://user-images.githubusercontent.com/55555187/154722989-45491230-7553-4e46-bca3-53182f86dbde.PNG)

### Script http-enum

Mientras se realizaba el reconocimiento básico puse en segundo plano a ejecutarse el script http-enum de nmap que efectúa un descubrimiento de directorios con un diccionario reducido que no es tan invasivo como wfuzz y herramientas similares.

```
nmap -script http-enum 10.10.11.125
```

Obteniendo:

![recon2](https://user-images.githubusercontent.com/55555187/154721510-a7012132-40c4-42f8-a45d-0a76f6aff64a.PNG)

Indagando en el directorio /wp-includes no encontramos nada relevante por lo que decido ver si el directorio **/wp-content/plugins** es accesible.

![recon4](https://user-images.githubusercontent.com/55555187/154721515-d72da8d2-3d6f-449c-aa99-7b2454a33bea.PNG)

Conseguimos acceder al directorio y observamos que está instalado el **plugin** **ebook download**.

![recon5](https://user-images.githubusercontent.com/55555187/154721484-f8ff2281-58da-4d9a-85cf-a856e8fa586e.PNG)

Dentro del directorio ebook-download encontramos un archivo Readme.txt en el que podemos ver la versión del plugin.

Con la versión del plugin buscamos en searchsploit posibles exploits y encontramos un .txt indicándonos que el plugin es vulnerable a **Local file inclusion**.

![exploit1](https://user-images.githubusercontent.com/55555187/154721490-3881246d-1d3b-4386-a232-485affb3c478.PNG)

![exploit2](https://user-images.githubusercontent.com/55555187/154721493-330a8f42-97c1-487d-8e5b-b0a646d9066d.PNG)

## Acceso al sistema

Aprovechándonos de la vulnerabilidad LFI del plugin accedemos al archivo wp-config.php que suele contener los parámetros de configuración de la base de datos.

![exploit3](https://user-images.githubusercontent.com/55555187/154721496-9ea8b321-46b4-4671-ac32-9201852173d2.PNG)
![exploit4](https://user-images.githubusercontent.com/55555187/154721498-561ff19f-3560-4fce-92ca-ca9632f2c8f4.PNG)

Probamos a iniciar sesión en h<span>ttp://</span>10.10.11.125/wp-login.php con las credenciales obtenidas, pero no hay suerte.

Probamos a iniciar sesión por ssh como el usuario wordpresusser pero tampoco hay suerte.

Aprovechando la vulnerabilidad LFI obtengo el **/etc/passwd** de la máquina donde se puede ver que el único usuario válido del sistema es **user**.

![exploit5](https://user-images.githubusercontent.com/55555187/154721503-76ef754a-1274-4fa7-8a8f-ff53fd89a368.PNG)

Probamos a loguearnos como user a través del panel de login de wordpress y por ssh pero no hay suerte.

Aprovechando el LFI intento acceder a la clase *id_rsa* para poder conectarme por ssh pero no obtengo nada.

También pruebo a acceder a otros recursos críticos como logs de acceso para ver si la máquina es vulnerable a *log poisoning* pero también sin éxito.

Web con listas de directorios interesantes a los que acceder explotando LFI: [Local File Inclusion. Total OSCP Guide](https://sushant747.gitbooks.io/total-oscp-guide/content/local_file_inclusion.html)

Tras comprobar que puedo acceder al directorio [/proc](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html) y después de investigar durante un tiempo descubro que es posible listar con que comando un usuario ha ejecutado una acción en el sistema.

Dichos comandos se guardan en el directorio **/proc/[PID]/cmdline** donde PID es el PID del proceso.

Sabiendo esto podemos programar un script en python que ejecute un ataque de fuerza bruta aprovechando el LFI:

```python
import requests

for i in range(800,1000):
	url = "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../proc/" + str(i) + "/cmdline"
	r = requests.get(url)
	print(r.content)
```

El script se podría mejorar filtrando las respuestas que tengan más de un determinado número de bytes para no hacer print de las respuestas que no nos devuelvan nada útil.

Otra forma de realizar esto sería mediante la opción **sniper de burpsuite** de la siguiente forma:

1- Interceptamos la petición a la web con **foxyproxy**.

2- Mandamos la petición al intruder con *ctrl + i*.

![burp2](https://user-images.githubusercontent.com/55555187/154809058-6b9a0938-009e-41ba-9747-f55d3bc610b4.PNG)

3- Marcamos la posición donde queremos realizar el fuzzing.

![burp3](https://user-images.githubusercontent.com/55555187/154809055-cee7f490-7d54-4c49-9cd4-5cccac3d38a8.PNG)

4- Indicamos el tipo de payload con el que queremos fuzzear, en este caso una lista de números.

![burp4](https://user-images.githubusercontent.com/55555187/154809054-eb2fef2c-eb5a-47e9-87d1-3d93c28a39d0.PNG)

5- Nos fijamos en las respuestas que tengan mayor tamaño. 

![burp2](https://user-images.githubusercontent.com/55555187/154809053-474e2c13-c9df-4799-a01b-8bd9fa1fdabd.PNG)

Mediante el script en python obtengo lo siguiente:

<img title="" src="https://user-images.githubusercontent.com/55555187/154809163-e9941c84-4e88-4742-b05e-ed7d17b5de58.PNG" alt="exploit6" width="669">

El usuario **user** ha lanzado un servicio **gdbserver** con la opción --once **por el puerto 1337**.

La opción --once hace que una vez que el servidor gdb ha aceptado la primera conexión, no acepte más. Por eso en el escaneo de Nmap no hemos encontrado el puerto abierto.

Si buscamos exploits para gdbserver mediante searchsploit no encontramos nada pero buscando por internet encontramos que existe un exploit utilizando msfvenom en el módulo **exploit/multi/gdb/gdb_serv_exec**.

Configuro el script de metasploit:

![acc1](https://user-images.githubusercontent.com/55555187/154809441-810027d7-8f7c-4715-a37a-d65403584c9d.PNG)

- TARGET sirve para fijar la arquitectura en la que queremos que se cree el exploit.

- RHOST y RPORT hacen referencia a la máquina victima mientras que LHOST Y LPORT hacen referencia a nuestra máquina.

- PAYLOAD indica que acción queremos realizar cuando se ejecute el exploit. En este caso queremos que nos entable una reverse shell.

Lanzo script de metasploit y obtengo una conexión como el usuario **user**:

![acc2](https://user-images.githubusercontent.com/55555187/154809288-932d80d7-4545-47ad-98f9-277fe4b962ac.PNG)

A continuación, haciendo uso de python spawneo una bash y me lanzo una reverse shell a otra terminal que tengo en escucha por el puerto 4040 para poder realizar un tratamiento de la tty de forma más cómoda.

![acc4](https://user-images.githubusercontent.com/55555187/154809489-f5dfbcad-61b3-4c05-a6c3-ab3e0e35b7ca.PNG)

![acc3](https://user-images.githubusercontent.com/55555187/154809286-8541a3d5-a489-4686-871c-8be5474ac52c.PNG)

### Tratamiento de la tty

Realizo un tratamiento de la tty para tener una shell interactiva y en la que pueda trabajar cómodamente. Para ello lanzo los siguientes comandos:

#### Tratamiento básico

```
script /dev/null -c bash
ctrl + z
stty raw -echo; fg
reset
xterm
export XTERM=xterm
export SHELL=bash
```

#### Escalado de editores de texto

```
stty -a (En una terminal de nuestro equipo para ver los valores de 
            columns y rows)

stty rows [rows] columns [columns]
```

Esto último es para que al abrir nano o algún editor en la máquina víctima nos salga bien escalado en la terminal.

## Escalado de privilegios

Una vez que hemos accedido al sistema hacemos un reconocimiento básico:

- Mediante el comando **id** miramos si estamos en algún grupo interesante.

- Utilizando **sudo -l** comprobamos si tenemos permisos de sudo en algún contexto.

- Con **find / -perm /4000 2>/dev/null** comprobamos si existe *permisos SUID* que podamos explotar.

- Miramos la versión del kernel mediante **uname -a** para ver si es vulnerable a *dirty cow*.



Al realizar el reconocimiento básico vemos que pkexec tiene permisos SUID lo que puede suponer un problema debido a la reciente vulnerabilidad [CVE-2021-4034](https://blog.qualys.com/vulnerabilities-threat-research/2022/01/25/pwnkit-local-privilege-escalation-vulnerability-discovered-in-polkits-pkexec-cve-2021-4034)



Viendo que no sacamos nada útil de momento, lanzo LinPEAS para enumerar el sistema (debería haber lanzado antes pspy para comprobar tareas Cron del sistema ya que linPEAS deja más huella).



Con LinPEAS obtengo 2 vectores potenciales de escalado de privilegios:

![privesc1](https://user-images.githubusercontent.com/55555187/154812674-9a7b785b-8d8d-4dfa-89a2-6e9cf2167659.PNG)

La máquina a priori es vulnerable a pwnKit (parece que la máquina ha sido parcheada por que el exploit no parece funcionar correctamente.).



![privesc0](https://user-images.githubusercontent.com/55555187/154812673-571d4a33-0cc2-43f7-aa22-103c26fcabe0.PNG)

Y se está ejecutando una tarea Cron interesante.



### Escalado aprovechando la tarea Cron



Como podemos observar, la tarea hace lo siguiente:

1- Comprueba si */var/run/screen/S-root* está vacio.

2- En caso de estar vacío lanza el comando **screen -dmS root**



Investigando en las páginas de manual podemos ver que screen -dmS root lo que hace es crear una sesión independiente con identificador "root".

![privesc3](https://user-images.githubusercontent.com/55555187/154812676-ed997548-5982-483e-a3e3-c07c1e253f7e.PNG)
![privesc4](https://user-images.githubusercontent.com/55555187/154812677-acc0167a-68b3-4bac-b0b3-15033022c2f8.PNG)

A su vez, investigando un poco más en las páginas del manual nos encontramos con la siguiente opción:

![privesc2](https://user-images.githubusercontent.com/55555187/154812675-bf214557-114e-4eed-a4f9-667a80dd3e08.PNG)



Por lo que la utilizamos para ganar root:

![privesc6](https://user-images.githubusercontent.com/55555187/154812672-642c1c0f-1825-464f-a124-fc82a00b6ede.PNG)

![privesc7](https://user-images.githubusercontent.com/55555187/154812839-3fdfe18b-a62a-42ad-852c-eb3e46159980.PNG)
