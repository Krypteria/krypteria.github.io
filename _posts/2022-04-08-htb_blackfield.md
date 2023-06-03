---
title: Blackfield
published: true
categories: [Active Directory,Hard] 
tags: [Active directory explotation, Kerberos, Privilege abuse]
author: Kripteria

image:
  path: https://user-images.githubusercontent.com/55555187/162499279-5e8afb95-25d0-4bc9-b7cb-6816f9a23c40.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Blackfield es una <font color="#C70039">máquina Hard</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca enumeración del servicio samba, validación de usuario con kerbrute, ASREPRoast en Kerberos, Privilege abusepara pivotar a través de rpc, dumpeo del lsass y abuso de privilegio SeBackup y SeRestore.

## Reconocimiento

Una vez que compruebo que la máquina es accesible, lanzo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.192
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/162497150-78d7c2ac-6f7d-4c83-be63-ef028493b49e.png" title="" alt="recon1" width="277">

Lanzo un escaneo con nmap para determinar la versión y servicio de los puertos abiertos.

```
nmap -sCV -p53,88,135,389,445,593,3268,5985 10.10.10.192
```

Obteniendo:

![recon2](https://user-images.githubusercontent.com/55555187/162497152-6d949094-fb51-44c0-a28d-09f0b680b3b7.PNG)

Viendo los puertos abiertos, pruebo a enumerar el servicio **ldap** a través de consultas anónimas pero no obtengo nada útil. Pruebo a enumerar el servicio **rpc** mediante **rpcclient** mediante una null sessión pero tampoco obtengo nada.

Por último, enumero el servicio smb utilizando **smbmap**.

```
smbmap -H 10.10.10.192
```

Pero no obtengo nada. 

Pruebo a conectarme mediante **smbclient** y un usuario vacío y ahí si que obtengo una serie de carpetas compartidas. Viendo esto decido volver a intentarlo por smbmap haciendo uso de un usuario cualquiera.

```
smbmap -H 10.10.10.192 -u "kripteria"
```

Obteniendo:

![fh1](https://user-images.githubusercontent.com/55555187/162497092-5b75dc53-a0e1-4c79-8b01-13a853f034bd.PNG)

Muestro el contenido del recurso **profiles$** de forma recursiva obteniendo lo que parece una lista de usuarios.

```
smbmap -H 10.10.10.192 -u "kripteria" -R profiles$
```

<img src="https://user-images.githubusercontent.com/55555187/162497097-4fd27bb4-597f-4384-b066-df5419a8cb94.PNG" title="" alt="fh2" width="589">

Valido los usuarios con **kerbrute** para identificar usuarios válidos del dominio.

```
kerbrute userenum users.txt -d blackfield.local
```

<img src="https://user-images.githubusercontent.com/55555187/162497101-5d946f8a-efda-4867-a2ce-6455a228029b.PNG" title="" alt="fh3" width="623">

Como la nueva versión de kerbrute también aplica **ASREPRoast** mientras valida usuarios, descubro que el usuario **support** tiene el parámetro **DONT_REQ_PREAUTH** activo permitiendo obtener un **KRB_AS_REP** cifrado con su contraseña.

## Obtención de acceso

Crackeo el hash con **hashcat** y valido las credenciales con **crackmapexec**.

```
hashcat -m 18200 -a 0 -d 1 hash /usr/share/wordlists/rockyou.txt
```

![fh4](https://user-images.githubusercontent.com/55555187/162497103-57f181d5-bb7c-4a10-b78a-f657839f97cb.PNG)

Con estas nuevas credenciales compruebo una serie de cosas:

- Miro el servicio samba por si el usuario support tuviese acceso a nuevas carpetas compartidas pero no hay nada relevante.

- Intento loguearme en el servicio rpc mediante rpcclient, lo consigo y empiezo a enumerar usuarios, grupos y descripciones del dominio.

- Realizo un dump del ldap usando **ldapdomaindump** y observo que el usuario svc_backup existe (lo cual tendrá relevancia más adelante) y pertenece al grupo **Remote Management Users**.

- Compruebo si support es **Kerberoasteable** pudiendo obtener TGS utiles pero no es el caso.

Viendo que nada de lo anterior me ha reportado nada útil, decido lanzar **bloodhound** y obtener información mediante **bloodhound-python**.

Marco el usuario support como owneado y mirando sus atributos descubro que puede cambiarle la contraseña al usuario **audit2020**.

<img src="https://user-images.githubusercontent.com/55555187/162497109-17856b29-4cfa-4b60-a7c7-7727ae84a9ec.PNG" title="" alt="fh6" width="403">

Esto se puede hacer desde rpc sin necesidad de una shell en la máquina víctima, concretamente de dos maneras.

- Usando el comando **net rpc**.
  
  ```
  net rpc passowrd audit2020 -U support -S 10.10.10.192
  ```
  
  ![fh8](https://user-images.githubusercontent.com/55555187/162497115-64d43cc1-1e83-4a1d-99d5-44b5883be72d.PNG)
  
  ![fh9](https://user-images.githubusercontent.com/55555187/162497117-b3759dbc-1f86-445c-b168-105e5c436b44.PNG)

- Usando el propio **rpcclient**.
  
  ```
  seruserinfo2 audit2020 23 'Kripteria10101010'
  ```
  
  ![fh10](https://user-images.githubusercontent.com/55555187/162497120-503f19fa-c6d8-4925-9fd2-b2681593e1e2.PNG)

Viendo el nombre del usuario y sabiendo que existe la carpeta **forensics** en samba, compruebo que tengo acceso a ella y listo su contenido.

<img src="https://user-images.githubusercontent.com/55555187/162497124-b51561dc-5cdf-4690-b020-72c5e4280103.PNG" title="" alt="fh11" width="612">

<img title="" src="https://user-images.githubusercontent.com/55555187/162497126-519cfb81-110b-49f8-bc4d-89a9f378d9da.PNG" alt="fh12" width="613">

En el directorio **memory_analysis** observo que hay un zip del lsass por lo que procedo a descargarmelo y analizarlo en mi máquina.

<img src="https://user-images.githubusercontent.com/55555187/162497129-313a9e78-4e81-4bb5-bd19-a2d6b49e09c8.PNG" title="" alt="fh13" width="610">

Al unzipear el archivo comprimido veo que me devuelve un archivo **lsass.DUMP** que puedo leer con **pypykatz**. Al leerlo obtengo información del usuario **svc_backup** del que ya sabía al haber analizado el servicio rpc y haber realizado un dump del ldap mediante **ldapdomaindump**. En especial, obtengo el **hash NT** que me permitirá hacer **Pass the Hash** para entrar en la máquina mediante **winrm**.

<img src="https://user-images.githubusercontent.com/55555187/162497131-5b1273bb-49a6-4829-9d84-f275c1410f72.PNG" title="" alt="fh14" width="456">

<img src="https://user-images.githubusercontent.com/55555187/162497132-fe5400d2-cd8b-4188-b6ae-5b974cf48c3c.PNG" title="" alt="fh15" width="626">

## Escalada de privilegios

El usuario svc_backup tiene una peculiaridad de la que nos podemos aprovechar a la hora de escalar privilegios. Si vemos sus permisos podemos observar que cuenta con el permiso **SeBackupPrivilege** y **SeRestorePrivilege**. 

<img src="https://user-images.githubusercontent.com/55555187/162497135-bb5897f3-978a-419d-9f1b-953bc002cc27.PNG" title="" alt="fh16" width="586">

Estos privilegios pueden ser abusados para realizar una copia del **NTDS.dit** y **System** en una unidad nueva, poder descargarlos posteriormente a la máquina local y extraer los hashes utilizando **secretsdump** de **impacket**.

Siguiendo los pasos del siguiente articulo: https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960 procedo a dumpear el NTDS.dit y System.

Primero le paso un script a la utilidad **diskshadow** el cual crea una copia de C: y la muestra como E: 

<img src="https://user-images.githubusercontent.com/55555187/162497138-6fd5015e-f333-44a2-af33-f7f7f399b056.PNG" title="" alt="fh17" width="557">

<img src="https://user-images.githubusercontent.com/55555187/162497140-d231bb1f-7736-43c6-82b5-3143e823a53f.PNG" title="" alt="fh18" width="602">

Copio el NTDS.dit de E: en el directorio Temp del disco C: mediante la utilidad **robocopy** y el registro System que se usará para descifrar el NTDS.dit en secretsdump.

<img src="https://user-images.githubusercontent.com/55555187/162498137-57649db6-a2b3-47bb-919a-edc92d6f5fa6.PNG" title="" alt="fh19" width="492">

Me descargo ambos archivos a mi máquina utilizando el comando download de **evil-winrm** y lanzo secretsdump obteniendo todos los hashes del dominio.

<img src="https://user-images.githubusercontent.com/55555187/162497145-69188842-3f92-4f8d-bedf-46a43a4047d4.PNG" title="" alt="fh20" width="618">

Utilizando evil-winrm y Pass the Hash me conecto como Administrator a la máquina.

<img src="https://user-images.githubusercontent.com/55555187/162497147-8631fea8-c34b-43ec-a924-9d339818252a.PNG" title="" alt="fh21" width="637">



*Nota: Con el hash del usuario krbtgt y su SID (que es facilmente obtenible) se puede construir un golden ticket mediante ticketer.py para persistir en el dominio pudiendo crear TGTs para impersonar usuarios.*
