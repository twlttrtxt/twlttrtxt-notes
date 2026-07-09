---
tags:
  - Linux
  - MongoDB
  - Guest access
---

... is a very simple HTB machine which requires an extended port range scan of `nmap` to find an instance of `mongodb` running on the server, which does not require authentication to read its data.

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
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
`SSH` (Secure Shell) is a pretty secure protocol (long waits on failed login attempts), and possible usernames on the system are not known, which is why a dictionary attack is not a smart approach. I have tried some default credentials (e.g. `root:root`, or `admin:admin`), but they do not seem to work.

To continue the reconnaissance, i will extend the range of the `nmap` scan to actually scan more than the 1000 most used. I have decided to increase the number of most used ports using this command:
```bash
sudo nmap -sV --top-ports 10000 $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# --top-ports: scan the 10.000 most frequently used ports. defaults to 1.000.
# -sC: leave this one out right now...
```

This new `nmap` scan gives us the following output:
```bash
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
27017/tcp open  mongodb MongoDB 3.6.8
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Alongside the `ssh` service, a `mongodb` instance is also running on the server! It is a NoSQL (non-relational) database which stores data in json-like documents instead of traditional tables.
After reading the docs of the official MongoDB website, i have stumbled upon [this tool](https://www.mongodb.com/docs/mongodb-shell/). `mongosh` is the tool to interact with local or remote MongoDB deployments! I went onto the download page and copied the link of the Linux package to execute this command:
```bash
curl -O https://downloads.mongodb.com/compass/mongosh-2.8.3-linux-x64.tgz
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -O: write output to file, with its intended name
```

With the `mongosh` utility on my disk, i unzip it using the following command:
```bash
tar -xvzf mongosh-2.8.3-linux-x64.tgz
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# tar: archive utility used on linux systems
# -x: extract files flag
# -v: show the files which are being extracted (verbose)
# -z: use gzip decompression (for .tgz or .tar.gz files)
# -f: specify the file after this flag
```

Now, in the extracted directory, the `mongosh` tool can be found and used in the `/bin` directory!
### Initial Exploitation
The documentation to the `mongodb-shell` also told me how to connect to a remote host, and perform CRUD and aggregation!
I initiate the connection to the server using this command:
```bash
./mongosh mongodb://$IP
```
... but i receive this error message:
```bash
MongoServerSelectionError: Server at 10.10.10.10:27017 reports maximum wire version 6, but this version of the Node.js Driver requires at least 8 (MongoDB 4.2)
```

This indicates a version mismatch, so i go back to the download page and select the version `1.10.6` instead of `2.8.3`, and repeat the steps of installation and unpacking.
With this version it worked! I didn't even need authentication.

As with other databases, there can be multiple databases on the system. The command for showing all available databases is `show dbs` or `show databases`. The available databases on this `mongodb` instance are:
```bash
test> show dbs
admin                   32.00 KiB
config                 108.00 KiB
local                   72.00 KiB
sensitive_information   32.00 KiB
users                   32.00 KiB
```

Now to interact with these `dbs`, we can set them as the current db using the command `use db`. In MongoDB, each database has 0 or more collections. To show the collections from the currently selected db, issue the command `show collections`. To view all `json` values from a collection, one may use `db.<collection>.find()`.

To visualize this approach, the following snippet of code can be seen as a continuation of the code above:
```bash
# use the admin database from now on
test> use admin
switched to db admin
# show me the collections (similar to tables, they hold json values though)
admin> show collections
system.version
# fetch all values from the collection 'system.version' and display them
admin> db.system.version.find()
[ { _id: 'featureCompatibilityVersion', version: '3.6' } ]
```

To get the flag, print all values from the collection `flag` of the database `sensitive_information`!
The `mongosh` installation can now be easily reverted using `rm -rf mongosh-*`!

### Summary

Below is a visualized summary of the exploitation steps used in this machine.

``` mermaid
graph LR
  A[MongoDB<br>service] -->|Unauthenticated<br>access| B[Database<br>access];
```