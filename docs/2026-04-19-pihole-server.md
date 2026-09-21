# Pi-hole Home Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install Pi-hole on the Omarchy Linux laptop as a DNS sinkhole, configure the Windows PC to use it, and publish the project to GitHub as a portfolio piece.

**Architecture:** Pi-hole runs directly on Omarchy (192.168.12.110), intercepting DNS queries from the Windows PC (192.168.12.143). Blocked domains return a dead address; allowed domains are forwarded to Google DNS (8.8.8.8). The Pi-hole web dashboard runs on port 80 of Omarchy.

**Tech Stack:** Pi-hole, Pi-hole FTL (built-in web server, no lighttpd on v6), UFW, SSH, Git, GitHub

---

## File Structure

```
F:/IT_Proj/pihole-server/
  README.md       # portfolio documentation
  notes.md        # commands reference and troubleshooting log
  .gitignore
```

---

## Phase 1 — Server Prep (IT Skills)

**Goal:** Get Omarchy ready to run Pi-hole — updated system, no port conflicts, firewall open.

---

### Task 1: Verify SSH Access

- [x] Open PowerShell on Windows and SSH into Omarchy:
```
ssh sergi@192.168.12.110
```
Expected: you see your Omarchy shell prompt. If you get a timeout, both machines must be on the same WiFi network and SSH must be running (`sudo systemctl start sshd`).

- [x] Confirm you're on the right machine:
```bash
hostname
```
Expected: prints your Omarchy machine name.

- [x] Leave this SSH session open — you'll use it for all remaining Linux steps.

---

### Task 2: Update the Omarchy System

**What you're learning:** Always update before installing server software. This ensures you get the latest security patches and the installer doesn't fail due to outdated dependencies.

- [x] Update all packages:
```bash
sudo pacman -Syu
```
When prompted `Proceed with installation? [Y/n]`, press `Y`.

Expected: packages download and install. May take 1-5 minutes.

- [x] If the kernel was updated, reboot and reconnect via SSH:
```bash
sudo reboot
```
Then from Windows PowerShell:
```
ssh sergi@192.168.12.110
```

---

### Task 3: Check for Port 53 Conflicts

**What you're learning:** Port 53 is the DNS port. On modern Linux, `systemd-resolved` often occupies it. Pi-hole needs exclusive use of port 53 or it will fail to start.

- [ ] Check if anything is using port 53:
```bash
sudo ss -tlnp | grep ':53'
```

- [ ] If you see output containing `systemd-resolve`, disable it:
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

- [ ] Remove the symlink that points `/etc/resolv.conf` to systemd-resolved:
```bash
sudo rm /etc/resolv.conf
```

- [ ] Create a plain `/etc/resolv.conf` pointing to Google DNS (temporary, until Pi-hole takes over):
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

- [ ] Verify port 53 is now free:
```bash
sudo ss -tlnp | grep ':53'
```
Expected: no output.

- [ ] If port 53 was already free (no output in the first check), skip the steps above — nothing to fix.

---

### Task 4: Open Firewall Ports for Pi-hole

**What you're learning:** Pi-hole needs two ports open: 53 for DNS queries and 80 for its web dashboard. Scope both to the LAN subnet, not `Anywhere` — an
open recursive DNS resolver on the public internet is a known amplification-attack
reflector, and the dashboard has no auth in front of it by default.

- [ ] Open DNS ports, scoped to your LAN (both TCP and UDP — DNS uses both):
```bash
sudo ufw allow from 192.168.12.0/24 to any port 53 proto tcp
sudo ufw allow from 192.168.12.0/24 to any port 53 proto udp
```

- [ ] Open the web dashboard port, scoped to your LAN:
```bash
sudo ufw allow from 192.168.12.0/24 to any port 80 proto tcp
```

- [ ] Verify all three rules are active:
```bash
sudo ufw status
```
Expected output includes:
```
53/tcp                     ALLOW IN    192.168.12.0/24
53/udp                     ALLOW IN    192.168.12.0/24
80/tcp                     ALLOW IN    192.168.12.0/24
```

**Phase 1 complete.** Omarchy is ready to install Pi-hole.

---

## Phase 2 — Pi-hole Installation

**Goal:** Install Pi-hole and confirm the dashboard is reachable from Windows.

---

### Task 5: Run the Pi-hole Installer

**What you're learning:** Pi-hole's automated installer handles the full setup — downloading blocklists, configuring the DNS resolver (FTL), and setting up the web dashboard.

