## OSINT

- `crt.sh`: If a SSL/TLS Certificate is created, the CA takes part in "Certificate Transparency logs". These are publicly available on `https://crt.sh`, which can show us subdomains!

- Google Dorking: `site:*.domain.com -site:www.domain.com`, shows us all subdomains WITHOUT the `www` subdomain!

- `./sublist3r.py -d domain.com` can automate the OSINT process!

## Bruteforce
```bash
dnsrecon -t brt -d domain.com
```

## VHosts
```bash
ffuf -w /path/to/wordlist -H "Host: FUZZ.domain.com" -u http://<ip>
# -fs <size> -> Ignores responses of <size>
# -fc <code> -> Ignores responses with specific code
# -fr <regex> -> Filter by regex
```