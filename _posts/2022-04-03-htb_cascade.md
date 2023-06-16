---
title: Cascade
published: true
categories: [Active Directory,Medium]
tags: [Active directory explotation, Kerberos, Binary reversing, Recycle Bin abuse]

image:
  path: https://user-images.githubusercontent.com/55555187/161436982-23731c92-e19a-4f7c-bb07-89e8e2a99ae1.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

Cascade es una <font color="#FFC300">máquina Medium</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca explotación de consultas anónimas en ldap, enumeración de samba, password spray en Kerberos, criptografía, reversing de aplicaciones .NET y abuso del grupo AD Recycle Bin.

## Reconocimiento

Una vez que la máquina está activa lanzo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.182
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161436219-32d2a738-d480-440f-9062-e752ad44bf89.png" title="" alt="recon1" width="291">

Lanzo un escaneo con nmap para obtener la versión y servicio de los puertos.

```
nmap -sC -sV -p53,88,135,139,389,445,636,3268,3269,5985,49154,49155,49157,49158,49170 10.10.10.182
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161436220-9a8e7db6-aaba-4217-bba5-9c27da11197b.png" title="" alt="recon2" width="572">

Lo primero que compruebo es si tengo acceso a carpetas compartidas en red a través de **samba** utilizando una null sessión.

```
smbmap 10.10.10.182
```

Pero no parece haber nada.

Como está abierto el servicio **ldap** pruebo a lanzar una consulta anónima para obtener el contexto del dominio con **ldapsearch**.

```
ldapsearch -x -h 10.10.10.182 -s base namingcontexts
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161436200-2c78c2d8-dbe2-47a7-a018-4c24cbbc9325.png" title="" alt="ldap1" width="578">

Pruebo a listar el ldap entero para comprobar si tiene información relevante.

```
ldapsearch -x -h 10.10.10.182 -b "DC=cascade,DC=local"
```

Como veo que me reporta una gran cantidad de output hago 2 cosas:

1- Obtengo una lista de usuarios del dominio.

```
ldapsearch -x -h 10.10.10.182 -b "DC=cascade,DC=local" | grep "userPrincipalName:"
```

<img src="https://user-images.githubusercontent.com/55555187/161436203-f2b8c1cb-baf9-4dbe-91b8-773c151d8269.png" title="" alt="ldap3" width="344">

2- Filtro por palabras clave como *password*,*pwd*,*Password*,*Pwd*,*userPassword* para ver si encuentro alguna contraseña. 

<img src="https://user-images.githubusercontent.com/55555187/161436205-c3a12add-b501-4b45-a675-123d216e9abd.png" title="" alt="ldap4" width="243">

<img src="https://user-images.githubusercontent.com/55555187/161436206-0d16e7df-4dcc-423a-b2ad-3ce45df29104.png" title="" alt="ldap5" width="341">

Viendo la contraseña y viendo que uno de los usuarios se llama **r.thompson** decido comprobar si pertece a él con **crackmapexec**.

```
crackmapexec smb 10.10.10.182 -u r.thompson -p rY4n5eva
```

