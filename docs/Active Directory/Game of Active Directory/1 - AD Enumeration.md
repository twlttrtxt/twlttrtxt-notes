First off, i will try to `ping` each IP address to see if they were set up correctly. The setup-script has given me these IP-addresses:
```bash
+------------+--------------------------------+-------+-------------+
| PROXMOX ID |            VM NAME             | POWER |     IP      |
+------------+--------------------------------+-------+-------------+
|    107     | GOAD1dbf1f-router-debian11-x64 |  On   | 10.3.10.254 |
|    108     | GOAD1dbf1f-GOAD-DC01           |  On   | 10.3.10.10  |
|    109     | GOAD1dbf1f-GOAD-DC02           |  On   | 10.3.10.11  |
|    110     | GOAD1dbf1f-GOAD-DC03           |  On   | 10.3.10.12  |
|    111     | GOAD1dbf1f-GOAD-SRV02          |  On   | 10.3.10.22  |
|    112     | GOAD1dbf1f-GOAD-SRV03          |  On   | 10.3.10.23  |
|    113     | GOAD1dbf1f-kali                |  On   | 10.3.10.99  |
+------------+--------------------------------+-------+-------------+
```
I have created an `ips.txt` file to make further automation steps easier:
```ips.txt
10.3.10.10
10.3.10.11
10.3.10.12
10.3.10.22
10.3.10.23
```
Each virtual machine was reachable via `ping`! To make sure that there are no other hidden IP addresses in the network, i issued the following command:
```bash
fping -g 10.3.10.0/24
```

### SMB Enumeration
The following command enumerates a IP-range for `SMB` services. This may find mis-configurations in the `SMB` service:
```bash
nxc smb 10.3.10.0/24
```
The output shows the following information:
```bash
SMB 10.3.10.12 445 MEEREEN [*] Windows 10 / Server 2016 Build 14393 x64 (name:MEEREEN) (domain:essos.local) (signing:True) (SMBv1:True)

SMB 10.3.10.11 445 WINTERFELL [*] Windows 10 / Server 2019 Build 17763 x64 (name:WINTERFELL) (domain:north.sevenkingdoms.local) (signing:True) (SMBv1:False) 

SMB         10.3.10.10      445    KINGSLANDING     [*] Windows 10 / Server 2019 Build 17763 x64 (name:KINGSLANDING) (domain:sevenkingdoms.local) (signing:True) (SMBv1:False) 

SMB 10.3.10.22 445 CASTELBLACK [*] Windows 10 / Server 2019 Build 17763 x64 (name:CASTELBLACK) (domain:north.sevenkingdoms.local) (signing:False) (SMBv1:False) 

SMB 10.3.10.23 445 BRAAVOS [*] Windows 10 / Server 2019 Build 17763 x64 (name:BRAAVOS) (domain:essos.local) (signing:False) (SMBv1:False)
```
This tells me that there are 3 domains under the 5 machines. Therefore, 3 domain controllers should also be in place. The domains are these:
```bash

- sevenkingdoms.local
- north.sevenkingdoms.local
- essos.local

```
It is also worth noting that `CASTELBLACK` and `BRAAVOS` do not use `SMB signing`, which allows for `SMB-Relay` attacks!

### DNS Enumeration
`nslookup` can be used to find out which servers are actually used as domain controllers. It can also be used for other scenarios:
```bash
# Find Domain Controllers
nslookup -type=srv _ldap._tcp.dc._msdcs.<DOMAIN> <DNS_SRV>

# Find Principal Domain Controllers
nslookup -type=srv _ldap._tcp.pdc._msdcs.<DOMAIN> <DNS_SRV>

# Find Global Catalog (DC but with more data)
nslookup -type=srv gc._msdcs.<DOMAIN> <DNS_SRV>

# Find other Services
nslookup -type=srv _kerberos._tcp.<DOMAIN> <DNS_SRV>
nslookup -type=srv _kpasswd._tcp.<DOMAIN> <DNS_SRV>
nslookup -type=srv _ldap._tcp.<DOMAIN> <DNS_SRV>
```

These scans require the IP address of the DNS server (Typically the domain controller). If needed, they can be found using `nmap`:
```bash
nmap -v -sV -p 53 10.3.10.0/24
```
```bash
# or, by using UDP:
```
```bash
nmap -v -sV -sU -p 53 10.3.10.0/24
```

If there are any `reverse lookup zones` configured in the DNS, the host-names can be found as follows:
```bash
host <IP>
# OR
nslookup -type=ptr <IP>
```

This enumeration leads to the following information:
```bash
10.3.10.10: KINGSLANDING -> DC01 in sevenkingdoms.local
10.3.10.11: WINTERFELL -> DC02 in north.sevenkingdoms.local
10.3.10.12: MEEREEN -> DC03 in essos.local
10.3.10.22: CASTELBLACK -> SRV02 in north.sevenkingdoms.local
10.3.10.23: BRAAVOS -> SRV03 in essos.local
```

