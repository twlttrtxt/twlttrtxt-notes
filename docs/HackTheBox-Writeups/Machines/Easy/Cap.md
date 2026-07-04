---
tags:
  - Linux
  - HTTP
  - IDOR
  - Capabilities
---

... is a easy HTB machine which hosts a `http` service which has a `IDOR` vulnerability, allowing you to read network captures of other users. One used his credentials in the `ftp` service, which was not encrypted. These can be reused for the `ssh` service. For the privilege escalation, the `python` binary has extended capabilities, allowing it to set its own `UID` to `0`.

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
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-title: Security Dashboard
|_http-server-header: gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
`ftp` being here is very nice, but as we don't see the output of the `ftp-anon` `nmap` script it means that `anonymous` logins are probably not allowed (i also verified it).
Which is why i decide to investigate the `http` service by opening `burpsuite` and visiting the web-page. It instantly shows me some kind of dashboard of the user `Nathan` without requiring authentication. While searching for `href="/` in the `view-source` to find out which services i can reach from this dashboard, i notice the three following endpoints:

- `/capture`: after 5 seconds, redirects me to `/data/1`, which shows me information about a `.pcap` capture, with the ability to download it.
- `/ip`: displays the output of the OS command (presumably) `ip a`. No user input.
- `/netstat`: displays the output of the OS command (presumably) `netstat -tulnp`. No user input.

### Initial Exploitation
`PCAP` files are usually created by tools which listen to and log the network activity, and due to the 5 second delay, it is possible to assume that a live capture of the data happened in those 5 seconds. After downloading that file, i can view its content using the tool `wireshark`.

Instead of capturing my network traffic in the GUI of `wireshark`, i select the `File > Open` option at the top to view the traffic of the `PCAP` file i just downloaded. When viewing it, i notice nothing out of the ordinary, it seems to only have captured a few TCP three-way handshakes (to verify the sending of packages), and no credentials were captured.

As the `PCAP` file was found on the endpoint `/data/1` i try to view whats on the endpoint `/data/0` (`IDOR`), as that may show me a network capture of someone else! And surely enough, that endpoint has way more packets than `/data/1`. 
Inspecting the packets, i first see `HTTP` communication where the user requested `/` of a host which i cannot reach. The response indicated some kind of `/upload` functionality...
The other communication which took place was towards the `FTP` service which is password-protected. The following information was extracted when right clicking a packet and clicking `Follow > TCP Stream`:
```bash
220 (vsFTPd 3.0.3)

USER nathan

331 Please specify the password.

PASS Buck3tH4TF0RM3!

230 Login successful.
```
> **_NOTE:_**  This communication wouldn't be visible if `sftp` or `https` were used!

I use these credentials in a `ftp nathan@$IP` session, in which i can `get` the `user.txt`.

### Lateral Movement
I assumed that this `ftp` access would be a dead end, as only the `flag.txt` was visible but executing a `cd ..` quickly revealed that the whole file-system was mapped to this service, which turned it into a pseudo-`bash` where i can only read and write files.

When navigating to `/var/www/html` i notice two interesting things.

- The web app is being ran from a `app.py`. I download this file.
- There is a `/uploads` directory which holds the `PCAP` files.

The logic of the `PCAP`-downloading process reveals that the user controls the name of the file, but not the extension, and the file is being sent as an attachment. So putting files here and rendering them doesn't give me `RCE` (after giving this idea a thought, i don't think this works for `python` web servers).

My next idea would be to use his `FTP` password for `SSH` (password reuse attack). And that worked...

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

Sadly, none of these approaches work here, so i need to do some further digging. I first try to list all `SetUID` files using the following command:
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

The ones that were found did not stand out in any way. I try to list all `cron` jobs using these commands:
```bash
for user in $(cut -f1 -d: /etc/passwd); do crontab -u $user -l; done
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# execute "crontab -u $user -l" for each user in /etc/passwd
```

```bash
cat /etc/crontab
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# shows system-wide cron-jobs
```

But i wasn't able to find anything.

When looking through the [HackTricks linux privesc checklist](https://hacktricks.wiki/en/linux-hardening/linux-privilege-escalation-checklist.html) i noticed that below `SUID` commands there are `Capabilities`. A google session taught me that the capabilities of a process refer to its effective `UID` (`SetUID` allows you to execute the binary as the owner). If it is `0`, the process is running as root, if it is not `0`, then not. It is possible though to give more capabilities to a binary using the command `setcap`, so that it can achieve more tasks without requiring its `UID` being `0`. An example of this is `ping`, as it has the extended capability of `CAP_NET_RAW`(allows it to craft arbitrary network packets), as that is usually preserved for `root`.
The capabilities of binaries can be listed using this command:
```bash
getcap -r / 2>/dev/null
```
It turns out that the `/usr/bin/python3.8` binary (presumably the one running the web-server) has the capability of `CAP_SETUID`. This capability allows the binary to change its own `UID` (`CAP_SETGID` allows you to set `GID`). Knowing this, this python command sets its own `UID` to `0`, making it `root` and then it creates a `bash`:
```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/bash", "bash")'
```
This instantly elevates the user to `root`, allowing to read the `/root/root.txt`.