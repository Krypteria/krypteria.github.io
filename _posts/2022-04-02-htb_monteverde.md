---
title: Monteverde
published: true
categories: [Active Directory,Medium] 
tags: [Active directory explotation, Privilege abuse]
author: Alejandro Rivera
image:
  path: https://user-images.githubusercontent.com/55555187/161390929-cf17b90b-94c1-4cbf-a973-c15301dbb52d.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

Monteverde es una <font color="#FFC300">máquina Medium</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca explotación de consultas anónimas en ldap, password spray en Kerberos, enumeración del servicio samba y explotación del servicio Azure AD Connect.

## Reconocimiento

Una vez que veo que la máquina es accesible, lanzo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.172
```

Obteniendo:

<img title="" src="https://user-images.githubusercontent.com/55555187/161388387-c5868f51-f8cd-4bbc-ae69-c0b3d0ee1d7c.png" alt="recon1" width="273">

Posteriormente, lanzo un escaneo con nmap para definir la versión y servicio que corren en esos puertos.

```
nmap -sC -sV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49693 10.10.10.172
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161388388-a52e0ff1-6472-432c-9e06-e344c8922774.png" title="" alt="recon2" width="627">

Utilizando smbmap intento listar carpetas compartidas en red pero no encuentro nada.

```
smbmap -H 10.10.10.172
```

Utilizando ldapsearch lanzo una consulta anónima a ldap para intentar conseguir el contexto bajo el que corre.

```
ldapsearch -x -h 10.10.10.172 -s base namingcontexts
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161388382-7768f0b5-27d5-45d9-85e8-a3055f3a9476.png" title="" alt="ldap1" width="565">

Me dumpeo todo el ldap utilizando el siguiente comando:

```
ldapsearch -x -h 10.10.10.172 -b "DC=megabank,DC=local"
```

Además de eso, realizo una consulta para obtener todos los usuarios del dominio.

```
ldapsearch -x -h 10.10.10.172 -b "DC=megabank,DC=local" "(objectClass=person)" | grep "userPrincipalName:"
```

<img src="https://user-images.githubusercontent.com/55555187/161388384-399c6c41-0625-4761-bda7-aaaacbbb1c54.png" title="" alt="ldap2" width="645">

Pruebo a búscas contraseñas filtrando el ldap mediante "pwd", "password", "Password", "pass", "userPassword" pero no me reporta nada interesante.

## Obtención de acceso

Teniendo la lista de usuarios pruebo a lanzar un password spray attack utilizando crackmapexec. Como lista de contraseñas siempre me gusta primero utilizar los propios nombres de los usuarios y si no encuentro nada lanzar alguna lista de SecLists.

```
crackmapexec smb 10.10.10.172 -u users.txt -p users.txt
```

Obtenemos una credencial válida:

![ldap3](https://user-images.githubusercontent.com/55555187/161388385-20da1e95-b311-4029-92e9-72d470bcdb7b.png)

Compruebo si puedo acceder mediante winrm al sistema con esta credencial mediante crackmapexec pero SABatchJobs no parece pertener al grupo remote management users.

Pruebo a listar carpetas compartidas por samba con las credenciales obtenidas:

```
smbmap -H 10.10.10.172 -u SABatchJobs -p SABatchJobs
```

![smbmap1](https://user-images.githubusercontent.com/55555187/161388377-10f213e5-8afd-4940-b7df-df4791d214c0.png)

Investigando la carpeta users$ puedo ver el archivo azure.xml en el directorio de mhope con credenciales.

<img src="https://user-images.githubusercontent.com/55555187/161388379-7bf160bf-197a-4047-a7fd-b1bed49cab71.png" title="" alt="smbmap2" width="627">

<img src="https://user-images.githubusercontent.com/55555187/161388380-14ace87c-0fb6-4bd1-b03e-46fb9b4e3064.png" title="" alt="smbmap3" width="631">

Valido las credenciales con crackmapexec y miro si puedo conectarme a la máquina por winrm con las nuevas credenciales.

<img src="https://user-images.githubusercontent.com/55555187/161388381-6eedd70a-ce0f-4ca1-9558-22cdadf15360.png" title="" alt="smbmap4" width="633">

El usuario mhope pertenece al grupo Remote Management Users. Me conecto al sistema y haciendo una enumeración muy simple veo que mhope es miembro de azure admins lo cual puedo aprovechar para elevar privilegios.

## Escalada de privilegios

A la vista del archivo azure.xml, decido investigar sobre los usos de azure en directorio activo. Descubro que Azure es utilizado para la sincronización de contraseñas utilizando ADSync. Investigando un poco más descubro el [articulo de XPN](https://blog.xpnsec.com/azuread-connect-for-redteam/) que habla en detalle sobre como abusar este privilegio si contamos con un usuario con los permisos necesarios.

Por suerte, mhope pertenece al grupo Azure Admins lo cual nos da vía libre para poder realizar el exploit.

<img src="https://user-images.githubusercontent.com/55555187/161390813-1ab86de9-11b8-4a26-8466-98b84c0d18c0.png" title="" alt="privesc1" width="501">

Al parecer, Azure por lo general instala una base de datos llamada **localdb** corriendo bajo el servicio **ADSync**.

La vulnerabilidad de esta implementación reside en que la contraseña del usuario administrador se encuentra cifrada en el archivo private_configuration_xml que se encuentra en la base de datos.

Dicho archivo se puede descifrar obteniendo un *keyset_id*,*instance_id* y un valor *entropy* que también se encuentran almacenados en la base de datos.

Como mhope tiene permisos de Azure Administrator, podemos lanzar peticiones a la base de datos para obtener todos esos valores y crackear el cifrado obteniendo la contraseña del administrador.

XPN incluye el código necesario en su articulo para realizar el ataque.

Al probar el script vemos que da error al abrir la base de datos:

![privesc3](https://user-images.githubusercontent.com/55555187/161390806-eb876560-6a4b-4700-b82d-3548e07c14aa.png)

Búscando información sobre vias alternativas de abrir la conexión doy con este [articulo de sqlshack](https://www.sqlshack.com/es/conectando-powershell-a-un-servidor-sql-server-2/) en el que explican vias alternativas.

<img src="https://user-images.githubusercontent.com/55555187/161390814-fb526caa-aa05-4ed4-9ff7-1b69bf8bbacd.png" title="" alt="privesc2" width="569">

Reemplazo la siguiente línea:

```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Data Source=(localdb)\.\ADSync;Initial Catalog=ADSync"
```

Con esta otra:

```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Data Source=localhost;Integrated Security=true;Initial Catalog=ADSync"
```

Logrando la ejecución y las credenciales de Administrator:

![privesc4](https://user-images.githubusercontent.com/55555187/161390809-543041a9-f515-41dc-b0f4-b08cbdf8eea5.png)

Compruebo la validez de las credenciales con crackmapexec y si puedo conectarme mediante winRM.

<img src="https://user-images.githubusercontent.com/55555187/161390810-fabdae03-a8d7-4608-ae57-5167a18ffc08.png" title="" alt="privesc5" width="553">

Me conecto a la máquina como Administrator.

<img src="https://user-images.githubusercontent.com/55555187/161390811-f55c83c8-5c50-4347-a953-e5c3e40a8c8c.png" title="" alt="privesc6" width="549">
