## Shodan

- Search engine for OSINT, you can search network ranges and it gives you information about websites, ports and hostnames. needs an account!

## Subfinder
```bash
subfinder -d <domain> -o subfinder_output
```

- Finds subdomains without brute-forcing
- Does not find all possible domains!
- Visit IPs from `nmap` scan or use the tool `waybackurls` which looks up the domain name on the way back machine to find many different previous versions:

```bash
cat domains.txt | waybackurls > urls
```

## `host` lookup
```bash
while IFS= read -r var; do host $var; done < subfinder_output
```

- gives you the IP addresses of the found sub directories. you can filter for specific ranges this way

## `httpx`
```bash
cat domains | httpx -title -tech-detect -status-code -o http_responses
```

- Gives information of status codes and technology used by visiting the domains via http/s

## Google Dorks
```google
site:<domain>
```

- finds actual websites for otherwise empty domains
