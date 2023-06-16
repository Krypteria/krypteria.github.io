---
title: Mantis
published: true
categories: [Active Directory,Hard]
tags: [Active directory explotation, MSSQL explotation, Permission abuse, MS14-068]

image:
  path: https://user-images.githubusercontent.com/55555187/185653921-771b71ec-5cc0-4562-9f68-f23db9c59da4.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

Mantis es una <font color="#C70039">máquina Hard</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca enumeración de servicios web a través de fuzzing, descifrado de credenciales, MSSQL explotation a través de DBeaver y explotación del MS14-068.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 1000 -Pn -n -vvv -oG portScan 10.10.10.52
```

<img src="https://user-images.githubusercontent.com/55555187/185568989-12bc329a-9a09-49c0-8d64-3953cdc40327.png" title="" alt="image" width="342">

```bash
nmap -sCV -p53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,47001,49152,49153,49154,49155,49157,49158,49164,49166,49168,50255 -oN portSV 10.10.10.52
```

<img src="https://user-images.githubusercontent.com/55555187/185569683-6047add1-aa70-4ab5-96d1-73ae229fbb78.png" title="" alt="image" width="607">

<img title="" src="https://user-images.githubusercontent.com/55555187/185569743-0cdd49a5-864f-494a-aec6-fd0f5a4a91dd.png" alt="image" width="608">

1. Compruebo si puedo acceder a *RPC* usando *rpcclient* y una *null session* pero no es el caso.

2. Compruebo *ldap* con una autenticación simple pero me dice que requiere credenciales

3. Compruebo si hay algún archivo público compartido en *smb* usando *smbmap* y *smbclient* pero no encuentro nada.

Mirando el output de *nmap* se puede ver que el servidor está alojando dos webs, una en el puerto *1337* y otra en el *8080*. Revisando la del puerto 8080 encuentro que están usando el *cms Orchard* y la web cuenta con un blog, una sección de comentarios y un panel de autenticación.

Revisando las vulnerabilidades del *cms* veo que la sección de comentarios tiene un *XSS* asociado, además al enviar un comentario sale el siguiente output:

![image](https://user-images.githubusercontent.com/55555187/185570271-07ae74d6-da64-49a6-b517-1eccc3889df0.png)

![image](https://user-images.githubusercontent.com/55555187/185572121-557afd40-86e2-4c87-baa8-17c31ad5c4d3.png)

Viendo esto se me ocurre el siguiente vector:

*XSS en comentario -> Administrador abre el comentario para revisarlo -> a través del XSS me envío la cookie de sesión a mi servidor -> session hijacking y accedo al panel de administración*

<img src="https://user-images.githubusercontent.com/55555187/185590916-f3d3112f-0ce1-43b4-a632-6421a8f994a0.png" title="" alt="image" width="302">

Lamentablemente no parece funcionar lo cual tiene sentido por varios motivos:

1. No conozco si la versión de *Orchard* de la web es vulnerable

2. No parece que se apruebe ningún tipo de comentario, no hay feedback de que alguien revise nada.

Viendo que en la web del puerto *8080* no puedo hacer mucho, reviso la del puerto *1337* que resulta ser el panel default de *IIS*, lanzo un *gobuster* para ver si hay algún directorio interesante:

```bash
gobuster dir -u http://10.10.10.52:1337 -t 50 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

![image](https://user-images.githubusercontent.com/55555187/185586804-4dbc3911-e940-4c39-8e90-cb0d3466ca1b.png)

Encuentro un directorio que tiene una nota de los desarrolladores indicando como ha sido el proceso de creación de la web del puerto *8080*

<img title="" src="https://user-images.githubusercontent.com/55555187/185587036-3d62bd3d-8114-4447-a9a2-e9b4bd34590c.png" alt="image" width="526">

<img src="https://user-images.githubusercontent.com/55555187/185587259-a3379724-ad12-4e9e-9bd6-95710ede4351.png" title="" alt="image" width="581">

Me llama la atención la cadena de texto que hay en el nombre del archivo, lo descifro ya que parece *base64* obteniendo lo que parece una cadena en *hexadecimal*, la descifro y obtengo lo que parece ser la contraseña para la base de datos:

![image](https://user-images.githubusercontent.com/55555187/185590658-4bec29ef-00b5-4135-99c9-c131986d3f36.png)

Con esto tenemos la siguiente información:

```bash
database: orcharddb
database version: SQL server 2014 Express
database password: m$$ql_S@_P@ssW0rd!
user: admin
```

Utilizando *DBeaver* me conecto a la base de datos de la máquina:

<img src="https://user-images.githubusercontent.com/55555187/185601227-83cb70c0-0e72-4220-bdad-5c08f28e677b.png" title="" alt="image" width="420">

Mirando las tablas identifico una que parece tener información de los usuarios, viendo su contenido identifico una contraseña en texto claro para el usuario *James*:

![image](https://user-images.githubusercontent.com/55555187/185601449-45b3b8cb-6c6b-4929-ac02-8265d212a282.png)

Valido la credencial haciendo uso de *Crackmapexec*:

![image](https://user-images.githubusercontent.com/55555187/185601799-6ce1d0f2-a186-4975-8531-51d4de3c4508.png)

## Elevación de privilegios

Como no está *winrm* ni *RDP* abiertos, no puedo conectarme a la máquina por lo que decido hacer un dump de la información del dominio usando *ldapdomaindump*:

![image](https://user-images.githubusercontent.com/55555187/185606168-b0bc960e-b303-4f5a-9fd5-e63e72e14535.png)

Si *RDP* estuviese abierto, podría conectarme a la máquina ya que *James* pertenece al grupo *Remote Desktop User*.

Revisando la información no consigo ver nada relevante, usando *bloodhound-python* junto a *bloodhound* reviso el dominio y las relaciones entre usuarios pero tampoco encuentro nada útil.

Decido investigar sobre ataques comunes a directorio activo y descubro que en el repositorio [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) hay una sección orientada puramente a ataques a directorio activo.

La primera vulnerabilidad de la que se habla es la **MS14-068** que hace referencia a un fallo que permite que el DC dé como válido un ticket simplemente comprobando que el checksum tenga un valor válido sin comprobar si la información de los grupos es válida haciendo que un usuario pueda crearse un ticket como *Administrator*.

Buscando encuentro un script en *Python* [ms14-068.py](https://raw.githubusercontent.com/SecWiki/windows-kernel-exploits/master/MS14-068/pykek/ms14-068.py) que hace uso de *Pykek* para crear el ticket, pruebo con él pero no consigo conectarme a la máquina:

![image](https://user-images.githubusercontent.com/55555187/185612323-0f95fe1b-c408-41db-8bd1-7e949c851409.png)

Al rato veo que el script *goldenPac.py* de *Impacket* hace uso de la vulnerabilidad y te genera directamente una shell ya que junta la utilidad del anterior exploit con *psexec*:

![image](https://user-images.githubusercontent.com/55555187/185612667-42814122-7a9b-4626-9470-0aa162281c3e.png)
