After installing Active Directory Domain Services, administrative tools become available through Server Manager -> Tools.

### Active Directory Users and Computers
This is the primary administration tool for managing objects in a domain. Common tasks may include:

- Creating and managing users and or groups.
- Creating and organizing organizational units.
- Managing computer accounts (resetting passwords, enabling-, disabling accounts).
- Assigning users to security groups

### Active Directory Administrative Center
A GUI for managing AD, with features such as:

- User, group and OU management.
- Password policy management.
- AD recycle bin management.
- PowerShell integration.

### Active Directory Domains and Trusts
Used to manage:

- Domain and forest trust relationships.
- `UPN` (User Principal Name) suffixes.
- Domain functional levels.

Most commonly used when integrating AD with a public domain name or Microsoft 365.

### Active Directory Sites and Services
Represents the organization's physical network topology, including tasks such as:

- Creating AD Sites.
- Mapping IP sub-nets to sites.
- Managing replication between DC's.
- Configuring replication schedules and site links.

Sites usually represent a physical location of a company (e.g. `Munich, Stuttgart, ...`).