- [ ] The official curl-installer does not support Arch (pacman) — install via AUR instead:
```bash
yay -S pi-hole-ftl-bin
yay -S pi-hole-core pi-hole-web
```

Pi-hole v6 includes its own built-in web server (FTL serves the dashboard
directly on ports 80/443) — lighttpd is not required and not installed by
this path.

- [ ] Create the log directory FTL expects (Pi-hole v6 on Arch doesn't create
this automatically and will fail to start dnsmasq without it):
```bash
sudo mkdir -p /var/log/pihole
sudo touch /var/log/pihole/pihole.log
sudo chown -R pihole:pihole /var/log/pihole
```

- [ ] Set the upstream DNS resolver (Pi-hole v6 installs with `upstreams = []`
by default, which accepts queries but times out on all of them). Edit
`/etc/pihole/pihole.toml`:
```toml
upstreams = [
  "8.8.8.8",
  "8.8.4.4"
]
```

- [ ] Start Pi-hole and set an admin password:
```bash
sudo systemctl enable --now pihole-FTL
sudo pihole setpassword
```

Expected: `pihole status` shows DNS service running and blocking enabled.

---

### Task 6: Verify Pi-hole is Running

- [ ] Check Pi-hole's DNS service is active:
```bash
pihole status
```
Expected:
```
  [✓] DNS service is running
  [✓] Pi-hole blocking is enabled
```

- [ ] Check the web server is running (v6 serves the dashboard from FTL itself, not lighttpd):
```bash
sudo systemctl status pihole-FTL
```
Expected: `Active: active (running)`.

- [ ] From your **Windows PC**, open a browser and go to:
```
http://192.168.12.110/admin
```
Expected: Pi-hole admin dashboard loads with a login page.

- [ ] Log in using the password from the installer. Expected: you see the Pi-hole dashboard with query stats.

---

### Task 7: Set a Permanent Admin Password (Optional but Recommended)

- [ ] On Omarchy, set a password you'll remember:
```bash
pihole -a -p
```
Enter and confirm your new password when prompted.

**Phase 2 complete.** Pi-hole is installed and the dashboard is accessible from Windows.

---

## Phase 3 — Client Configuration

**Goal:** Point your Windows PC's DNS at Pi-hole and verify ad blocking works.

---

### Task 8: Change Windows DNS Settings

**What you're learning:** DNS settings tell your computer where to send name-lookup requests. Changing this to Pi-hole's IP makes all your browser traffic go through Pi-hole's blocklist.

- [ ] On Windows, press `Win + R`, type `ncpa.cpl`, press Enter. This opens Network Connections.

- [ ] Right-click your active network adapter (WiFi or Ethernet) → **Properties**.

- [ ] Select **Internet Protocol Version 4 (TCP/IPv4)** → click **Properties**.

- [ ] Select **"Use the following DNS server addresses"** and enter:
  - Preferred DNS server: `192.168.12.110`
  - Alternate DNS server: `8.8.8.8` (fallback if Pi-hole is unreachable)

- [ ] Click **OK** → **OK** to save.

- [ ] Flush the DNS cache so Windows uses the new settings immediately. Open PowerShell and run:
```
ipconfig /flushdns
```
Expected: `Successfully flushed the DNS Resolver Cache.`

---

### Task 9: Test Ad Blocking

**What you're learning:** Verifying that your change actually works — a core IT skill. Never assume a config change worked; always test it.

- [ ] Open a browser on Windows and go to a site known for heavy ads (e.g. a news aggregator or free streaming site). Ads should not load or should appear as blank spaces.

- [ ] Go to the Pi-hole dashboard at `http://192.168.12.110/admin` and click **Query Log** in the left sidebar.

Expected: you see live DNS queries from your Windows PC. Blocked queries appear in red with a domain name and `BLOCKED` status.

- [ ] On Windows PowerShell, confirm DNS is resolving through Pi-hole:
```
nslookup google.com
```
Expected output includes:
```
Server:  192.168.12.110
Address:  192.168.12.110
```
That `Server:` line confirms your Windows PC is using Pi-hole as its DNS.

- [ ] Test that a known ad domain is blocked:
```
nslookup doubleclick.net
```
Expected: returns `0.0.0.0` or `::` — Pi-hole is sinkholing the domain.

---

### Task 10: Verify Blocking Stats

