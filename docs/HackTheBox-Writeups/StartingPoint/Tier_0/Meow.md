---
tags:
  - Linux
  - telnet
  - Guest access
---

... is a very simple HTB machine which demonstrates the usage of `nmap` to find a vulnerable port which allows `telnet` connections with a highly privileged user, without providing a password.

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
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Initial Exploitation
After researching and studying the `man telnet` page, it is made clear that this is an older alternative to `ssh`. The typical usage of `telnet` is: `telnet <IP> <port>`. 
After using that command with the variable `$IP` and the port `23`, we are presented with the following banner:
```bash
  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: 
```

A brute-forcing attempt using the tool `hydra`, or the `nmap` script `telnet-brute` can now be started against this login. Although possible, it is always wiser to try simple login combinations such as `administrator:password`, `admin:admin`, or `root:root`. It turns out, that the username `root` does not require a password, and simply lets you execute `telnet` commands on the target with elevated privileges.

### Summary

Below is a visualized summary of the exploitation steps used in this machine.

``` mermaid
graph LR
  A[FTP<br>service] -->|Weak<br>credentials| B[telnet command<br>execution];
```