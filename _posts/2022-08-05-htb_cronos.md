---
title: Cronos
published: true
categories: [Linux,Medium]
tags: [Web explotation, Domain zone transfer, Cron abuse]
author: Alejandro Rivera
image:
  path: https://user-images.githubusercontent.com/55555187/183108404-1544dff6-cd66-4339-b3a4-f3333245939f.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Cronos es una <font color="#FFC300">máquina Medium</font> de temática <font color="#D35400">Linux</font> en la que se toca domain zone transfer para enumerar subdominios, EAR (Execute After Redirect) permitiendo entablar una reverse shell y explotación de tareas cron permitiendo lanzar comandos como root.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.10.13
```

![image](https://user-images.githubusercontent.com/55555187/182629438-497c6a4d-8966-4257-90c4-532ddd241e88.png)

```bash
nmap -sVC -p22,54,80 -oN portSV 10.10.10.13
```

![image](https://user-images.githubusercontent.com/55555187/182629833-1061724a-d8f1-456f-9fa9-f68dd5bdb43d.png)

Lo primero que me llama la atención es el puerto 53 abierto, si está mal configurado puede dar lugar a un *Domain Zone Transfer* donde obtengamos todas las direcciones del servidor. 

Lanzo un *Domain Zone Transfer* con *dig*:

```bash
dig AXFT @10.10.10.13 cronos.htb
```

El nombre del dominio no es posible obtenerlo de ninguna forma durante la enumeración, es adivinarlo teniendo en cuenta que casi todas las máquinas de Hack The Box usan el formato <span><</span>nombre máquina>.htb

![image](https://user-images.githubusercontent.com/55555187/182638812-b7d22ed6-d603-4f70-b336-d2bcd2edeeb0.png)

Añado las direcciones al archivo */etc/hosts* y sigo enumerando.

- En *ns1.cronos.htb* tenemos la página de inicio de Apache, nada interesante

- En *admin.cronos.htb* tenemos un panel de login

- En cronos.htb tenemos la web principal que parece solo tener enlaces a sitios externos

Durante la enumeración realizo las siguiente pruebas:

- Busco recursos en *www.cronos.htb* y encuentro los directorios /js y /css sin nada relevante

- Lanzo un fuzzing para obtener recursos válidos en *admin.cronos.htb* pero no obtengo nada

- Como sé que se está usando *Laravel*, que es un framework de php, vuelvo a lanzar el fuzzing pero con extensión .php obteniendo:
  
  ![image](https://user-images.githubusercontent.com/55555187/182641883-656fb57a-12da-498d-8331-8a23769166fc.png)

En *welcome.php* observo un formulario que parece permitir lanzar comandos de networking, a pesar de que *welcome.php* hace una redirección, podemos ver su contenido y ejecutar peticiones ya que es vulnerable a EAR (Execution After Redirect)

<img title="" src="https://user-images.githubusercontent.com/55555187/182642151-c2ae1e57-38b9-4041-bc53-307f3e22815e.png" alt="image" width="428">

Para comprobar si tengo *RCE*, lanzo un ping usando el comando del formulario contra mi equipo:

<img src="https://user-images.githubusercontent.com/55555187/182643594-640a86c8-88ae-4238-a9d1-38504f0f410e.png" title="" alt="image" width="432">

Viendo que el formulario funciona, cambio el comando para que sea *curl* y haga una petición *GET* contra un servidor que voy a levantar con *Python*

![image](https://user-images.githubusercontent.com/55555187/182645949-ad0158cd-2607-49c3-9fea-289e09b5ca9a.png)

He recibido la petición y encima en la respuesta se puede ver el valor de *index.html* que previamente había creado.

La idea de esta comprobación es mirar si el firewall permitía conexiones salientes desde ese formulario o si había alguna regla rara que impidiese por ejemplo entablar una revese shell a través de este método.

Antes de probar a entablar una reverse shell, leo el contenido de *config.php* el cual sé que se encuentra en el directorio actual ya que he lanzado un *ls*

<img src="https://user-images.githubusercontent.com/55555187/182646650-5dfaa932-2c96-40c0-92b3-a62cb79bad59.png" title="" alt="image" width="388">

Compruebo si me puedo loguear en el panel de administración y por *SSH* pero no es el caso.

Compruebo la existencia de netcat en la máquina víctima usando el comando *which netcat* y posteriormente compruebo si existe la opción *-e* usando *man netcat* pero por desgracia no está habilitada en este *netcat*:

![image](https://user-images.githubusercontent.com/55555187/182647465-396e6286-1c6b-477e-a1e9-55139cbcdfbf.png)

Aún así, existe otro método para obtener una reverse shell con *netcat* usando *mkfifo*, obtengo una reverse shell:

<img src="https://user-images.githubusercontent.com/55555187/182648598-c4ceca21-df18-473c-a843-81bb0c61cc63.png" title="" alt="image" width="626">

## Escalada de privilegios

Lo primero que se me viene a la cabeza es el *MySQL* ya que a priori tengo credenciales válidas

<img src="https://user-images.githubusercontent.com/55555187/182650804-c5f31ae1-5521-4b23-8b98-ec853d7d1318.png" title="" alt="image" width="607">

Con *hashid* identifico que el hash es un *MD5* y lo intento crackear con *john*, como no obtengo nada, pruebo a crackearlo online y obtengo lo siguiente:

![image](https://user-images.githubusercontent.com/55555187/183107161-144fb436-8682-4bab-ac78-7e31dcfa222b.png)

Compruebo si es la contraseña de root (reutilización de credenciales) pero no es el caso.

Paso *pspy* a la máquina víctima, lo lanzo para ver si hay tareas *cron* programadas y encuentro que root ejecuta la siguiente:

![image](https://user-images.githubusercontent.com/55555187/182654187-0dca2957-5963-46a0-84a5-5ccdb8bb93d4.png)

Tengo permisos totales en ese directorio por lo que podría modificar artisan para obtener una shell con privilegios.

Hago una copia del archivo artisan y modifico el original haciendo que root ejecute un *chmod 4755* a la */bin/bash* haciendo que podamos posteriormente obtener una shell como root usando *bash -p*:

![image](https://user-images.githubusercontent.com/55555187/183104033-9b1d5732-9fba-4b25-95c5-d6f6ba15eaa6.png)

![image](https://user-images.githubusercontent.com/55555187/183104460-caccd020-4db4-4d5a-bedc-c01182a9fdc9.png)

![image](https://user-images.githubusercontent.com/55555187/183104973-2e6ef62d-a579-4100-bc57-3c35c715bce5.png)

```bash
/bin/bash -p
```

![image](https://user-images.githubusercontent.com/55555187/183105045-093f8ebe-1bd0-4ea8-976e-e034e0d66c9d.png)
