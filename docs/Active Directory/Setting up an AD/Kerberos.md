... is a protocol that provides secure authentication to services over an untrusted network. It gets used when an Active Directory (AD) user wants to interact with a server that provides a service (e.g., SQL, HTTP, FTP). The server is identified by a Service Principal Name (SPN), which acts as the AD alias for the service account.

The usual workflow of using it is as follows:
### 1. Requesting a Ticket Granting Ticket (TGT)

- The AD user communicates with the Domain Controller (DC), which also functions as the Key Distribution Center (KDC), and requests a Ticket Granting Ticket (TGT) (using a `AS-REQ: Request TGT`).
- The user's password is converted into an NTLM hash, and a timestamp is encrypted using this hash. The encrypted timestamp is then sent to the DC for authentication.
- The Domain Controller verifies the user's identity and determines what the user is allowed to do (group memberships, restrictions, permissions, etc.) before creating the TGT.

### 2. Receiving the TGT

- If authentication is successful, the Domain Controller returns the TGT (using `AS-REP: Receive TGT`).
- The TGT is encrypted and digitally signed. Only the Kerberos Ticket Granting Service (`KRBTGT`) can decrypt and read the ticket.

### 3. Requesting a Ticket Granting Service (TGS) Ticket

- The AD user now wants to access a specific server and therefore requests a Ticket Granting Service (TGS) ticket (`TGS-REQ: Present TGT, Request TGS`).
- The user presents the previously obtained TGT to the Domain Controller.
- The DC decrypts the TGT and validates the Privilege Attribute Certificate (PAC) checksum.
- If the TGT is valid and the checksum matches, the DC copies the relevant information from the TGT to create a new TGS ticket.

### 4. Receiving the TGS

- The Domain Controller returns the Ticket Granting Service (TGS) ticket (`TGS-REP`).
- The TGS is encrypted using the NTLM password hash of the target service account and is then sent to the user.

### 5. Accessing the Service

- The AD user presents the TGS to the target server in order to access the requested service (`AP-REQ: Present TGS for Access`).
- The server decrypts the TGS using its own NTLM password hash and validates the ticket.

### 6. Server response

- If the ticket is valid, the server responds with an Application Reply (`AP-REP`).
- This step is only required when mutual authentication between the client and server is enabled.

## SPN-Scanning
Authenticated AD users can search Active Directory for accounts that have a Service Principal Name (SPN) registered. This allows attackers to identify all service accounts that support Kerberos authentication and determine which services they provide. Service accounts that are members of highly privileged groups (e.g., Domain Admins) are particularly valuable targets and can also be enumerated.

## Kerberoasting
In a Kerberoasting attack, an attacker requests a Kerberos service ticket (TGS) for the target account's SPN. To perform this attack, the attacker must first possess a valid Ticket Granting Ticket (TGT) from a legitimate domain user. To enumerate all user accounts that have an SPN assigned, the following `impacket` command can be used:
```bash
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC_IP> -request
```