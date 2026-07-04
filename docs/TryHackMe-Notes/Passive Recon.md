```bash
whois <domain-name>
```

- gives info about registrar, contact info of registrant, creation-, update-, and expiry dates, and name server

```bash
nslookup <domain-name>
```

- gives info about the IP-Address of a domain name. 
- you can use different options:

```bash
nslookup -type=A <domain> # IPv4 Addresses
nslookup -type=AAA <domain> # IPv6 Addresses
nslookup -type=CNAME <domain> # Canonical Names
nslookup -type=MX <domain> # Mail Servers
nslookup -type=SOA <domain> # Start of Authority
nslookup -type=TXT <domain> # TXT Records
```

- You may also use different public DNS servers by appending them like this:
- options may be `1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4`, `9.9.9.9`, `194.112.112.112`

```bash
nslookup <domain> <DNS-Server>
```

```bash
dig <domain> <option> # like A, AAAA, TXT, SOA, ...
```

- is an alternative to nslookup which might give more info

- [DNSDumpster](https://dnsdumpster.com/), and [shodan.io](https://www.shodan.io/)can give valuable info about domain names / subdomains automatically

## More Tools

- amass (passive mode)

```bash
amass enum -passive -d example.com
```

- subfinder

```bash
subfinder -d example.com
```

- assetfinder
- crt.sh searches
- certificate transparency logs
- Wayback Machine
- search engine indexing