![ldap6](https://user-images.githubusercontent.com/55555187/161436207-daefbf96-b374-43fd-a76d-d264a6cb18d3.png)

Como pertenece a él, compruebo si r.thompson está en el grupo **Remote Management Users** y puede acceder a la máquina mediante **winrm** pero por desgracia no se da el caso.

```
crackmapexec winrm 10.10.10.182 -u r.thompson -p rY4n5eva
```

## Obtención de acceso

Como no pertenece al grupo, decido ver si con sus credenciales puedo acceder a alguna carpeta compartida por samba.

```
smbmap -H 10.10.10.182 -u r.thompson -p rY4n5eva
```

<img src="https://user-images.githubusercontent.com/55555187/161436224-15fb5bd6-2636-4687-adcc-ef779e29d7ee.png" title="" alt="smb0" width="606">

Listo recursivamente el contenido del directorio **Data**.

<img src="https://user-images.githubusercontent.com/55555187/161436222-32fec764-5a8f-4796-962f-87f57a665126.png" title="" alt="smb0 5" width="575">

Me descargo todos los archivos y los inspecciono en mi máquina local. En los archivos encontramos lo siguiente:

1- en **Meeting_Notes_June_2018.html** encontramos una referencia a la creación de un usuario **TempAdmin** que comparte la misma contraseña que **Administrator**.

![smb1](https://user-images.githubusercontent.com/55555187/161436225-97f1cf24-55b9-4688-a076-9305f5177f46.png)

2- Veo que el usuario **TempAdmin** ha sido enviado a la papelera de reciclaje por el usuario **arksvc** en el archivo **ArkAdRecycleBin.log**.

<img src="https://user-images.githubusercontent.com/55555187/161436451-9258b828-5073-42d4-8157-ce4c7b918681.PNG" title="" alt="arkpapelera" width="638">

3- En **VNC Install.reg** encontramos una contraseña en hexadecimal. Si la convertimos a texto plano mediante *xxd -r -p* podemos ver que está cifrada.

<img src="https://user-images.githubusercontent.com/55555187/161436226-9dc179e2-3308-4e5c-991b-8ddb71774652.png" title="" alt="smb2" width="355">

Búscando en internet descubro el siguiente recurso: [GitHub - frizb/PasswordDecrypts: Handy Stored Password Decryption Techniques](https://github.com/frizb/PasswordDecrypts) en el que se explica que el servicio TightVNC utiliza un método muy concreto para cifrar la contraseña. Siguiendo los pasos que se exponen logro descifrarla:

<img src="https://user-images.githubusercontent.com/55555187/161436228-9375e484-d6d7-4285-a0bd-176833b51c4b.png" title="" alt="smb4" width="633">

Como la contraseña se encontraba en el directorio de **s.smith** asumo que es suya pero aún así compruebo su validez con crackmapexec y además si este pertenece al grupo **Remote Management Users**.

<img src="https://user-images.githubusercontent.com/55555187/161436229-87e1cdb9-0e31-409c-abc9-67b5e6a0b5d8.png" title="" alt="smb5" width="627">

Como pertenece, me conecto a la máquina utilizando **evil-winrm**.

Al listar la información del usuario veo que el usuario pertenece al grupo **Audit Share** y antes había visto una carpeta **Audit$** en el servicio Samba lo que me hace pensar que **s.smith** puede tener acceso.

<img src="https://user-images.githubusercontent.com/55555187/161436209-67f0cdb1-6de4-4ec8-93f2-88be74f7913d.png" title="" alt="privesc1" width="487">

```
smbmap -H 10.10.10.182 -u s.smith -p sT333ve2
```

<img src="https://user-images.githubusercontent.com/55555187/161436803-0a441e08-aa83-4c78-b011-eead5741f110.png" title="" alt="smb7" width="587">

Listo el contenido recursivamente.

<img src="https://user-images.githubusercontent.com/55555187/161436230-fe4fe12c-7287-43bd-b4dd-9f561633ceb5.png" title="" alt="smb6" width="597">

Veo muchos archivos interesantes por lo que decido bajarmelos todos y analizarlos en mi máquina. Lo primero que hago es abrir la base de datos mediante sqlite3. Observo 3 tablas, la primera y la tercera no contienen nada relevante pero la segunda nos da otras credenciales.

<img src="https://user-images.githubusercontent.com/55555187/161436211-d078c542-69c9-4d38-841d-4b4531a92b0b.png" title="" alt="privesc2" width="519">

Al igual que con las anteriores, si desciframos el base64 de la contraseña vemos que se encuentra cifrada.

Además de la base de datos, también tenemos el fichero **CascCrypto.dll** que es una librería dinámica que potencialmente ejecutará operaciones criptográficas y el archivo **CascAudit.exe** que es altamente probable que use el dll.

Lo que hago es, utilizando **dnSpy.exe** a través de **wine64** descompilar el ejecutable para indagar en el código. En la función **DecryptString** encontramos el vector IV, el key size y el algoritmo utilizado para cifrar. En la función Main encontramos la clave secreta utilizada para cifrar la contraseña.

<img src="https://user-images.githubusercontent.com/55555187/161436214-147aa16c-808d-4a3f-8563-ba0f3dbda3a8.png" title="" alt="privesc4" width="585">

<img src="https://user-images.githubusercontent.com/55555187/161436215-1302279e-5351-467f-8586-a8cd59a6f98c.png" title="" alt="privesc5" width="573">

Con todo eso, podemos ir a alguna web como [Online AES Encryption and Decryption Tool](https://www.javainuse.com/aesgenerator) para descifrar la contraseña obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161436449-fb63ac3f-eb30-401d-bd8d-45578a586b28.PNG" title="" alt="passark" width="152">

Valido las credenciales con crackmapexec y compruebo si arksvc pertenece al grupo Remote Management Users.

<img src="https://user-images.githubusercontent.com/55555187/161436941-e13e00bb-69b5-45dc-95ab-01cd4b1822d9.PNG" title="" alt="arkcrackmap" width="660">

## Escalada de privilegios

Una vez conectado como **arksvc**, listo la información del usuario.

<img src="https://user-images.githubusercontent.com/55555187/161436450-727a45fc-2e5a-44b3-8903-d42fa041a20b.PNG" title="" alt="userark" width="481">

Veo que pertenezco al grupo **AD Recycle Bin** e inmediatamente recuerdo que el usuario **TempAdmin** fue enviado a la papelera. 

Investigando maneras de escalar privilegios con esta información encuentro el siguiente articulo: [Domain Privilege Escalation - Offsec Journey](https://notes.offsec-journey.com/active-directory/domain-privilege-escalation) donde se muestra una vía para recuperar la información del usuario eliminado.

Lanzando el comando consigo la información del TempAdmin incluida su contraseña en base64 (que como sabemos, es la misma que la de Administrator).

<img src="https://user-images.githubusercontent.com/55555187/161436216-99d2a9db-ac38-40a1-a4aa-633bf442c8bb.png" title="" alt="privesc6" width="596">

Valido las credenciales y accedo al sistema como Administrator mediante winrm.

<img src="https://user-images.githubusercontent.com/55555187/161436217-334fecb0-05da-49c8-b088-d7c9674cdf6f.png" title="" alt="privesc7" width="602">
