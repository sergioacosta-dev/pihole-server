# Pi-hole Home Server

A self-hosted DNS sinkhole that blocks ads and tracking domains, built as a home lab project covering IT and networking fundamentals.

## What It Does

- Intercepts DNS queries from devices on the local network
- Blocks requests to known ad and tracking domains using an 88,000+ domain blocklist
- Returns a dead address for blocked domains so ads never load
- Provides a web dashboard showing live query logs and blocking statistics

## Architecture

```
Windows PC (192.168.12.143)
  --> DNS query --> Pi-hole (192.168.12.110:53)
                     --> blocked? --> return 0.0.0.0
                     --> allowed? --> forward to 8.8.8.8 --> return real IP
```

## Tech Stack

- Pi-hole v6 (DNS sinkhole + blocklist management)
- Pi-hole FTL (built-in DNS resolver and web server)
- UFW (firewall)
- Arch Linux (server OS)

## Setup

### Prerequisites

- A Linux machine on your local network with a static local IP
- SSH access from your client machine
- UFW installed and active

### Install Pi-hole (Arch Linux)

The official Pi-hole curl installer does not support Arch. Install via AUR:

```bash
yay -S pi-hole-ftl-bin
yay -S pi-hole-core pi-hole-web
```

### Create the log directory

```bash
sudo mkdir -p /var/log/pihole
sudo touch /var/log/pihole/pihole.log
sudo chown -R pihole:pihole /var/log/pihole
```

### Configure upstream DNS

Edit `/etc/pihole/pihole.toml` and set:

```toml
upstreams = [
  "8.8.8.8",
  "8.8.4.4"
]
```

### Open firewall ports

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Start Pi-hole

```bash
sudo systemctl enable pihole-FTL
sudo systemctl start pihole-FTL
```

Dashboard available at `http://<server-ip>/admin`

### Configure client DNS (Windows)

1. Open Network Connections (`ncpa.cpl`)
2. Right-click adapter → Properties
3. Uncheck IPv6 (prevents IPv6 DNS from bypassing Pi-hole)
4. Select IPv4 → Properties → Use the following DNS:
   - Preferred: `<pi-hole-ip>`
   - Alternate: `8.8.8.8`
5. Run `ipconfig /flushdns`

### Verify

```powershell
nslookup google.com        # Server line should show Pi-hole IP
nslookup doubleclick.net   # Should return 0.0.0.0
```

## What I Learned

- **DNS:** How domain name resolution works, query forwarding, DNS sinkholes
- **IT:** Firewall port management, systemd service configuration, port conflict diagnosis
- **Networking:** How DNS settings affect all traffic on a device, IPv4 vs IPv6 DNS resolution
- **Linux:** AUR package management, systemd-resolved conflicts, SSH-based server administration

## Screenshots

![Pi-hole Dashboard](dashboard.png)
