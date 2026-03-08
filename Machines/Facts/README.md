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
<img src="images/nmap.png" width="700">


















