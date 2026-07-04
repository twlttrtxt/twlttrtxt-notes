They are used in AD to structure objects in a managable way. They are important for applying Group Policies across multiple users and computers.

Typical use-cases are structuring these objects by location (e.g. countries, cities, sites), departments (e.g. IT, finance, HR), or device types (e.g. desktop, mobile, tablet).

### Creating Organisational Units
They can be created in the Server Manager:

1. Go to Tools
2. Select Active Directory Users and Computers
3. Navigate to the domain (e.g. `test.local`)
4. Right-click the domain -> New -> Organisational unit

It is also possible to create nested OUs , such as:

- `Administration`
	- `Desktop`
	- `Mobile`

They should not however exceed the maximum depth of `4-5` levels, as that may impact the manageability or the performance.

### Moving objects into OUs
Objects such as users or computers may then be moved into the desired OUs using Server Manager. Alternatively, a OU can be created and managed by `Powershell` which enables bulk creation or script-based management.