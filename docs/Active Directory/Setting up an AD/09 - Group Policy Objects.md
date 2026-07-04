A GPO is a set of rules and configurations used in AD to manage users and computers. GPOs are used to enforce settings such as power-saving options, default browser, desktop background, security policy, and so on. They allow admins to apply configurations across groups of users or computers.

A GPO typically consists of two parts:

- `Computer Config`: Applies to machines regardless of who logs in. (e.g. `Firewall rules`)
- `User Config`: Applies to user accounts (e.g. `Desktop background`)

A domain usually has a default domain policy which is automatically created and linked to the domain (e.g. `test.local`). It shouldn't be modified carelessly, as it affects everything. This is the only policy which can enforce a password policy (next to password security objects).

### Processing order
They are typically applied in a specific order known as `LSDOU`:

- `L`: Local policies
- `S`: Site policies
- `D`: Domain policies
- `OU`: Organisational unit policies

This means that policies applied later in the order overwrite newer ones:

- Local policies are overridden by Site policies
- Site policies are overridden by Domain policies
- Domain policies are overridden by OU-policies

This ensures that the most specific policies (e.g. `OU`-level) have the highest priority.

### Managing GPOs
They can be managed in the Server Manager:

- Go to Tools -> Group Policy Management
- Navigate to Forest -> Domains -> `test.local`
- Expand Group Policy Objects

To create a new GPO:

- Right-click Group Policy Objects
- Select New

This creates a GPO but it won't be active until linked to a domain, site or organisational unit.

