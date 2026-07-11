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
