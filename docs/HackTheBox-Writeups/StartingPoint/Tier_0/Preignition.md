---
tags:
  - Linux
  - HTTP
  - PHP
  - Weak credentials
---

... is a very simple HTB machine which hosts a simple web-server displaying the default page for a `nginx` service. After forceful browsing, a `admin.php` can be found which allows the use of weak credentials.

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
80/tcp open  http    nginx 1.14.2
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.14.2
```
This server seems to only host a HTTP service on port 80. Interaction is therefore done with a browser, i am using `firefox`.

When visiting the web-page of that server, it greets us with a default `nginx` page which indicates that the server has been set up correctly, so there is no way of interacting with this page. `nginx` is a very popular open-source web server.
I have looked for CVEs matching the version `1.14.2` of the service, but most of them lead to denial of service. The source code of the page does not display any secrets in comments. Different browser debugging tools also do not show anything interesting.
### Initial Exploitation
My next approach is to use forceful browsing, as I've hit a dead end. many different tools exist for a such purpose. Due to it's simplicity and speed, I've chosen to use `dirb` as follows:
```bash
dirb http://$IP
```

`dirb` has found the following file:
```bash
---- Scanning URL: http://10.10.10.10/ ----
+ http://10.10.10.10/admin.php (CODE:200|SIZE:999)
```

This page displays an "Admin Console Login", where the user is asked for credentials. My first approach of trying default credentials such as `root:root` or `admin:admin` worked instantly, as `admin:admin`was a correct combination of credentials. The flag is displayed on the screen!