---
title: Search
published: true
categories: [Active Directory,Hard] 
tags: [Active directory explotation, gMSA abuse, Permission abuse]

image:
  path: https://user-images.githubusercontent.com/55555187/186196250-730acb8c-e3d5-4f67-9aff-2239c7ab5c19.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

Search es una <font color="#C70039">máquina Hard</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca enumeración de servicios web, enumeración del servicio smb, Kerberoasting, Excel protection bypass, abuso del permiso ReadGMSAPassword, Pass the hash sobre rpc y abuso del permiso GenericAll.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.11.129
```

<img src="https://user-images.githubusercontent.com/55555187/185891258-05deba56-b710-4b2c-a722-3f8069304eda.png" title="" alt="image" width="374">

```bash
nmap -sCV -p53,80,88,135,139,389,443,445,464,593,636,3268,3269,8172,9389,49667,49675,49676,49702,49712 -oN portSV 10.10.11.129
```

<img title="" src="https://user-images.githubusercontent.com/55555187/185891823-29813071-0a05-444f-9977-7f8ff8643894.png" alt="image" width="667">

<img title="" src="https://user-images.githubusercontent.com/55555187/185891916-20d0c4dd-6d2f-4867-99a1-5909e59cd83d.png" alt="image" width="687">

Viendo todos los puertos abiertos de la máquina decido probar las siguientes cosas:

1. Investigo las web del puerto 80 y puerto 443 (que al final resultan ser la misma), encuentro un listado de empleados, creo una wordlist usando diferentes combinaciones (inicial + apellido, nombre + apellido) y lanzo un *userenum* con *Kerbrute* obteniendo 0 usuarios válidos.

2. Como *RPC* está abierto, intento conectarme haciendo uso de una *null session rpcclient 10.10.11.129 -U "" -N)*, entro pero no puedo ejecutar comandos por falta de permisos.

3. Intento ver si por *ldap* puedo encontrar algo haciendo uso de *ldapsearch* pero requiere autenticación.

4. Como está el puerto 53 abierto y tenemos una web, lanzo un *domain zone transfer* con dig para ver si me devuelve direcciones web pero no obtengo nada.

5. Intento enumerar *smb* usando una *guest session (smbmap -H 10.10.11.129 -U "" -P ""* pero no obtengo nada.

6. Fuzzeo la web haciendo uso de *secLists*, encuentro algunos directorios pero nada especialmente relevante

7. Me descargo alguna imagen de la web y mediante *exiftool* compruebo si el campo *Creator* me puede proporcionar algún usuario válido pero no es el caso.

Después de bastante rato dando vueltas sin obtener nada, me doy cuenta de que en una de las imagenes de la web hay, lo que parece ser un usuario y una contraseña (WTF) 

<img title="" src="https://user-images.githubusercontent.com/55555187/185896812-897bfb9b-ef0a-4d3c-aa01-0a516ffde94d.png" alt="image" width="438">

Compruebo si existe *Hope Sharp* o HSharp con *Kerbrute* y no existen. En ese momento se me ocurre que también puedo probar con una política de nombres *nombre.apellido* o *inicial.apellido*, pruebo ambas con todos los usuarios que había recolectado y obtengo 4 usuarios válidos:

<img title="" src="https://user-images.githubusercontent.com/55555187/185899207-4376ecc3-6c8c-4c7f-a781-a3140d68a876.png" alt="image" width="519">

Usando *Impacket* compruebo si algún usuario es *ASREPRoasteable* pero no es el caso. Como tengo una credencial (en teoría), hago un passwordspray con *Kerbrute* (hago un *passwordspray* ya que esa credencial podría ser la credencial que les dan a los usuarios por defecto, *Hope Sharp* podría haberla cambiado pero quizás otro usuario no)

```bash
/opt/kerbrute/kerbrute passwordspray --dc 10.10.11.129 -d search.htb potusers.txt IsolationIsKey?
```

<img src="https://user-images.githubusercontent.com/55555187/185899658-37ee1aec-7a1b-400f-8bc3-c36a547bc64a.png" title="" alt="image" width="623">

## Escalada de privilegios

Como no estan los puertos de *Winrm* ni *RDP* abiertos, no merece la pena ni preguntarse si el usuario pertenecerá a un grupo que le permita conectarse remotamente.

Compruebo el servicio *smb* con *smbmap* aportando las credenciales de *Hope Sharp*

```bash
smbmap -H 10.10.11.129 -u hope.sharp -p "IsolationIsKey?"
```

![image](https://user-images.githubusercontent.com/55555187/185900375-42d98104-1e72-48dc-9aeb-27589f70cc39.png)

En *RedirectionalFolders$* podemos ver los directorios personales de algunos usuarios del sistema (aunque no tenemos acceso a la mayoría de ellos)  pero no hay nada relevante

Haciendo uso de *ldapdomaindump* dumpeo la información de dominio y revisando encuentro un par de máquinas que quizás en el futuro puedan ser relevantes:

![image](https://user-images.githubusercontent.com/55555187/185901019-a8d15b8b-fbca-4a00-a3be-f0b7bb4a6a6c.png)

Enumerando la información del dominio a través de *RPC* mediante *rpcclient* y la consulta *querydispinfo* veo que el usuario *Administrator* va a ser deshabilidado y que en su lugar *Tristan.Davies* va a pasar a ser el único usuario privilegiado.

<img src="https://user-images.githubusercontent.com/55555187/185901298-281ac9c4-a928-4b47-bef6-d4b29f25691b.png" title="" alt="image" width="685">

Antes de hacer nada más, como tengo credenciales, compruebo, usando *Impacket*, si algún usuario es *Kerberoasteable* y encuentro una cuenta de servicio:

![image](https://user-images.githubusercontent.com/55555187/185903045-a7e4e4a4-4cd3-457a-a8d7-d1180eae625e.png)

Crackeo su contraseña usando *john* y la valido usando *Crackmapexec*:

![image](https://user-images.githubusercontent.com/55555187/185903366-1d538dfb-1108-4880-914f-24c0f7610576.png)

![image](https://user-images.githubusercontent.com/55555187/185903502-7502436c-0f8a-4aad-8822-60f38dc80fef.png)

Como ya poseo dos usuarios del dominio, decido lanzar *bloodhound* junto a *bloodhound-python* para buscar posibles vías para elevar privilegios.

Revisando el output veo el camino de escalada muy claro, el primer paso es pivotar hacia el usuario *sierra.frye*:

<img src="https://user-images.githubusercontent.com/55555187/185905116-51471012-07dc-4a06-b023-3061922726f0.png" title="" alt="image" width="578">

<img src="https://user-images.githubusercontent.com/55555187/185905218-3c0adc28-5d3f-459b-99e6-4ee91f7adb9e.png" title="" alt="image" width="554">

Revisando todos los servicios y puertos no encuentro nada que me permita, a priori, realizar dicho pivote por lor que se me ocurre que alguna de las contraseñas que tengo puede que sean válidas para otros usuarios, quizás hay un usuario del sistema que gestiona la cuenta *web_svc* y ha reusado su credencial. 

Usando *rpcclient* dumpeo todos los usuarios del dominio y con *crackmapexec* hago un *password spray* usando la opción *--continue-on-sucess* descubriendo que la credencial de *web_svc* es la misma que la de *edgar.jacobs*

![image](https://user-images.githubusercontent.com/55555187/185912591-eafac12b-0385-4e64-a9fe-ddf5f6324ed4.png)

Cuando revisé la carpeta *RedirectedFolders$* en *smb* vi una carpeta con su nombre, pruebo a volver a enumerar el servicio *smb* pero esta vez como edgar:

<img src="https://user-images.githubusercontent.com/55555187/186100833-3877c450-f19a-471b-b852-73c531f427e3.png" title="" alt="image" width="641">

Me llama la atención el archivo *Phising_Attempt.xlsx* por lo que trato de descargarmelo usando *smbmap* y la opción --download:

![image](https://user-images.githubusercontent.com/55555187/186102015-fc869700-a8d0-4360-9602-04f16eda0ce5.png)

Al abrir el archivo me encuentro con que contiene información sobre un grupo de usuario, observo también que abajo pone *passwords* y que la columna C está protegida:

<img title="" src="https://user-images.githubusercontent.com/55555187/186104940-36f98205-d3b1-437f-b798-bbafa4cdeac5.png" alt="image" width="408">

Buscando en internet vias de bypasear esta protección encuentro el siguiente post donde explican varias formar de desproteger el documento [How to Unlock Cells without Password in Excel](https://www.exceldemy.com/unlock-cells-in-excel-without-password/#1_Remove_Password_to_Unlock_Cells_in_Excel)

1. Descomprimo el documento xlsx:
   
   <img title="" src="https://user-images.githubusercontent.com/55555187/186105814-6c39c305-9075-4f49-a461-b2051e3534c6.png" alt="image" width="233">

2. Reviso el contenido de *xl/worksheets/sheet1.xml* y *xl/worksheets/sheet2.xml*, en sheet2.xml encuentro el campo *sheetprotection*
   
   ![image](https://user-images.githubusercontent.com/55555187/186105734-ec4d0e86-5477-41b1-9f28-af24e9317257.png)

3. Usando *xmllint --format* abro el archivo en formato *xml*, copio el contenido, sobreescribo el contenido de *sheet2.xml* con él y elimino la fila *sheetprotection*
   
   ![image](https://user-images.githubusercontent.com/55555187/186106493-5c05a819-8edc-487e-a28a-27a25ac24635.png)

4. Una vez hecho eso, comprimo todos los archivos nuevamente a formato *zip* y posteriormente cambio la extensión a *xlsx*
   
   ![image](https://user-images.githubusercontent.com/55555187/186107737-6486b865-50c7-499d-8216-60dfb7f728a2.png)

Ahora al abrir el archivo que acabo de crear puedo ver la columna C con las contraseñas de los usuarios:

![image](https://user-images.githubusercontent.com/55555187/186107642-16befa7b-5355-41db-b88b-190ff8bcc1b0.png)

Entre esas contraseñas encuentro la de *sierra.frye*, la valido usando *Crackmapexec* (importante escapar los signos *$*):

```bash
crackmapexec smb 10.10.11.129 -u sierra.frye -p "\$\$49=wide=STRAIGHT=jordan=28\$\$18"
```

![image](https://user-images.githubusercontent.com/55555187/186108105-4bab49fa-67fe-43c6-ad5d-ca68c91b93a5.png)

Como *sierra.frye* pertenece al grupo *ITSec* y dicho grupo tiene el permiso *ReadGMSAPassword* sobre el usuario *BIR-ADFS-GMSA$*, usando [gMSADumper](https://github.com/micahvandeusen/gMSADumper) puedo obtener el hash *LM* del usuario con el que puedo hacer *Pass The Hash*:

![image](https://user-images.githubusercontent.com/55555187/186108479-122dca76-f237-45a8-afd9-98e45eec4821.png)

Teniendo el hash *LM* del usuario y sabiendo que dicho usuario tiene permisos *GenericAll* sobre *Tristan.Davies* lo primero que se me ocurre es lo siguiente:

1. Accedo a *RPC* como *BIR-ADFS-GMSA$* usando *pth-rpcclient* y haciendo un *pass the hash*, cambio la contraseña de *Tristan.Davies* haciendo uso de la función *setuserinfo2*:
   
   <img src="https://user-images.githubusercontent.com/55555187/186112633-7311b6ba-7571-4cd2-96d8-f2d4a1f97822.png" title="" alt="image" width="657">

2. Compruebo a través de *Crackmapexec* que la nueva contraseña de *Tristan.Davie* es la que le he puesto:
   
   <img src="https://user-images.githubusercontent.com/55555187/186112749-5258e86c-185d-4500-a644-744e7ab74fc0.png" title="" alt="image" width="644">

3. Haciendo uso de *wmiexec* me conecto a la máquina como *Tristan.Davie* (Administrator):
   
   <img title="" src="https://user-images.githubusercontent.com/55555187/186113219-3300160b-6d2a-4332-974e-1d364f60fbf7.png" alt="image" width="626">
