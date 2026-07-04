First off, the `SSH` server must be started. It can be achieved with these commands:
```bash
/etc/init.d/ssh start
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# starts or stops (stop) ssh server
```

```bash
ssh-keygen -t rsa
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# generates a public/private keypair for ssh named 'rsa'
```

```bash
chmod 600 ./rsa
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# make sure the key is read/write only for the owner
```

```bash
ssh-copy-id username@remotehost
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# copies the public key to the target server
```

To then start the proxy-connection, issue this command when connecting to the server:
```bash
ssh -D 1096 username@remotehost
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# starts a dynamic port forward on port 1096
```

#### Using it
Check if the `/etc/proxychains4.conf` is setup correctly to show :
`socks4  127.0.0.1 9050`. If that is the case, you can use any command by pre-pending `proxychains` (or `proxychains4` in some versions):
```bash
proxychains <any-command>
```
That command will then reach ports on the target and not on `localhost`.

Alternatively, when trying to access web-applications, `burpsuite` allows the use of socks proxies. Therefore, go to Burps Settings and search `Socks Proxy`: 

- Override options for this project only
- host: 127.0.0.1
- port: 1096
- Do DNS lookups over SOCKS proxy