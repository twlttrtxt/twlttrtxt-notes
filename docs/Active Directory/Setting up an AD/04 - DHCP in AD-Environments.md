The Dynamic Host Configuration Protocol (`DHCP`) is especially important in AD environments, as a typical router-based DHCP server often does not provide the correct DNS server configuration for an AD environment. In an AD, clients must receive the Domain Controller as the DNS server to ensure domain resolution and authentication.

### Role Separation Best Practice
A good AD design is when each server has is own dedicated role. The domain controller should only run Active Directory Domain Services and a DNS server. 

It should be avoided that a single server has too many roles, as it makes server migration and troubleshooting more difficult, and increases the risk during updates.

### Installing the DHCP server role
To do so, navigate to Server Manager and open:

1. Manage -> Add roles and features
2. Select DHCP Server
3. Complete the installation.
4. Finish DHCP post-installation configuration.

If the DHCP server will be part of the domain, it must be authorized within the active directory, or it won't be allowed to distribute IP-addresses.

### Behavior without DHCP
Executing these commands will typically result in the following output on a `client01` machine:
```powershell
hostname -> client01 
ipconfig -> 169.254.x.x /16
```

The IP range `169.254.0.0/16` is called Automatic Private IP Addressing (`APIPA`) and it indicates that no DHCP server was reachable and the client assigned itself a fallback IP address.

### Creating a DHCP scope
To create a scope for DHCP, go to Server manager and navigate to the following settings:

- Tools -> DHCP Server -> `DC01.test.local`
- Navigate to IPv4
- Right-click -> New Scope

Then, configure the scope with the desired values (alternatively, create a large range and use exclusions for DC IP's or servers):

- `Name`: `Test DCHCP`
- `Start IP`: `172.16.67.100`
- `End IP`: `172.16.67.129`
- `Subnet Mask`: `/24`
- `Default Gateway`: `172.16.67.2` (important!)
- `DNS Server`: `172.16.67.10` (important! must be DC!)
- `Domain Name`: `test.local` (important!)

### Client Domain Join process
When configured correctly, the clients receive an IP address automatically. Alongside that, they receive the correct DNS server `DC01`, and can resolve the domain `test.local` to join the AD domain successfully.

To verify that it works, issue this command:
```powershell
ipconfig /all
```
And look for the assigned dynamic IP, DHCP server information and the DNS server pointing to the DC.

The first client typically receives `.100`, and the next clients receive `.101`, `.102`, ... The DHCP server tracks and manages these IP-leases automatically.