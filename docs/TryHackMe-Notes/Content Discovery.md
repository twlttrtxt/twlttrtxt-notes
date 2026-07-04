## Manual Discovery

- `/robots.txt` / `sitemap.xml` exposed: guaranteed directories

- If you can download the `favicon.ico`, get its MD5 Hash and look up which framework it uses

- `curl <website> --head` -> can reveal info via `X-Powered-By` or `Server` headers

## OSINT

- Google Dorking:

```bash
site:website.com #results from this site
inurl:admin #must have this world in url
filetype:pdf #only these files
intitle:admin #has this in the title
```

- `Wappalyzer` can help identify used technology stack

- `Waybackmachine` can reveal old versions of the website

- `Github` to search for company names, or target names, maybe revealing code

- Misconfigured `S3 Buckets` can reveal sensitive information. They can be found by visiting `https://{name}.s3.amazonaws.com`, with the name being the name or domain of the company, or variations:

```bash
{name}-assets
{name}-www
{name}-public
{name}-private
...
```

## Automated Discovery (should be last resort)
```bash
ffuf -w /path/to/wordlist -u http://<url>/FUZZ
```

```bash
dirb http://<url>/ /path/to/wordlist
```

```bash
gobuster dir --url http://<url>/ -w /path/to/wordlist
```