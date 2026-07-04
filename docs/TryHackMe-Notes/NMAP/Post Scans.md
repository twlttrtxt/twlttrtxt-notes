- Service detection to know what services are actually running behind ports (not guesses)

```bash
sudo nmap -sV --version-intensity <level> <host>
# level may range from 0-9 in intensity
```

- OS detection to find out what OS is running these services:

```bash
sudo nmap -sS -O <host>
```

- Scripting allows you to extend nmaps features. There are many scripts in `/usr/share/nmap/scripts`, which you can list by different categories like:

```bash
ls /usr/share/nmap/scripts/http*
```

- To use `default` Scripts, you can use `--script=default` or `-sC`. You may also use single scripts like `--script <name>`. instead of default, you have these other options of scripts:

| Script Category | Description                                                            |
| --------------- | ---------------------------------------------------------------------- |
| `auth`          | Authentication related scripts                                         |
| `broadcast`     | Discover hosts by sending broadcast messages                           |
| `brute`         | Performs brute-force password auditing against logins                  |
| `default`       | Default scripts, same as `-sC`                                         |
| `discovery`     | Retrieve accessible information, such as database tables and DNS names |
| `dos`           | Detects servers vulnerable to Denial of Service (DoS)                  |
| `exploit`       | Attempts to exploit various vulnerable services                        |
| `external`      | Checks using a third-party service, such as Geoplugin and Virustotal   |
| `fuzzer`        | Launch fuzzing attacks                                                 |
| `intrusive`     | Intrusive scripts such as brute-force attacks and exploitation         |
| `malware`       | Scans for backdoors                                                    |
| `safe`          | Safe scripts that won’t crash the target                               |
| `version`       | Retrieve service versions                                              |
| `vuln`          | Checks for vulnerabilities or exploit vulnerable services              |
```bash
sudo nmap -A <host>
# is equivalent to
sudo nmap -sC -sV -O --traceroute <host>
```

```bash
sudo nmap -oN <host> # saves output in normal nmap output
sudo nmap -oG <host> # saves output in greppable nmap output
sudo nmap -oX <host> # saves output in XML output
sudo nmap -oN <host> # saves output in all outputs
```