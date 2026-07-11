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
Nmap scan report for makesense.htb (<IPAddress>)
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
wpscan --url https://makesense.htb --disable-tls-checks --enumerate p,u --no-update
```
```
--url https://makesense.htb - Specifies the target WordPress site URL to scan
--disable-tls-checks - Disables SSL/TLS certificate verification (allows scanning sites with self-signed or invalid certificates)
--enumerate p,t - Enumerates (lists and identifies) WordPress components:
p = plugins installed on the site
u = users
--no-update - Skips updating the vulnerability database before scanning (uses the existing local database to save time)
```
[+] Upload directory has listing enabled: https://makesense.htb/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

Found one file only at the path: https://makesense.htb/wp-content/uploads/2026/01/voice-message.wav

And some users.
[+] jake
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] walter
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)


Whisper is a state-of-the-art speech-to-text (automatic speech recognition) model developed by OpenAI. It is widely used to transcribe spoken audio into highly accurate written text. (You can also just slow down the media player speed to 0.5)
```
whisper voice-message.wav --model medium --language English
```
Hey, this is Jake. I'm testing the new feature, and it's exciting. I'm going there.  
Oops. Login.  
Um, Jake?  
Clear.  
L****.  
N****.  
S****.  
F****.  
N****.  
T****.    
T****.  

It seems to be creds:   
Jake:C****L*****N*****S********  

Meanwhile let's search for directories/commonfiles and subdomains:
```
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -u 'https://makesense.htb/FUZZ' -ac
```
```
.git/logs/              [Status: 200, Size: 34914, Words: 5289, Lines: 349, Duration: 276ms]
.gitignore              [Status: 200, Size: 1055, Words: 49, Lines: 90, Duration: 72ms]
cgi-bin/                [Status: 200, Size: 34914, Words: 5289, Lines: 349, Duration: 2199ms]
javascript              [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 105ms]
scripts                 [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 653ms]
wp-admin                [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 539ms]
wp-includes             [Status: 301, Size: 322, Words: 20, Lines: 10, Duration: 416ms]
wp-content              [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 721ms]
xmlrpc.php              [Status: 405, Size: 42, Words: 6, Lines: 1, Duration: 1459ms]
:: Progress: [4750/4750] :: Job [1/1] :: 30 req/sec :: Duration: [0:02:59] :: Errors: 0 ::

```
Let's try those creds in the /wp-admin panel. And it works, but Jake has just a contributor role, can't publish shit.
```
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H 'Host: FUZZ.makesense.htb' -u 'https://makesense.htb/' -ac 

 :: Method           : GET
 :: URL              : https://makesense.htb/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.makesense.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500

fqbSrJhU                [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 149ms]

```
```
echo '<IPAddress>    fqbSrJhU.makesense.htb' | sudo tee -a /etc/hosts
```

Navigating to the Contact Submissions section in the WordPress admin panel, we can see voice recording files submitted by users. To understand how these files are uploaded, we intercept the traffic with Burp Suite while browsing to the subdomain fqbsrjhu.makesense.htb.
In the page source, we find an inline JavaScript block that exposes the AJAX endpoint and a valid nonce:
```
<script id="whisper-wrapper-js-extra">
var webagency_ajax = {"ajax_url":"https://makesense.htb/wp-admin/admin-ajax.php","nonce":"50fbeb8031","theme_url":"https://makesense.htb/wp-content/themes/webagency","site_url":"https://makesense.htb"};
//# sourceURL=whisper-wrapper-js-extra
</script>
```
This reveals that the voice recording feature communicates with admin-ajax.php using the save_voice_raw action. We can now replicate the browser's request directly with curl, bypassing the JavaScript frontend entirely.  

We send a POST request to admin-ajax.php with the save_voice_raw action, attaching the voice recording file found earlier and the nonce extracted from the page source.
```
curl -sk -X POST https://makesense.htb/wp-admin/admin-ajax.php \                                        
-F "action=save_voice_raw" \
-F "nonce=50fbeb8031" \
-F "voice_recording=@voice-message-2.wav"
{"success":true,"data":{"message":"Audio saved, processing started.","post_id":77}}    
```
The server responds with a success message and a post_id, confirming that the endpoint accepts unauthenticated file uploads. The response also hints at a background processing pipeline - "processing started" - suggesting the audio is analyzed server-side after upload.  

```                                                                 
curl -sk "https://makesense.htb/index.php?rest_route=/wp/v2/media" | jq '.[0] | .source_url, .id'
"https://makesense.htb/wp-content/uploads/2026/07/voice-message-2.wav"
78
```
From Contact Submissions section in the admin panel, as jake, we can't do much. Just listing the submitted files and waiting for an admin to approve.
















