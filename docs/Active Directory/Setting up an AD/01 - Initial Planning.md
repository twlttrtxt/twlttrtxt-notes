- Gather basic information about the desired active directory, such as company name, number of employees or external domain name.
- Determine an IP address range with enough IP-addresses for the employees.
- Choose an internal AD domain name. It should match the external domain name, if there will be an integration in Microsoft 365.

If active directory was set up using VMWare:

- Navigate to Settings -> Network and create a virtual network.
- Allow NAT and connect your host computer to the network.
- After creating the network, take note of the assigned IP range (e.g. `172.16.67.0/24`).

- Take note of the `Gateway IP address` (typically ends with `.2`).
- The address range of `172.16.67.0-9` should be reserved for network infrastructure and potential gateways
- Assign domain controllers starting at `.10`.
- Configure `DHCP` to give IP-addresses to users.

This leads to the following common addressing scheme:

- `.10 - .19`: Domain Controllers
- `.20 - .29`: Member Servers (SQL Servers, web servers, ...)
- `.30 - .39`: Network Infrastructure (DNS, ...)
- `.100 - .129`: DHCP pool for AD users

To configure a static IP address for a domain controller, the following steps may be used on the DC machine itself:

1. Open Network Connections.
2. Select Ethernet.
3. Open Internet Protocol Version 4 (TCP/IPv4).
4. Select Use the following IP address.
5. Assign a static IP address (e.g. `172.16.67.10`)
6. Configure the default gateway (e.g. `172.16.67.2`)
7. Configure the DNS server (typically, the AD itself)