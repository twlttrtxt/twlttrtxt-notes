# Cleartext Protocols
### Telnet

- Outdated Remote shell service -> SSH without encryption, cleartext readability in wireshark
- On port `23`, Usable via `telnet <host>`.

### HTTP

- telnet can also be used to manually send HTTP Requests:

```bash
telnet <host> <port>
# ...
GET /index.html HTTP/1.1
host: telnet
```

### FTP

- Usual workflow:

```bash
ftp <host>
#...
Name (<host>:<user>): frank
331 Please specify the password.
Password: #...
230 Login successful
#...
ftp> ls
150 Here comes the directory listing
#...
ftp> get <filename>
226 Transfer complete.
#...
ftp> exit
221 Goodbye.
```

### SMTP

- Usual workflow:

```bash
telnet <host> 25
#...
helo telnet
#...
mail from: <your email>
#...
rcpt to: <receiver email>
#...
data
<type email>
#...
quit
```

### POP3

- Usual workflow:

```bash
telnet <host> 110
#...
USER <name>
#...
PASS <pass>
#...
STAT
OK nn mm # nn -> number of mails in inbox,  mm -> inbox size
#...
LIST
#... list of new messages on server
RETR 1
#... retrieves first message in list
```

### IMAP

- Usual workflow:

```bash
telnet <host> 143
#...
c1 LOGIN <name> <pass>
# c1 OK LOGIN Ok.
c2 LIST "" "*"
#... Lists mail folders
c3 EXAMINE INBOX
#... Lists inbox
```
 
