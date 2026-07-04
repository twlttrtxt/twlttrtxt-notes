The process of joining domains in AD relies heavily on DNS, especially to locate AD services via SRV records. To observe and understand this behavior, `wireshark` can be used.

### Joining a Client to the domain
On the client machine, do these following steps:

1. Run -> Control Panel
2. Navigate to System -> Advanced system settings
3. Open the Computer Name tab
4. Click Change...
5. Select Member of Domain, and enter the domain name `test.local`.

When prompted enter the credentials of `TEST\Administrator`.

### Wireshark investigation
These key processes occur during the client attempt of joining the domain.
#### 1. DNS query for SRV records.
The client contacts the DNS server (DC) and queries the SRV (Service Resource Records) to locate the domain services. These records tell the client where the DC is located and which services are available to him.

#### 2. Kerberos discovery
The client searches for `Kerberos` services using DNS, especially queries for the domain `test.local`. This ensures that the Key Distribution Center is located for authentication.

### Inspecting SRV records
The records can be manually inspected via the Server Manager:

- Open Tools -> DNS.
- Go to Forward Lookup Zones -> `test.local`
- Open the `_tcp` folder.

### Logging into the domain
After the domain join is successful, the client can log in using a domain using the format:
```powershell
TEST\Administrator
```
Or with any other domain user account.