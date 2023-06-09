---
title: Seekolver
published: true
categories: [Tool, Attack-surface mapping]
tags: [Tool, Attack-surface mapping, Python, Personal project]

pin: true
image:
  path: https://github.com/Krypteria/Seekolver/assets/55555187/7432aeb8-bc23-445c-b9af-931bbb4a03ec
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA

---

[Seekolver](https://github.com/Krypteria/Seekolver) is a tool focused on attack-surface mapping. It performs searches for subdomains associated with root domains and root domains associated with organisations using free sources, additionally, it resolves these domains and subdomains in search of HTTP and HTTPS services and then filters the information obtained based on their response.

```
usage: seekolver.py [-h] [-f FILE] [-o OUTPUT] [-te TARGETENTITY] [-cn] [-on] [-t THREADS] [-to TIMEOUT] [-r] [-k] [-v] [-s] [-sc SHOWCODES [SHOWCODES ...]]
                    [-sd SHOWDOMAINS [SHOWDOMAINS ...]]

options:
  -h, --help            show this help message and exit
  -f FILE, --file FILE  file with urls to resolve
  -o OUTPUT, --output OUTPUT
                        name of the output file used for writing the results
  -te TARGETENTITY, --targetEntity TARGETENTITY
                        target entity on which the subdomain search will be applied
  -cn, --commonName     the aplication will use the target entity as common name
  -on, --organisationName
                        the aplication will use the target entity as organisation name
  -t THREADS, --threads THREADS
                        number of threads to be used (default 50)
  -to TIMEOUT, --timeout TIMEOUT
                        timeout value for requests (default 3s)
  -r, --redirect        resolves redirections
  -k, --insecure        Allow insecure server connections
  -v, --verbose         enable verbose output
  -s, --show            displays the information of an output file in colour
  -sc SHOWCODES [SHOWCODES ...], --showCodes SHOWCODES [SHOWCODES ...]
                        filters the show parameter output to certain status codes
  -sd SHOWDOMAINS [SHOWDOMAINS ...], --showDomains SHOWDOMAINS [SHOWDOMAINS ...]
                        filters the show parameter output to certain domains
```

## About the tool

It is not uncommon in a Pentesting / Red Team / Bug Bounty exercise to come across an organisation that has a large number of root domains and these root domains have a countless number of subdomains. Manually sifting through them to see which ones might be of interest often leads to frustration if many of these turn out to be empty or unresolvable domains. With the idea of facilitating pre-filtering and attack surface mapping, Seekolver was born.

Currently there are fantastic tools for attack surface mapping like [Amass](https://github.com/OWASP/Amass) and tools like [httpx](https://github.com/projectdiscovery/httpx) for address resolution so it occurred to me that it could be interesting to centralise both functionalities in the same tool, automating certain processes to obtain results more quickly. 

Currently Seekolver makes use of **Securitytrails**, **Alienvault**, **Askdns**, **Spyonweb**, **Crt.sh** and **Virustotal** to search for subdomains, but more may be added in the future.


## Configuration

To make use of the *Virustotal* and *Securitytrails* services to search for subdomains, it is necessary to have an account in both and create an **apitokens.json** file with the following format in the same directory where the tool is located:


```
{
        "securitytrails":["apikey","<YOUR-API-TOKEN>"],
        "virustotal":["x-apikey","<YOUR-API-TOKEN>"]
}
```

## Example of use

![image](https://user-images.githubusercontent.com/55555187/222470674-1d2501ca-ed38-4a83-b9f8-b0897aaa461b.png)

### Basic usage: resolution of the urls contained in a file

This mode resolves all urls contained in a file passed as input, the contents of the file can be either the full address or just the domain. The only difference is that, if the domain is passed, the tool will resolve using the HTTP and HTTPS protocols whereas if a complete url is passed it will resolve only for the corresponding protocol.

```
https://www.corp.com -> valid
www.corp.com -> valid
```

```
python3 seekolver.py -f recon.txt
```

### Basic usage: subdomain search and resolve over a domain


This mode performs a search of all subdomains belonging to the given domain using multiple public information sources, once it has obtained them, it applies a resolution looking for HTTP or HTTPS services discarding all subdomains that do not resolve.


```
python3 seekolver.py -te www.corp.com -cn
```

### Basic usage: domain search and resolve over a organisation


This mode performs a search of all domains belonging to the given organisation using multiple public information sources, once it has obtained them, it applies a resolution looking for HTTP or HTTPS services discarding all subdomains that do not resolve.


```
python3 seekolver.py -te "CORP SA" -on
```

### Optional arguments


The tool has a number of optional parameters that enhance the basic functionality.

The **-t** parameter allows to select the number of threads the tool will use when resolving addresses, the **-r** parameter tells the tool to follow redirects and display the address in the output file, the **-to** parameter allows you to set a timeout value for requests, the **-v** parameter enables verbose output during the resolution stage to get a better idea of how much time is left to complete the task, and finally the **-o** parameter allows you to choose the name of the output file.


### Format arguments


To facilitate the visualisation of results, the tool has the ability to interpret the information in the output files and apply a series of colours using the **-s** parameter, additionally a filtering based on status codes can be applied while maintaining this format with the **-sc** parameter and a filtering based on domain names can be applied using **-sd**


```
python3 seekolver.py -f available.txt -s -sc 200 301 302 401
```

![image](https://user-images.githubusercontent.com/55555187/222472393-bd2fde0d-8300-4efd-949a-d06369e053fc.png)

## Dependencies

Seekolver requires **python3** and the **bs4** and **requests** python modules to be installed on the system to function properly.

```
pip3 install bs4
pip3 install requests
```