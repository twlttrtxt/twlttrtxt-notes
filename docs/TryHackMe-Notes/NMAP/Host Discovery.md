 (`-sn for no portscan`)
 
- ARP Scans: for hosts in the local network 

```bash
sudo nmap -PR -sn MACHINE_IP/24
```

- ICMP Echo Scan: ping scans, might be blocked by firewall
- Default to ARP scans before (if reachable in local network)

```bash
sudo nmap -PE -sn MACHINE_IP/24
```

- ICMP Timestamp request: might not be blocked by firewall

```bash
sudo nmap -PP -sn MACHINE_IP/24
```

- ICMP Adress Mask Scan: Alternative way to scan with icmp:

```bash
sudo nmap -PP -sn MACHINE_IP/24
```

- TCP SYN Ping: Check if host is up by sending SYN requests to common ports

```bash
sudo nmap -PS22,80,443 -sn MACHINE_IP/30
```

- TCP ACK Ping: Check if host is up by sending ACK requests to common ports (requires root)

```bash
sudo nmap -PA22,80,443 -sn MACHINE_IP/30
```

- UDP Ping Scan: Check if host is up by sending UDP packets to common ports

```bash
sudo nmap -PU53,161,162 -sn MACHINE_IP/30
```