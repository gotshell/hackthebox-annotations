# Hack The Box - Enigma Walkthrough

## Machine Information
| Field | Value |
|------|------|
| Name | MakeSense |
| Platform | Hack The Box |
| OS | Linux |
| Difficulty | Medium |
| Release Date | 4th July, 2026 |

## Overview
MakeSense is a Linux machine that starts with a WordPress site where a voice recording feature exposes a hardcoded AES-GCM encryption key in client-side JavaScript, enabling payload forgery to deliver stored XSS against an admin bot reviewer. XSS is chained with CSRF to create a new administrator account, granting access to the theme editor for RCE as www-data. Credentials found in wp-config.php allow SSH access as a local user, and privilege escalation is achieved by abusing a root-running OCR service (Tesseract) that writes attacker-controlled output to arbitrary PHP files.

## Skills / Techniques
- Subdomain Enumeration
- Client-Side Source Code Analysis
- AES-GCM Payload Forgery
- Stored XSS via Encrypted Voice Transcription
- CSRF-based Admin Account Creation
- WordPress Theme Editor RCE
- Reverse Shell
- Database Credential Discovery (wp-config.php)
- SSH Access & Credential Reuse
- SSH Local Port Forwarding
- OCR Engine Abuse (Tesseract)
- Image Crafting for OCR Exploitation
- Arbitrary File Write via Root Service
- Privilege Escalation to Root

## Table of Contents
1. Reconnaissance
2. Subdomain Discovery
3. WordPress Enumeration
4. Voice Recording Feature Analysis
5. AES-GCM Key Extraction & Payload Forgery
6. Stored XSS Delivery & Admin Bot Exploitation
7. CSRF-based Administrator Account Creation
8. WordPress Theme Editor RCE
9. Foothold as www-data
10. Credential Discovery (wp-config.php)
11. Lateral Movement to Walter (SSH)
12. Internal Service Discovery (Port 8001)
13. SSH Local Port Forwarding
14. OCR Service Abuse (Tesseract + PHP File Write)
15. Root Access

## RECON
```
sudo nmap -sV -p- <IPAddress>                       
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-10 11:51 +0200
Nmap scan report for makesense.htb (10.129.28.244)
Host is up (0.087s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE    SERVICE     VERSION
22/tcp   open     ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp   filtered http
443/tcp  open     ssl/http    Apache httpd 2.4.58 ((Ubuntu))
8001/tcp filtered vcom-tunnel
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 221.29 seconds
```
Nice, small attacking surface. Let's dive that webapp.
```
echo '<IPAddress>    makesense.htb' | sudo tee -a /etc/hosts
```
A WordPress website, use wpscan to find common vulns/misconfig.  
```
wpscan --url https://makesense.htb --disable-tls-checks --enumerate p,t --no-update
```
```
--url https://makesense.htb - Specifies the target WordPress site URL to scan
--disable-tls-checks - Disables SSL/TLS certificate verification (allows scanning sites with self-signed or invalid certificates)
--enumerate p,t - Enumerates (lists and identifies) WordPress components:
p = plugins installed on the site
t = themes installed on the site
--no-update - Skips updating the vulnerability database before scanning (uses the existing local database to save time)
```