- [ ] Go to Pi-hole dashboard → main page. After a few minutes of browsing, you should see:
  - **Total queries** — number of DNS lookups made
  - **Queries blocked** — percentage blocked (typically 10–30% is normal)
  - **Domains on blocklist** — should show 88,000+ (default Pi-hole v6 list)

- [ ] Click **Top Blocked Domains** to see which ad networks are being blocked most.

**Phase 3 complete.** Your Windows PC is routing DNS through Pi-hole and blocking ads.

---

## Phase 4 — Portfolio (Git + Documentation)

**Goal:** Publish the project to GitHub with documentation.

---

### Task 11: Create Project Folder and Notes File

- [ ] On Windows PowerShell, create the project folder:
```
mkdir F:\IT_Proj\pihole-server
cd F:\IT_Proj\pihole-server
```

- [ ] Create `notes.md` and fill it in as you go. Start with this template (replace bracketed values with your actual info):
```markdown
# Pi-hole Server Notes

## Environment
- Server: Omarchy (192.168.12.110) — Arch Linux
- Client: Gaming PC (192.168.12.143) — Windows 11
- Pi-hole dashboard: http://192.168.12.110/admin

## Key Commands

### On Omarchy (via SSH)
# Check Pi-hole status
pihole status

# Update blocklists
pihole -g

# Check DNS port
sudo ss -tlnp | grep ':53'

# Check firewall
sudo ufw status

# Restart Pi-hole
pihole restartdns

### On Windows
# Flush DNS cache
ipconfig /flushdns

# Check which DNS server is being used
nslookup google.com

# Test a blocked domain
nslookup doubleclick.net

## Troubleshooting
- If Pi-hole stops working: ssh into Omarchy and run `pihole restartdns`
- If dashboard unreachable: check `sudo systemctl status pihole-FTL`
- If port 53 conflict: check `sudo ss -tlnp | grep ':53'`
```

---

### Task 12: Initialize Git and Create .gitignore

- [ ] In PowerShell, in your project folder:
```
cd F:\IT_Proj\pihole-server
git init
```

- [ ] Create `.gitignore`:
```
__pycache__/
*.pyc
.env
```
Save as `F:\IT_Proj\pihole-server\.gitignore`

- [ ] Stage and commit:
```
git add .
git commit -m "feat: initial Pi-hole server documentation"
```
Expected: commit succeeds, lists `notes.md`, `.gitignore`.

---

### Task 13: Create GitHub Repo and Push

- [ ] Go to github.com → sign in → click `+` → **New repository**
  - Name: `pihole-server`
  - Description: `Pi-hole DNS sinkhole home server — Arch Linux, UFW, ad blocking`
  - Set to **Public**
  - Do NOT initialize with README
  - Click **Create repository**

- [ ] In PowerShell, run the commands GitHub shows you:
```
git remote add origin https://github.com/sergioacosta-dev/pihole-server.git
git branch -M main
git push -u origin main
```
Expected: code pushed, live at `github.com/sergioacosta-dev/pihole-server`

---

### Task 14: Write the README

**Note:** This task's original draft is superseded — the repo's actual
`README.md` is the source of truth for setup steps, stack, and blocklist
count. Don't duplicate its content here; edit `README.md` directly and keep
this plan pointing at it, to avoid the two drifting out of sync again.

- [ ] Commit and push the README:
```
git add README.md
git commit -m "docs: add README with architecture and setup"
git push
```

- [ ] Take a screenshot of your Pi-hole dashboard and add it to the GitHub repo (drag and drop into the README editor on github.com).

---

## Success Checklist

- [ ] Pi-hole dashboard accessible at `http://192.168.12.110/admin` from Windows browser
- [ ] `nslookup google.com` shows `Server: 192.168.12.110`
- [ ] `nslookup doubleclick.net` returns `0.0.0.0`
- [ ] Pi-hole dashboard shows live query log with blocked entries
- [ ] Queries blocked percentage is above 0%
- [ ] Public GitHub repo live at `github.com/sergioacosta-dev/pihole-server` with README and screenshot

---

## What Comes Next

**IT:** Set a static local IP on Omarchy so Pi-hole always stays at `192.168.12.110` even after reboots. Add more blocklists (e.g. OISD, Steven Black).

**Networking:** Add local DNS records in Pi-hole so you can reach services by name (e.g. `homewatch.local` → `192.168.12.110`).

**Development:** Build a simple status page that shows Pi-hole stats alongside HomeWatch scan results — combining both projects.
