---
tags:
  - Linux
  - HTTP
  - UDP
  - Weak credentials
  - Sudoers
---

... is a easy HTB machine which hides information in a UDP service, which allows you to identify an application running on a `http` service. There, the default credentials give you access to the `ssh` password of a user. For privilege escalation, a `ssh`-alternative server can be started and connected to as `root`!

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
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 48:b0:d2:c7:29:26:ae:3d:fb:b7:6b:0f:f5:4d:2a:ea (ECDSA)
|_  256 cb:61:64:b8:1b:1b:b5:ba:b8:45:86:c5:16:bb:e2:a2 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```
I visit the web-page on `http://$IP` and it displays (like hinted in by the `nmap` script `http-title`) the `Apache2 Default Page` indicating a correct installation. As this does not provide any possibilities of interaction, i use `ffuf` for a forceful browsing attack to find other endpoints:
```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -u http://$IP/FUZZ
```
Sadly, this did not find any endpoints.

This is why i went back to the `nmap` scan and tried scanning a wider range of ports (not only the top 1000):
```bash
sudo nmap --top-ports 10000 $IP
```
I intentionally left out the flags `-sCV` to find out if there are any open ports. But sadly, even after increasing the number, it did not find anything.

That is why i resorted to `nmap`'s `UDP` scan. `UDP` is a protocol similar to `TCP`, but it does not use the `3-way-handshake`. This makes it a lot quicker, but there is no possibility of making sure the data reaches the target, nor is there a way of telling that an open port actually exists. This reason makes `UDP` port-scans extremely slow. I use a very small range of `UDP` ports in the scan:
```bash
sudo nmap -sU --top-ports 25 $IP
```
One port stands out as `open`, which is `UDP port 161`. That is typically used for `snmp`.

### Initial Exploitation
After researching `snmp` (`Simple Network Management Protocol`), it turns out that this protocol is used for network monitoring. It exposes variables which can be remotely queried and manipulated by applications.

I can use the tool `snmp-check $IP` to find out what variables this system exposes. The output tells us this:
```bash
Host IP address               : 10.10.10.10
Hostname                      : UnDerPass.htb is the only daloradius server in the basin!
Description                   : Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64
Contact                       : steve@underpass.htb
Location                      : Nevada, U.S.A. but not Vegas
Uptime snmp                   : 00:14:05.90
Uptime system                 : 00:13:52.11
```
As i did not know what `daloradius` stands for, i googled it. It turns out that it is a web-based management application for `ISP` deployments.

The [daloradius wiki](https://github.com/lirantal/daloradius/wiki/Installing-daloRADIUS) tells me that the default credentials are `administrator:radius` (at the bottom). I further look into the `daloradius` github by specifically searching the repository for `path:**/login.php`. This reveals the endpoints `/daloradius/app/users/login.php` and `/daloradius/app/operators/login.php`. I am able to log into `http://$IP/daloradius/app/operators/login.php` with the default credentials `administrator:radius`! 

After logging in i see a `Users` tab, where it indicates that there is one entry. The users list shows the username `svcMosh` with an `MD5`-hashed password. [CrackStation](https://crackstation.net/) makes quick work of this hash and reveals the clear-text password  `underwaterfriends`. I can use this for the `ssh` access to the machine using the credential-combination `svcMosh:underwaterfriends`. This also terminates the need for `Lateral Movement`!

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`sudo -l` reveals that the binary `/usr/bin/mosh-server` is allowed to be executed by `svcMosh` as `root`! `mosh-server` is an alternative to `ssh` which provides more robust and responsive shells over long distance links. 

[GTFOBins](https://gtfobins.org/gtfobins/mosh-server/) shows an easy way of instantly getting a shell as `root` (this simultaneously starts a `sudo mosh-server` and connects to it):
```bash
mosh '--server=sudo mosh-server' localhost /bin/sh
```
