---
title: Active
published: true
categories: [Active Directory,Easy]
tags: [Active directory explotation, cpassword]
author: Alejandro Rivera
image:
  path: https://user-images.githubusercontent.com/55555187/161146970-4648c341-08a0-4114-a625-50470cdfe68f.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Active es una <font color="#98E256">máquina Easy</font> de temática <font color="#8E44AD">Active Directory</font> en la que se toca enumeración del servicio samba, explotación de cpassword con gpp-decrypt y Kerberoasting en Kerberos. 

## Reconocimiento

Una vez que veo que la máquina está operativa, lanzo un escaneo de puertos con nmap.

```
nmap -p- --open -sS --min-rate 5000 -Pn -n 10.10.10.100
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161145961-3146c3f2-e2f6-4597-a250-2505074fb3ea.png" title="" alt="recon1" width="296">

Lanzo otro escaneo para detectar la versión y servicio de los puertos.

```
nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152,49153,49154,49155,49157,49158,49165,49168,49169 10.10.10.100
```

Obteniendo:

<img src="https://user-images.githubusercontent.com/55555187/161145966-e3f4f979-2c1f-495a-8900-93ae72542655.png" title="" alt="recon2" width="610">

Usando **smbmap** listo los recursos compartido mediante el servicio samba. Encuentro un recurso llamado Replication que parece interesante. Haciendo uso de la opción -R de smbmap listo todo el contenido del directorio.

```
smbmap -H 10.10.10.100 -R Replication
```

Entre toda la cantidad de output puedo ver un directorio Group que me llama la atención.

<img src="https://user-images.githubusercontent.com/55555187/161145970-f1f246bb-ecb6-4fb2-9fbe-11f994b30a3d.png" title="" alt="recon3" width="579">

![recon4](https://user-images.githubusercontent.com/55555187/161145971-1e58613c-0fa8-4ac6-a2d0-e1d3be51acff.png)

Haciendo uso de **smbclient** accedo al directorio y descargo el archivo group.xml de su interior y listo su contenido con **xmllint --format**.

<img src="https://user-images.githubusercontent.com/55555187/161146346-0e40bf82-04d4-408d-9ef6-dd5cadd05736.png" title="" alt="recon12" width="670">

Como se puede ver, contiene información sobre el usuario **SVC_TGS**, en concreto, me llama la atención el campo *cpassword*.

## Obtención de acceso y escalada de privilegios

Investigando un poco descubro que las cpassword eran contraseñas que los administradores podían poder a través de políticas de grupo. Esto se bloqueó para nuevas políticas en 2014 con el parche (MS14-025).

Este tipo de cadenas se pueden descifrar facilmente con la utilidad **gpp-decrypt** obteniendo en este caso la contraseña **GPPstillStandingStrong2k18**.

Valido las credenciales con crackmapexec.

![recon5](https://user-images.githubusercontent.com/55555187/161145973-8d56ccba-6790-4f9c-b78c-b9328494965c.png)

Viendo que no puedo entrar por winRM ya que no está el servicio habilitado, puedo a ver si con estas nuevas credenciales tengo acceso a algún directorio más del servicio samba, en este caso puedo acceder a SYSVOL y a Users.

<img src="https://user-images.githubusercontent.com/55555187/161145939-34aea3f7-dc59-4aa0-a146-c59d62c713e2.png" title="" alt="recon6" width="641">

Desde el directorio Users puedo enumerar la máquina pero no hay posibles vias de escalar privilegios.

<img src="https://user-images.githubusercontent.com/55555187/161145944-67a751fd-2a42-46f3-9faa-a68c106870af.png" title="" alt="recon7" width="562">

Intento loguearme en el servicio rpc como el usuario svc_tgs para poder enumerar los usuarios y grupos de la máquina de forma cómoda.

```
rpcclient 10.10.10.100 -U svc_tgs 
```

Enumero los usuarios.

```
enumdomusers
```

![recon8](https://user-images.githubusercontent.com/55555187/161145949-91db75f9-a887-4feb-9bad-33f3885bc3a0.png)

El dominio no contiene otros usuarios interesantes.

Como tengo credenciales, pruebo a lanzar un Kerberoast attack con el script **GetUserSPN.py** de impacket para intentar obtener tickets TGS que posteriormente pueda crackear para obtener contraseñas.

```
GetUserSPN.py -request active.htb/svc_tgs -dc-ip 10.10.10.100
```

Obtengo un TGS perteneciente a Administrator.

<img src="https://user-images.githubusercontent.com/55555187/161145952-f0b42f4c-a4e8-4233-ae60-3c1230d402d0.png" title="" alt="recon9" width="572">

Crackeo la contraseña con John, valido las credenciales con crackmapexec, compruebo si puedo conectarme mediante winRM y como veo que no es posible, me conecto mediante **psexec**.

![recon10](https://user-images.githubusercontent.com/55555187/161145955-3168a941-8685-4be0-9755-24fb93ece50c.png)

<img src="https://user-images.githubusercontent.com/55555187/161145958-91d093be-25f1-4763-96e4-a04499392076.png" title="" alt="recon11" width="640">
