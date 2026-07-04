Using `PowerShell`:
```powershell
$newPassword = ConvertTo-SecureString -String "Pass123!" -AsPlainText -Force
New-ADUser -name Bob -AccountPassword $newPassword -Enabled $TRUE
```
or with the GUI:

- Go to Active Directory Users and Computers
- Click New -> User
- Fill in the user details.

It is common to use a pre-configured disabled admin template account (e.g. `_Admin`). These accounts already include the correct group membership and access permissions. These template accounts can then be copied to create real user accounts.