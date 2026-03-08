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

## Enumeration

### WEB APP Discovery
```
> ffuf -w '/path/to/wordlist/SecLists/Discovery/Web-Content/common.txt' -u 'http://facts.htb/FUZZ' -fw 1328
```

### Findings:

- /admin: The CMS dashboard login.
- /search: Search functionality.
- /ajax: Potential API endpoints for dynamic content.

* Creates an account on the /admin panel * 

### Identifying the IDOR:

After logging in, we access the user profile page. The interface displays a Role dropdown, but it is disabled (disabled="disabled"). By inspecting the page source, we find the following HTML:
```html
<select name="user[role]" id="user_role" disabled="disabled">
    <option value="admin">Administrator</option>
    <option selected="selected" value="client">Client</option>
</select>
```
Although the disabled attribute can be removed using the browser’s developer tools and a request can be crafted to change the role via /admin/users/[ID], the application enforces server-side validation that prevents this direct modification.

### Exploiting the AJAX Password Endpoint: 

The vulnerability is located in the “Change Password” feature. When this action is triggered, the application sends an AJAX request to /admin/users/[ID]/updated_ajax.
The backend, built with Ruby on Rails, likely relies on Strong Parameters, but in this specific password‑update endpoint the allowed parameters are not strictly enforced. This makes it possible to perform parameter smuggling.
By intercepting the password update request in Burp Suite, an additional parameter can be injected into the POST body:

<img src="images/admin_exploit.png" width="500">












