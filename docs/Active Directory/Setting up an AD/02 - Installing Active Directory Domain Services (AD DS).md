Using Server Manager:

1. Select Manage -> Add Roles and Features
2. Choose Role-based or feature-based installation
3. Select the targeted server (e.g. `dcsrv01.test.local`)
4. Use the Active Directory Domain Services (AD DS) role.
5. When prompted, include the required management tools.

Alternatively, `PowerShell` can be used:
```powershell
Get-WindowsFeature *service*
```
and:
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeAllSubFeature -IncludeMangementTools
```

### Promoting the Server to a DC
Now, it is time to promote the server to a domain controller. But before doing so, the DNS settings must be configured. Since a Domain Controller also functions as a DNS server, the preferred DNS server on the network adapter must be set to the server's own IP address.
To promote the server, the following steps must be done:

1. Click the notification flag in Server Manager.
2. Select Promote this server to a domain controller.
3. Choose `Add a new forest`.
4. Enter the root domain (e.g. `test.local`)
5. Select the appropriate forest and domain levels. These determine which AD features are available.
6. Optionally configure external DNS settings if required.
7. Set a strong Directory Services Restore Mode (`DSRM`) password. It is critical for restoring / recovering an AD environment.

The server will automatically restart after this promotion.

### Running the Best Practices Analyzer (BPA)
After the restarting of the server, open Server Manager and then:

1. Select Local Server.
2. Scroll down to Best Practices Analyzer.
3. Click Tasks -> Start BPA Scan.

The BPA analyzes the DC for common configuration issues.

### Configuring a Reverse Lookup Zone
Running `nslookup` may display something like this:
```powershell
Default Server: Unknown 
Address: ::1
```

Although host-name resolution works correctly, a reverse DNS lookup zone is missing. To create one, head to Server Manager and then:

1. Go to Tools -> DNS.
2. Expand the server.
3. Right-click Reverse Lookup Zones.
4. Select New Zone.
5. Enter the network ID, which is the network portion of the sub-net (e.g. `172.16.67` for a `/24` network)

By default, Active Directory does not create reverse lookup zones automatically, but configuring one results in a cleaner DNS environment.
After creating the zone, the DNS records can be registered as follows:
```powershell
ipconfig /registerdns
```

### Adjusting the DNS Configuration
After installing AD, Windows may automatically configure the DNS server to use the IPv4 loopback address. To restore the preferred DNS server:

1. Open Network and Sharing Center.
2. Open the properties of the Ethernet adapter.
3. Select Internet Protocol Version 4 (TCP/IPv4).
4. Set the Preferred DNS Server back to the server's own IPv4 address.

For TCP/IPv6, leave the DNS server configuration set to "Obtain DNS server address automatically", unless the environment specifically requires IPv6 DNS.

### Configure Time Synchronization
The `BPA` (Best Practices Analyzer) may report a warning that the `PDC Emulator FSMO role` is not configured with a reliable time source. This is important, as the primary domain controller (`PDC Emulator`) acts as the time source for the entire AD domain. All domain controller and domain-joined clients synchronize their clocks with it. If this time synchronization shifts beyond +- 5 minutes, Kerberos authentication will fail, preventing users from logging on.

The external NTP source can be configured using the recommended command in the `BPA`, but make sure to replace Microsoft's default time server `time.windows.com` with a regional `NTP` server. Using an external NTP server ensures correct time synchronization through the AD environment.