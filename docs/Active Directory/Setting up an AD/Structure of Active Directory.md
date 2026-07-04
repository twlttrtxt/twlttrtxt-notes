- "Database of Objects (Users, Groups, ...), each having attributes (email, name)" -> Allows Users to find and access specific resources
- Domain Controllers contain master copy of account directories. They are Servers that replicate copies of the database
- `Server Manager` comes preinstalled on windows servers, it allows for the management of AD. -> `Active Directory Users and Computers`
- Domain Names contain different folders -> Builtin Groups & Features. (Organizational Units), for example based on location, departments, ...
- Example: Go to Domain Name -> `New Object - Organizational Unit: Operations` -> `New User` -> `New Group: Managers` (Assign permissions)

### Active Directory Objects:

- `Users`: Tom (age,email,name,login, ...), Alex (...),
- `Resources`: Hardware Elements (PCs, Printers, ...)
- `Groups`: Managing rights for multiple users (Admins, ...)
- `Organizational Units`: Manage AD Objects, hold users, groups, computers, other OUs. Create a hierarchical structure within domain.
- `Group Policy Objects`: Enable restriction of access to folders/executables,... Can be applied to Domains or OU's.

### Forest

- The highest-level container in Active directory.
- Contains one or more domains that share a common configuration.
- A forest can contain multiple root domains and child domains.

### Domain
Primary administrative and authentication boundary within AD. The forest root domain is the first domain created within the forest.

All objects (users, computers, groups, ...) can authenticate against a domain. A domain user can log on to a domain joined computer using their AD credentials. Domains are considered authentication boundaries but not security boundaries, as Administrators can still gain access across domains within the same forest.

### Additional Root domains
A forest can have multiple tree root domains (e.g. `test.local` and `test2.local`). Each root domain starts a new DNS namespace while still belonging to the same AD forest.

### Child domains
... extend the namespace of their parent (e.g. `dev.test.local` extends `test.local`). They inherit a two-way transitive trust relationship with their parent by default.

### Trust relationships
Allow users from one domain or forest to access resources in other domains or forests. They enable resource sharing across different AD environments.

### NetNTLM authentication:

1. Client sends authentication request to server
2. Server replies with a "NTLM Challenge", which is a random string/number
3. Client uses his NTLM Hash on the challenge, and returns that in the response to the server.
4. Server sends the Response of the client & The challenge to the Domain Controller.
5. The Domain Controller uses his known NTLM Hash on the challenge, and compares the result to the result of the client. He then replies to the servers with request, if the authentication was successful or not.
6. The server tells the client if he is successfully authenticated.

### Domain Controller: 

- AD is based on multiple Domain Controllers that synchronize directory data between them, ensuring information consistency throughout forest.
- Handle Communication between users and domains (Authentication)
- Read Only DC: Remote sites to access network resources.


- Interesting Files: On a Domain Controller   

-> `C:\Windows\NTDS\ntds.dit`: Active Directory Database!!!