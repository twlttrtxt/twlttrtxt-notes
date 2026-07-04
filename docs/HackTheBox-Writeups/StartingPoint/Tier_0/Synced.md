---
tags:
  - Linux
  - rsync
  - Guest access
---

... is a very simple HTB machine which demonstrates the usage of `nmap` to find a vulnerable port which allows unauthenticated users to use `rsync` to synchronize remote files to their machine.

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
PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)
```
This server seemingly only hosts an instance of `rsync` (a.k.a. Remote Sync). It is typically used to synchronize (transfer) data between two paths on one machine or between two machines, and it comes preinstalled on my `kali`.
### Initial Exploitation
To find out how `rsync` works, i have issued the `rsync --help` command. Under the usage section i find this command and the very helpful part:
```bash
rsync [OPTION]... [USER@]HOST::SRC [DEST]
...
The ':' usages connect via remote shell, while '::' & 'rsync://' usages connect
to an rsync daemon, and require SRC or DEST to start with a module name.
```

So i infer that i can use either of these commands:
```bash
rsync $IP::
```
```bash
# or
```
```bash
rsync rsync://$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# NOT this, as it tries to connect via SSH, which is (presumably) not open on that machine
# rsync $IP:
```

And those commands return this info:
```bash
$ rsync $IP::
public          Anonymous Share
```
To further step into this `public` share, simply append it to the command like `rsync $IP::public` or `rsync rsync://$IP/public`.
Within that folder, the `flag.txt` is reachable. To synchronize it to my machine, i specify it like this:
```bash
rsync $IP::pubic/flag.txt .
# or
rsync rsync://$IP/public/flag.txt .
```
