---
title: Shocker
published: true
categories: [Linux, Easy]
tags: [Web explotation, Shellsock, SUDO abuse]

image:
  path: https://user-images.githubusercontent.com/55555187/191332797-a3d8413c-28ed-42dd-9911-4f57ba3b5d72.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Shocker es una <font color="#98E256">máquina Easy</font> de temática <font color="#D35400">Linux</font> en la que se toca enumeración y fuzzing de servicios web, shellsock attack y Permission abuse sudo sobre el binario perl.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.10.56
```

<img src="https://user-images.githubusercontent.com/55555187/188194143-041531d0-9289-4a55-b569-6bf8508afc04.png" title="" alt="image" width="437">

```bash
nmap -sVC -p80,2222 -oN portSV 10.10.10.56
```

![image](https://user-images.githubusercontent.com/55555187/188194594-c2e88d25-e3be-4ab7-be01-25cb7e71e66f.png)

Revisando la web del puerto 80 veo que es la página de inicio de Apache, haciendo fuzzing mediante *wfuzz* y la wordlist *directory-list-2.3-medium* no obtengo nada relevante por lo que decido cambiar de herramienta y de wordlist.

Pruebo con *gobuster* y la wordlist *common.txt* y obtengo una serie de directorios interesantes:

![image](https://user-images.githubusercontent.com/55555187/188197609-6b96e3e4-7d8c-45a1-950c-af77dfa83e19.png)

Realizo una búsqueda dentro del directorio */cgi-bin/* por archivos *.sh* y *.cgi* encontrando un script *user.sh* que tiene información sobre la sobrecarga del sistema:

<img src="https://user-images.githubusercontent.com/55555187/188198542-230aca8f-6d60-44d8-bb4d-4b4a05390942.png" title="" alt="image" width="517">

Inmediatamente lo primero que se me ocurre es testear un *shellshock attack*, para ello lo que hago es inyectar en el User-Agent haciendo uso del comando echo para ver como se comporta el servidor.

Como veo que el servidor responde con un código 500 puedo intuir que estoy logrando ejecutar algo por debajo pero que el formato quizás no es el adecuado:

<img src="https://user-images.githubusercontent.com/55555187/188199728-68295416-76ac-4141-9bfd-6c3dc3316aa5.png" title="" alt="image" width="379">

Pruebo a ejecutar un* curl* contra un servidor levantado en mi máquina especificando que quiero que se ejecute el comando sobre una bash y aportando el path absoluto de esta:

![image](https://user-images.githubusercontent.com/55555187/188199863-dd16a51d-af84-4b0c-bb78-c3eff5dcfe9c.png)

Recibo una petición GET por lo que puedo asegurar que tengo *RCE* en el servidor.

Me entablo una *reverse shell* haciendo uso de un payload en base64:

#### ![image](https://user-images.githubusercontent.com/55555187/188200333-b419394f-e951-4ab6-9dae-132a37d678b7.png)

Como ya he explicado en varios writeups, la idea es simple:

1. Se realiza el echo con la reverse shell encodeada en base64

2. Se pipea y decodea usando *base64 -d *

3. Se pipea el payload decodeado en una bash para que se interprete

## Escalada de privilegios

Una vez dentro, realizo un reconocimiento básico y observo que el usuario *shelly* puede ejecutar *perl* con permisos de superusuario por lo que la escalada es trivial.

<img src="https://user-images.githubusercontent.com/55555187/188200501-dee8bcfe-d00b-48ad-98fe-ee0e4f41e847.png" title="" alt="image" width="563">

**Nota:** haciendo uso de [gtfobins](https://gtfobins.github.io/gtfobins/perl/) se pueden ver las formas de elevar privilegios dependiendo de los permisos que tengamos sobre determinados binarios del sistema.

En este caso es tan sencillo como ejecutar lo siguiente:

![image](https://user-images.githubusercontent.com/55555187/188200573-7a8d1cc2-3784-40b4-b603-75afcd778edc.png)

<img src="https://user-images.githubusercontent.com/55555187/188200694-b8dc5cdf-3112-4ce7-a3bf-577be30355a9.png" title="" alt="image" width="299">
