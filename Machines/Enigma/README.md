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



## RECON

# Nmap 7.99 scan initiated Mon Jun 28 15:00:01 2026 as: /usr/lib/nmap/nmap -sC -sV -p- -oN nmap_enigma.txt <IP ADDRESS>
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
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      39550/udp6  mountd
|   100005  1,2,3      48374/udp   mountd
|   100005  1,2,3      52727/tcp6  mountd
|   100005  1,2,3      57219/tcp   mountd
|   100021  1,3,4      33974/udp   nlockmgr
|   100021  1,3,4      40183/tcp   nlockmgr
|   100021  1,3,4      40459/tcp6  nlockmgr
|   100021  1,3,4      47490/udp6  nlockmgr
|   100024  1          39479/tcp   status
|   100024  1          39501/udp   status
|   100024  1          41824/udp6  status
|   100024  1          56609/tcp6  status
|   100227  3           2049/tcp   nfs_acl
|_  100227  3           2049/tcp6  nfs_acl
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
> echo '<IP ADDRESS>    enigma.htb' | sudo tee -a /etc/hosts
```

