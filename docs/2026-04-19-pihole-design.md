# Pi-hole Home Server — Design Spec

**Date:** 2026-04-19
**Project directory:** `F:/IT_Proj/pihole-server/`

---

## Goal

Install Pi-hole as a DNS sinkhole on the Omarchy Linux laptop, configure the Windows PC to use it as its DNS server, verify network-wide ad blocking, and publish the project to GitHub as a portfolio piece.

---

## Architecture

```
Windows PC (192.168.12.143)
  → DNS query → Pi-hole (192.168.12.110 : port 53)
                  → blocked domain? → return 0.0.0.0 (dead end, ad never loads)
                  → allowed domain? → forward to 8.8.8.8 (Google DNS)
                                        → return real IP to Windows PC
```

Pi-hole runs directly on Omarchy (Arch Linux). No Docker. The Pi-hole web dashboard runs on port 80 of Omarchy and is accessible from Windows at `http://192.168.12.110/admin`.

---

## Components

| Component | Role |
|-----------|------|
| Pi-hole FTL | DNS resolver — intercepts queries, checks blocklist |
| lighttpd | Web server for Pi-hole dashboard (auto-installed) |
| Pi-hole blocklist | Default list of ~300k ad/tracking domains |
| UFW | Firewall — must allow ports 53 (DNS) and 80 (dashboard) |
| Windows DNS settings | Point Windows to 192.168.12.110 instead of router |

---

## Phases

### Phase 1 — Server Prep
- Verify SSH access to Omarchy from Windows
- Update the Omarchy system (`sudo pacman -Syu`)
- Open firewall ports: 53/tcp, 53/udp (DNS), 80/tcp (dashboard)
- Confirm no other service is using port 53

### Phase 2 — Pi-hole Installation
- Run the official Pi-hole installer via SSH
- Set upstream DNS to 8.8.8.8 (Google) during setup
- Confirm Pi-hole dashboard loads at `http://192.168.12.110/admin` from Windows browser

### Phase 3 — Client Configuration
- Change Windows DNS settings to `192.168.12.110`
- Test: visit an ad-heavy site and verify ads are blocked
- Verify queries appear in the Pi-hole dashboard
- Test: restore original DNS settings to confirm Pi-hole is doing the work

### Phase 4 — Portfolio
- Initialize Git repo in `F:/IT_Proj/pihole-server/`
- Write README with architecture, setup steps, and what was learned
- Push to GitHub at `github.com/sergioacosta-dev/pihole-server`

---

## File Structure

```
F:/IT_Proj/pihole-server/
  README.md       # portfolio documentation
  notes.md        # commands reference and troubleshooting log
  .gitignore
```

Pi-hole config lives on Omarchy at `/etc/pihole/` — not tracked in this repo.

---

## Success Criteria

- Pi-hole dashboard accessible at `http://192.168.12.110/admin` from Windows
- Windows PC DNS points to `192.168.12.110`
- Ad-heavy site loads without ads
- Pi-hole dashboard shows live query log with blocked entries
- Public GitHub repo with README and screenshot
