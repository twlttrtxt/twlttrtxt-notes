This is a best-practice model for structuring security groups in AD environment, especially with multiple domains or forests. It consists of the following blocks:

- `Identity`: User or computer accounts
- `Global Group`: Groups that represent roles or departments
- `Domain Local`: Groups that assign permissions within a domain such as access to resources.
- `Access`: Actual permissions that are assigned to resources.

This model works by separating identity, grouping and permissions:

1. Users are added to a global group.
2. The global group is added to a domain local group.
3. The domain local group is assigned permissions to resources.

This ensures clean separation of roles and permissions and easier administration.

### Example Implementation
The domain `test.local` is acquired by a company called `aplic`, and a trust relationship is established between both domains. Now users from the finance department in `test.local` require access to a shared folder from `alpic`. `IGDLA` provides cross-domain access in a controlled way:

1. Create global group in `test.local` called `alpic_finance`.
2. Create a domain local group in `alpic` called `fileaccess_finance`.
3. Add the `alpic_finance` to the domain local group `fileaccess_finance`.
4. The `fileaccess_finance` group has access to the file share.

### Practical implementation

1. Go to the correct OU
2. Right-click -> New -> Group
3. Create the global security group and the domain local security group.
4. Open domain local security group
5. Go to properties -> members -> add
6. Add the global security group