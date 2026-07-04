---
tags:
  - Linux
  - redis
  - Guest access
---

... is a very simple HTB machine which demonstrates the usage of `nmap` with a wider range of ports than usual, to find a `redis` service which allows users to retrieve stored key-value pairs on the server.

### Reconnaissance
The tool `nmap` is used to do the initial reconnaissance of any target, as it very reliably sends packets to specific ports of the target to verify if they are open, closed, or filtered. The following command is used as a standard `nmap` scan:
```bash
sudo nmap -sCV $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sudo: optional, but makes the scan a bit faster and stealthier, as no TCP connect() is used.
# -sC (or --script=default): uses the default scripts of nmap. can quickly discover simple vulnerabilities, such as anonymous logins.
# -sV: further scans open ports to determine the actual service which is running on them, as an open port 80 does not directly imply a HTTP service.
```

the output of `nmap` tells us this:
```bash
All 1000 scanned ports on localhost (10.10.10.10) are in ignored states.
Not shown: 1000 closed tcp ports (reset)
```
This tells us that the 1000 most frequently used network ports are not open on this machine. What this means is that we need to extend the range of the `nmap` scan to actually scan more than those 1000. We could try scanning a big range of ports using `-p 1-10000`, or all 65.535 network ports using `-p-`. As scanning all ports takes too long, and scanning a range of ports includes a lot of rarely used ports, i have decided to increase the number of most used ports using this command:
```bash
sudo nmap -sCV --top-ports 10000 $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# --top-ports: scan the 10.000 most frequently used ports. defaults to 1.000.
```

This new `nmap` scan gives us the following output:
```bash
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```
The service `redis` (a.k.a remote dictionary server) is a in-memory-database which stores data in key-value pairs in the RAM instead on the disk.
### Initial Exploitation
After researching `redis` and how to interact with it, i have stumbled across the tool `redis-cli` , which comes pre-installed on kali linux!

After using `redis-cli --help` to find out how to specify the remote host, i have issued the following command:
```bash
redis-cli -h $IP
```
This has thrown me into an interactive session where i can issue `redis` commands to the server.

That lead me to the [redis docs](https://redis.io/docs/latest/develop/tools/cli/) where i found out that i can get a list of all keys (values are mapped to those!) using the redis command `SCAN 0`. The inferior option to this `SCAN` command is the `KEYS *` command, as it does not block the server. The output of either `KEYS *` or `SCAN 0` displays the following keys:
```bash
1) "flag"
2) "numb"
3) "temp"
4) "stor"
```

The following info was taken from [this stackoverflow question](https://stackoverflow.com/questions/8078018/get-redis-keys-and-values-at-command-prompt).Using `redis`, fetching the values to these keys is a bit more nuanced, as you will need to know the type of value which is stored. To find out the type of value which is stored in a key, you can issue the command `TYPE <key>`. To fetch the values for specific types, you will need these respective commands:

- `string`-type: `GET <key>`
- `hash`-type: `HGETALL <key>`
- `list`-type: `LRANGE <key> 0-1`
- `set`-type: `SMEMBERS <key>`
- `zset`-type: `ZRANGE <key> 0-1 WITHSCORES`

In the CTF, the values are mostly strings, so the flag can be retrieved using the command `GET flag`.
