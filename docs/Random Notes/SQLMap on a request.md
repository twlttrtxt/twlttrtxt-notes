Save the request with vulnerable parameters. Make sure the host is a actual host-name which is reachable, and the vulnerable parameter is marked with a star (Custom injection marker).
```bash
POST /login HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

username=admin&password=pass123*
```

You can then specify this request in a `sqlmap` scan 
```bash
sqlmap -r <request_file> <options like --batch, ...>
```