These IPs can then be added to the `/etc/hosts` file using the following entries, so that local DNS resolution and `Kerberos` work properly:
```bash
# GOAD
10.3.10.10.10 sevenkingdoms.local # additionally, due to it being a DC
10.3.10.10.10 kingslanding.sevenkingdoms.local
10.3.10.10.10 kingslanding

10.3.10.10.11 north.sevenkingdoms.local # additionally, due to it being a DC
10.3.10.10.11 winterfell.north.sevenkingdoms.local
10.3.10.10.11 winterfell

10.3.10.10.12 essos.local # additionally, due to it being a DC
10.3.10.10.12 meereen.essos.local
10.3.10.10.12 meereen

10.3.10.10.22 castelblack.north.sevenkingdoms.local
10.3.10.10.22 castelblack

10.3.10.10.23 braavos.essos.local
10.3.10.10.23 braavos
```

### NBT-NS
`NetBIOS-NameServer` is a standard protocol which acts as a fallback protocol for AD-environments for name resolution, if `DNS` may not be working. This protocol can be enumerated with the tools `nbtscan` and `nmblookup`:
```bash
nbtscan -r 10.3.10.0/24
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# does a reverse lookup on a IP-range
```

```bash
nmblookup -A <IP>
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# finds names and workgroups from a IP address
```


### Responder
`responder` is a tool which is typically used to poison protocols which act as fallback for DNS (e.g. `LLMNR`, `NBTNS`, `MDNS`), or to do `WPAD` spoofing.

It is possible however, to also use `responder` in `analyze` mode to possibly find the following information:

- Domain Controllers, SQL Servers, Workstations
- Fully qualified domain names
- Windows version
- State (`ENABLED / DISABLED`) of protocols such as `LLMNR, NBTNS, MDNS, LANMAN, ...`

To analyze the network traffic, the following `responder` command can be used:
```bash
responder --interface "eth0" --analyze
```
```bash
# or, shorter:
```
```bash
responder -I "eth0" -A
```

### Kerberos setup
First off, the tool `krb5-user` is required:
```bash
sudo apt install krb5-user
```
... then, its configuration file `/etc/krb5.conf` must be edited as follows:
```bash
[libdefaults]
  default_realm = essos.local
  kdc_timesync = 1
  ccache_type = 4
  forwardable = true
  proxiable = true
  fcc-mit-ticketflags = true
[realms]
  north.sevenkingdoms.local = {
      kdc = winterfell.north.sevenkingdoms.local
      admin_server = winterfell.north.sevenkingdoms.local
  }
  sevenkingdoms.local = {
      kdc = kingslanding.sevenkingdoms.local
      admin_server = kingslanding.sevenkingdoms.local
  }
  essos.local = {
      kdc = meereen.essos.local
      admin_server = meereen.essos.local
  }
  ...
```

To test if the setup was successful, the tool `getTGT` from the `impacket` suite may be used:
```bash
impacket-getTGT essos.local/khal.drogo:horse
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# gets the Ticket-granting-ticket for the user khal.drogo on the domain essos.local
```

```bash
export KRB5CCNAME=./khal.drogo.ccache
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# the .ccache file location is saved in an environment variable, so that it can be used further by other tools
```

```bash
impacket-smbclient -k @braavos.essos.local
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# authenticate using the kerberos TGT to the smb service of BRAAVOS
```

### NMAP
`nmap` is a great tool to map the offered service in a network. The following `nmap` scan was used on the `ips.txt` which i have saved before:
```bash
sudo nmap -sCV -oA output -iL ips.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -sCV: Use Default Scripts & Version Detection
# -oA: Use all output formats: gnmap, nmap, xml
# -iL: Input from List
```

These are the most interesting ports which should be looked out for in AD-environments:

- `53/TCP` and `53/UDP` for DNS
- `88/TCP` for Kerberos authentication
- `135/TCP` and `135/UDP` MS-RPC epmapper (EndPoint Mapper)
- `137/TCP` and `137/UDP` for NBT-NS
- `138/UDP` for NetBIOS datagram service
- `139/TCP` for NetBIOS session service
- `389/TCP` for LDAP
- `636/TCP` for LDAPS (LDAP over TLS/SSL)
- `445/TCP` and `445/UDP` for SMB
- `464/TCP` and `445/UDP` for Kerberos password change
- `3268/TCP` for LDAP Global Catalog
- `3269/TCP` for LDAP Global Catalog over TLS/SSL

`nmap` may also be used to scan for domain controllers using the following command:
```bash
nmap -sS -n --open -p 88,389 10.3.10.0/24
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -sS: for TCP SYN scan 
# -n: for no name resolution 
# --open: to only show (possibly) open port(s) 
# -p for port(s) number(s) to scan
```

It also offers nice scripts which can be used to possibly find information on active users. These can be used using the following `nmap` scan:
```bash
nmap -n -sV --script "ldap* and not brute" -p 389 <IP> --script-args="ldap.maxobjects=300" -oA output
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -n: Never do DNS Resolution
# -sV: Version Detection
# --script: Use following scripts
# -p 389: only scan port 389
# ldap.maxobjects=300: not only the first 20, but more
# -oA: output into all formats
```

Another script of `nmap` may be also used to enumerate `DHCP` hosts:
```bash
nmap -n -sV --script broadcast-dhcp-discover <IP> -oA output
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -n: Never do DNS Resolution
# -sV: Version Detection
# --script: Use following script
# -oA: output into all formats
```
