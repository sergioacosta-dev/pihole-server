## Environment
- Server: Omarchy (192.168.12.110) — Arch Linux
- Client: Gaming_PC (192.168.12.143) — Windows 11
- Pi-hole dashboard: http://192.168.12.110/admin

---

## Installation Notes (2026-04-19)

Pi-hole's official curl installer does not support Arch Linux (pacman).
Install via AUR instead:

```bash
yay -S pi-hole-ftl-bin      # use -bin variant; source build has broken patch URL
yay -S pi-hole-core pi-hole-web
```

Pi-hole v6 includes its own built-in web server — lighttpd is NOT required.
Dashboard is served by pihole-FTL directly on ports 80 and 443.

---

## Port 53 Conflict Fix

Arch Linux runs systemd-resolved by default, which occupies port 53.
Pi-hole needs exclusive use of port 53 — disable systemd-resolved first:

```bash
sudo systemctl stop systemd-resolved-varlink.socket systemd-resolved-monitor.socket
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved-varlink.socket systemd-resolved-monitor.socket
sudo systemctl disable systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Verify port 53 is free:
```bash
sudo ss -tlnp | grep ':53'   # should return no output
```

---

## Upstream DNS Fix

Pi-hole v6 installs with `upstreams = []` by default — it will accept queries
but time out on all of them because it has nowhere to forward requests.

Fix by editing the config file:
```bash
sudo nano /etc/pihole/pihole.toml
```
Find `upstreams = []` and change to:
```toml
upstreams = [
  "8.8.8.8",
  "8.8.4.4"
]
```

---

## Missing Log File Fix

Pi-hole v6 on Arch may fail to start dnsmasq due to a missing log directory:
```
dnsmasq: cannot open log /var/log/pihole/pihole.log: No such file or directory
```

Fix:
```bash
sudo mkdir -p /var/log/pihole
sudo touch /var/log/pihole/pihole.log
sudo chown -R pihole:pihole /var/log/pihole
sudo systemctl restart pihole-FTL
```

---

## Windows DNS Configuration

IPv6 must be disabled on the Windows adapter — otherwise Windows uses
the router's IPv6 DNS instead of Pi-hole's IPv4 address.

Steps:
1. Win + R -> ncpa.cpl
2. Right-click adapter -> Properties
3. Uncheck "Internet Protocol Version 6 (TCP/IPv6)"
4. Select IPv4 -> Properties -> Use the following DNS:
   - Preferred: 192.168.12.110
   - Alternate:  8.8.8.8
5. ipconfig /flushdns

Verify:
```
nslookup google.com       # Server should show: pi.hole / 192.168.12.110
nslookup doubleclick.net  # Should return 0.0.0.0
```

---

## Key Commands

### On Omarchy (via SSH)

```bash
# Check Pi-hole status
pihole status

# Update blocklists
sudo pihole -g

# Set admin password (Pi-hole v6 syntax)
sudo pihole setpassword

# Restart Pi-hole DNS
sudo systemctl restart pihole-FTL

# Check what's listening on port 53
sudo ss -tlnp | grep ':53'

# Check firewall
sudo ufw status

# Test DNS locally
dig @127.0.0.1 google.com
```

### On Windows (PowerShell)

```powershell
# Flush DNS cache
ipconfig /flushdns

# Verify Pi-hole is the DNS server
nslookup google.com

# Test a blocked domain
nslookup doubleclick.net
```

---

## Firewall Ports

```bash
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 53/tcp    # DNS
sudo ufw allow 53/udp    # DNS
sudo ufw allow 80/tcp    # Pi-hole dashboard (HTTP)
sudo ufw allow 443/tcp   # Pi-hole dashboard (HTTPS)
```

---

## Troubleshooting

| Symptom | Check | Fix |
|---------|-------|-----|
| DNS queries timing out | dig @127.0.0.1 google.com | Check upstreams in pihole.toml, check log dir exists |
| Dashboard unreachable | sudo systemctl status pihole-FTL | Restart pihole-FTL |
| Port 53 conflict | sudo ss -tlnp | grep ':53' | Disable systemd-resolved |
| Windows using wrong DNS | nslookup google.com (check Server line) | Disable IPv6 on adapter, flush DNS |
| Blocklist empty | pihole status | Run sudo pihole -g |
