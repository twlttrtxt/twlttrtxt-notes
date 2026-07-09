---
tags:
  - Linux
  - HTTP
  - WordPress
  - CVE
  - Directory Traversal
---

... is a easy HTB machine where a `HTTP` header reveals a domain name. A subdomain and its login endpoint are revealed via a `WordPress` vulnerability. After logging in, a `directory traversal` attack allows you to read the clear-text password of a `ssh` user. For privilege escalation, a vulnerability in `polkit` allows you to gain `root` access.

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
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   2048 10:05:ea:50:56:a6:00:cb:1c:9c:93:df:5f:83:e0:64 (RSA)
|   256 58:8c:82:1c:c6:63:2a:83:87:5c:2f:2b:4f:4d:c3:79 (ECDSA)
|_  256 31:78:af:d1:3b:c4:2e:9d:60:4e:eb:5d:03:ec:a0:22 (ED25519)
80/tcp  open  http     Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_http-title: HTTP Server Test Page powered by CentOS
443/tcp open  ssl/http Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| http-methods: 
|_  Potentially risky methods: TRACE
| tls-alpn: 
|_  http/1.1
|_http-title: HTTP Server Test Page powered by CentOS
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US
| Subject Alternative Name: DNS:localhost.localdomain
| Not valid before: 2021-07-03T08:52:34
|_Not valid after:  2022-07-08T10:32:34
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_ssl-date: TLS randomness does not represent time
```
This is a typical HTB web-vulnerability machine where a mis-configuration in the `http` service leads to `RCE` or `ssh` credentials. To further investigate the `http` service, i visit `http://$IP` in `burpsuite's` integrated browser, as no domain name seems to be in place.

The home-page of the `http` service displays a `CentOS` default page, indicating that the server has been set up correctly. The `https` page on port `443` has the same content. I decided to further investigate the responses to the `http` and `https` requests and noticed a discrepancy between them. The `http` service uses this additional `HTTP Header`: 
```http
X-Backend-Server: office.paper
```
Due to this, i added this domain to my `/etc/hosts` file using the following command:
```bash
echo "$IP office.paper" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting the new domain `http://office.paper`, i receive new content. I see the home-page to a `Blunder Tiffin, Paper Company`. It seems to only serve blog post of the user `Prisonmike`. Scrolling all the way down reveals the footer which says `Proudly Powered By WordPress`. Due to this, i start a broad vulnerability scan using `wpscan`:
```bash
wpscan --url http://office.paper/
```
This results in the following information:
```bash
# ...
[+] WordPress version 5.2.3 identified
# ...
```

### Initial Exploitation
A quick google search for this `WordPress` version reveals this [exploit-db exploit](https://www.exploit-db.com/exploits/47690). By adding the URL parameter `?static=1`, i should be able to view secret content. And i do get to see the drafts from the blog pages. A very nice piece of information results from this secret draft:
```http
# Secret Registration URL of new Employee chat system

http://chat.office.paper/register/8qozr226AhkCHZdyY
```
I can add this new sub-domain to my `/etc/hosts` file using this command:
```bash
sudo sed -i "s/$IP office.paper/$IP office.paper chat.office.paper/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

Visiting the new sub-domain `http://chat.office.paper` gives me a login window for a application called `rocket.chat` where i am asked for credentials. It explicitly states that `Registration can only be done using the secret registration URL!`, which i have luckily already found from the secret `WordPress` draft. I visit `http://chat.office.paper/register/8qozr226AhkCHZdyY`, where i register a new account to access the protected content of `rocket.chat`.

The `rocket.chat` application seems to have `Channels`, where `Users` can exchange messages! I can see that i am part of the `#general` chat (`readonly`), so i decide to read the previous messages of it. It turns out that a bot was installed which can be interacted with by using the command `recyclops help` to view its commands.

To start a chat with this `recyclops` bot, i navigate to `Directory` (`world` icon), and then to `Users`, where i can click his name. The `recyclops help` command reveals his commands. Only these seem interesting though:

- `recyclops file <filename>`: executes the `OS-command` `cat` on the specified file and shows me its output. Executing it on a non-existing file tells me that the current directory is `/home/dwight/sales/`
- `recyclops list`: executes the `OS-command` `ls -la` and shows me it's output. Just executing this shows me the content of `/sales/`, of which `dwight` is the owner.

My first instinct was to inject `OS-commands`, but all command separators seem to be sanitized. The next idea is `path traversal`. And funnily enough, both commands are vulnerable to `path traversal`:

- `recyclops file ../../../etc/passwd`: shows me its content
- `recyclops list ../../../`: shows the `root` directory `/`

I was not able to find any `ssh` private-keys in `dwight`'s directory, as there were no files there. Using `recyclops list ../` reveals a directory `hubot`. I also list that and notice a `.env` file. I am able to read it using `recyclops file ../hubot/.env`, which holds the password to the `recyclops` bot, which is `Queenofblad3s!23`.

I am able to do a password-reuse attack on the `ssh` service using the credentials `dwight:Queenofblad3s!23`, removing the need to do lateral movement.

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

But none of them show anything interesting. What is interesting however, is the look of the `ssh` session, as it does not look like the familiar `ssh` interfaces I've encountered before. When investigating the `nmap` output again, it says this in the version tab:
```bash
OpenSSH 8.0 (protocol 2.0)
```
... instead of the usual `ssh` version:
```bash
OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
```

This made me think that the operating system is something different than `ubuntu`. Investigating `ls /etc | grep release` tells me that this system is running `redhat`. But googling it's version and the `linux` version found in `uname -a` did not result in any usable `CVE's`.

The next step in privilege escalation would be to enumerate the `SetUID` binaries which can be executed as `root` with the following command:
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

There are no binaries which stood out, they were mostly standard `linux` binaries. Although, in the output i noticed the `polkit-agent-helper-1`. Due to the history of vulnerabilities in `polkit` (`PolicyKit`, manages privileges between processes), i decided to investigate it's version using `pkexec --version`, but it says `pkexec must be setuid root`. I alternatively can check the package version using `rpm -q polkit` and it told me the version `0.115-6`. 

Googling this version reveals the two `CVE's` `CVE-2021-4034` and `CVE-2021-3560`. Trying `CVE-2021-4034` did not work, as the `pkexec` binary somehow does not work (`must be setuid root`). `CVE-2021-3560` however is more interesting, as it does not require the usage of `polkit's` binary. When requesting to create a new user and abruptly terminating the command while `polkit` is processing it, `polkit` gets an error and assumes that the request was from `root`, effectively bypassing credential checks.

As the timing with this can be very delicate, a [PoC on Exploit-DB](https://www.exploit-db.com/exploits/50011) will automatically abuse this. I deploy this `exploit.sh` on the machine, make it executable with `chmod +x exploit.sh` and run it with `./exploit.sh`.

After succeeding, i am able to `su - hacked` to the new account with the password `password`. That account is then allowed to run `sudo su`, which instantly gives him `root` access!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|WordPress<br>vulnerability| B[Secret<br>leak]
  B -->|Login| C[Hidden<br>HTTP<br>Service];
  B -->|Register| D[Hidden<br>Registration];
  C -->|Arbitrary<br>file-read| E[Config<br>file];
  E -->|Password<br>reuse| F[SSH<br>access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>access] -->|Access| B[vulnerable<br>polkit];
  B -->|Bypass<br>credential-check| D[Switch user<br>to root];
```