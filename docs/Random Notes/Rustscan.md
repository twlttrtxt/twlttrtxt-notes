... is a faster alternative to `nmap`

## Installation
```bash
cd ~/Downloads
```
```bash
wget https://github.com/bee-san/RustScan/releases/download/2.4.1/rustscan.deb.zip
```
```bash
unzip rustscan.deb.zip
```
```bash
sudo dpkg -i rustscan_2.4.1-1_amd64.deb
```

## Usage
```bash
rustscan -a <ip> -r 1-10000 -- -sC -sV
```

- Scans a wide range of ports quickly, and hands over the results to `nmap` to do script scanning and version detection!