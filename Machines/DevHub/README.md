# Hack The Box — Facts Walkthrough

## Machine Information
| Field | Value |
|------|------|
| Name | Facts |
| Platform | Hack The Box |
| OS | Linux |
| Difficulty | Easy |
| Release Date | 31st January, 2026 |

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
> nmap -sV -T4 -p- 10.129.8.222  
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-05 06:13 -0400
Nmap scan report for devhub.htb (10.129.8.222)
Host is up (0.031s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
6274/tcp open  unknown  
