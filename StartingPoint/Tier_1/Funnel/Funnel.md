# HTB Starting Point - Tier1 - Funnel 
### How many TCP ports are open?
  `nmap -sT -p- -T5 <target-ip>`  
  PORT   STATE SERVICE  
  21/tcp open  ftp  
  22/tcp open  ssh  
> 2

### What is the name of the directory that is available on the FTP server?
  `ftp anonymous@<target_ip>`  
  Connected to 10.129.181.13.  
  220 (vsFTPd 3.0.3)  
  331 Please specify the password.  
  Password:   
  230 Login successful.  
  Remote system type is UNIX.  
  Using binary mode to transfer files.  
  ftp> `ls`  
  229 Entering Extended Passive Mode (|||27618|)  
  150 Here comes the directory listing.
> drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup

  226 Directory send OK.    
  ftp> `cd mail_backup`  
  250 Directory successfully changed.  
  ftp> `ls`  
  229 Entering Extended Passive Mode (|||35323|)
  150 Here comes the directory listing.
> -rw-r--r--    1 ftp      ftp         58899 Nov 28  2022 password_policy.pdf  
> -rw-r--r--    1 ftp      ftp           713 Nov 28  2022 welcome_28112022

  226 Directory send OK.
  ftp> `mget password_policy.pdf welcome_28112022` // to download  

# What is the default account password that every new member on the "Funnel" team should change as soon as possible?

password_policy.pdf:

![screenshot](./img/password_policy.png)

> funnel123#!#

# Which user has not changed their default password yet?

welcome_28112022: 

Frome: root@funnel.htb  
To: optimus@funnel.htb albert@funnel.htb andreas@funnel.htb christine@funnel.htb maria@funnel.htb  
Subject:Welcome to the team!  

Hello everyone,  
We would like to welcome you to our team.   
We think you’ll be a great asset to the "Funnel" team and want to make sure you get settled in as smoothly as   possible.  
We have set up your accounts that you will need to access our internal infrastracture. Please, read through the   attached password policy with extreme care.  
All the steps mentioned there should be completed as soon as possible. If you have any questions or concerns   feel free to reach directly to your manager.   
We hope that you will have an amazing time with us,  
The funnel team.   

`ftp christine@<target_ip>` 

331 Please specify the password.  
Password:   
230 Login successful.  
Remote system type is UNIX.  
Using binary mode to transfer files.  
ftp> `ls`
229 Entering Extended Passive Mode (|||11719|)
150 Here comes the directory listing.
226 Directory send OK.

`ssh christine@<target_ip>`

christine@funnel:~$ `ss -tl`
State      Recv-Q      Send-Q      Local Address:Port      Peer Address:Port      Process  
LISTEN      0          4096         127.0.0.1:39181            0.0.0.0:*  
LISTEN      0          4096         127.0.0.53%lo:domain       0.0.0.0:*  
LISTEN      0          128          0.0.0.0:ssh                0.0.0.0:*  
LISTEN      0          4096         127.0.0.1:postgresql       0.0.0.0:*  
LISTEN      0          32           *:ftp                      *:*  
LISTEN      0          128          [::]:ssh                   [::]:*  
