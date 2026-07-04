At this point i was curious of how the AD would look like from an attackers standpoint. The output of `nmap -sV $IP` looks like this:
```powershell
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: test.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Datacenter Evaluation 14393 microsoft-ds (workgroup: TEST)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: test.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
MAC Address: 00:0C:29:07:7D:96 (VMware)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```
Below is a breakdown of all ports:

- `Port 53`: `DNS` server. Translates host-names like `DC01.test.local` into valid IP-addresses.
- `Port 88`: `Kerberos` authentication protocol. Handles login and ticket-based access to services in AD.
- `Port 135`: `RPC` services. Required for AD replication, windows management instrumentation and various windows internal services.
- `Port 139`: `NetBIOS` over TCP. Legacy protocol for file and printer sharing on older windows systems.
- `Port 389 and Port 636`: `LDAP` and `LDAPS` (with `SSL`). used for querying and modifying directory information.
- `Port 445`: `SMB`. Used for file and printer sharing. distributed by group policies.
- `Port 464`: `Kerberos kpasswd`. used for changing or resetting AD passwords.
- `Port 593`: `RPC over HTTP`. Used for Outlook anywhere or AD-administration over HTTP.
- `Port 3268 and Port 3269`: Global Catalog and using `SSL`. Used for forest-wide directory searches for user and groups.
- `Port 5985`: `WinRM`. Enables `PowerShell` remoting for system administration. Similar to `SSH`.