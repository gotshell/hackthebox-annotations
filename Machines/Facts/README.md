# Hack The Box — Facts Walkthrough

## Machine Information

| Field | Value |
|------|------|
| Name | <Machine Name> |
| Platform | Hack The Box |
| OS | <Linux / Windows> |
| Difficulty | <Easy / Medium / Hard / Insane> |
| Release Date | <date> |
| Target IP | <target ip> |

## Overview

Brief description of the machine and the main techniques used to solve it.

**Skills / Techniques**

- Enumeration
- Exploitation
- Privilege Escalation
- Web / Burp / AWS s3 / Cracking

## Table of Contents

- Reconnaissance
- Enumeration
- Exploitation
- Privilege Escalation
- Root / Administrator Access
- Lessons Learned

## Recon

Initial information gathering and port scanning.

```
> nmap -sT -T4 -sV -p- --script=http-headers {IP_Addr}
```
### Findings:
- 22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
- 80/tcp    open  http    nginx 1.26.3 (Ubuntu)
- 54321/tcp open  http    Golang net/http server
  http-server-header: MinIO
  // Port 54321/TCP is open running an HTTP service built with Go, and the HTTP header reveals the server is MinIO, an S3‑compatible object storage service.
<img src="images/nmap.png" width="500">



















