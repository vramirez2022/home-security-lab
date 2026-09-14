# pfSense Setup

Notes on getting the pfSense firewall up and running.

## Accessing the pfSense GUI from Windows

Most lab setups follow the same flow:

1. Boot the pfSense VM in your hypervisor.
2. Open PowerShell or Command Prompt on your Windows machine and run `ipconfig`.
3. Confirm your virtual network adapter attached to the pfSense LAN switch has an IPv4 address starting with `192.168.1.x` (other than `.1`).
4. If it shows an APIPA address (`169.254.x.x`), your virtual switch or DHCP is misconfigured.
5. Open a browser and navigate to the pfSense web GUI (typically `http://192.168.1.1`).

## Verification

Run `ipconfig` and verify the IPv4 Address falls within `192.168.1.2` to `192.168.1.254`. Otherwise the VM cannot reach the firewall's DHCP and the GUI will not be reachable.

## Interfaces

- WAN = hn0 (10.0.0.x - Hyper-V NAT, treated as internet)
- LAN = hn1 (192.168.1.0/24 - trusted)
- Guest Net / OPT1 = hn2 (192.168.2.0/24 - untrusted guests)

Adding an interface: Interfaces -> Assignments -> pick the NIC -> Add -> configure IPv4 static (e.g. 192.168.2.1/24) -> Save/Apply.

## Guest Network Setup

- Interface: hn2 assigned as OPT1, IP 192.168.2.1/24
- DHCP Server (Services -> DHCP Server -> Guest Net):
  - Enabled, Address Pool Range: 192.168.2.2 - 192.168.2.100
  - Gateway: 192.168.2.1, DNS: 1.1.1.1 / 8.8.8.8
  - Static Mapping: MAC 00:15:5d:00:e9:69 -> 192.168.2.101 (guest-vm)
    - Note: pfSense requires static mapping IPs to be OUTSIDE the pool range
- Fedora guest VM: DHCP works, got 192.168.2.101. SSH enabled (sshd) for remote access.

## Firewall Rules

Rules are evaluated top-to-bottom. Guest Net rules:

| # | Action | Protocol | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | Guest Net subnet | Firewall self | 67-68 | DHCP |
| 2 | Pass | TCP/UDP | Guest Net subnet | Firewall self | 53 | DNS |
| 3 | Pass | any | Guest Net subnet | any | - | Internet |
| 4 | Block | any | Guest Net subnet | LAN subnet (192.168.1.0/24) | - | Isolate LAN |
| 5 | Block | any | Guest Net subnet | 10.0.0.0/24 | - | Block WAN-side lab net |

- LAN interface uses the default pfSense allow-all rule (LAN subnet -> any).
- WAN: any requirement plus a Pass UDP 51820 rule for WireGuard when the VPN is set up.

Verified: guest can reach internet (8.8.8.8 + gateway), LAN pings (192.168.1.1/.2) are blocked and logged.

## Suricata (IDS)

- Installed package, engaged on LAN and Guest Net interfaces (WAN optional - heaviest edge to inspect).
- Both run in IDS mode (blocking disabled - alert only). IPS mode deferred until alerts are tuned.
- Rule categories: all Emerging Threats Open categories selected.
- Rule Update Interval: set to Daily.
- Alerts: watch Services -> Suricata -> Alerts after generating traffic.
- Heavy on CPU/memory with 2+ instances; monitor Status -> Dashboard.

## WireGuard VPN (working)

Server side:
- Tunnel tun_wg0: listen 51820, address 10.10.10.1/24, MTU 1420
- Interface assigned as WGT (opt2) with IPv4 config NONE
- IMPORTANT FIX: WGT interface must get a STATIC IPv4 (10.10.10.1/24) so pfSense installs the route 10.10.10.0/24 via tun_wg0. Without it, clients connect (RX/TX climb) but return traffic has no route and nothing loads.
- Peers: vr_iphone (10.10.10.2), vr-mac (10.10.10.3, key in wg-vr-mac.conf) - Allowed IPs -/32 + 192.168.1.0/24 + 192.168.2.0/24
- Firewall: WGT Pass rule (source 10.10.10.0/24 -> any); WAN Pass UDP 51820
- NAT: Manual outbound. Rule WAN <- 10.10.10.0/24 (mapping must exist for internet)

Client (iPhone):
- Interface: Address 10.10.10.2/24, DNS 1.1.1.1, MTU 1420
- Peer: pfSense server public key, Endpoint 73.100.240.182:51820 (home public IP), AllowedIPs 0.0.0.0/0 (full tunnel)
- "Exclude private IPs" toggle must be OFF

Home router (Xfinity app): Port forward UDP 51820 -> 10.0.0.28 (pfSense WAN).
Note: home public IP is dynamic - switch to Dynamic DNS later.

Verified: internet browsing works from phone over cellular tunneled through pfSense.

## Backups

- Diagnostics -> Backup / Restore -> Download configuration as XML (full config).
- Automatic config backup can be enabled under System -> Advanced.
- Restore: same page, browse XML, Restore.

## Sharing Files to Windows (Hyper-V)

- No shared folders in Hyper-V. Use SCP from Windows (Linux has SSH; pull the file):
  - scp vramirez@<linux-ip>:/path/file C:\Users\vero\Documents\
- Or SMB share mount (//10.0.0.230/Users/vero/Documents via cifs).

## Portfolio / Dashboard Hosting (pfsense.veronica-ramirez.com)

Intent: display the pfSense dashboard under `pfsense.veronica-ramirez.com`, configured/hosted on Netlify.

Question: is it okay to use this approach?

Answer: Decided NOT to use Netlify for a live dashboard. Use the project documentation approach below instead.

- The pfSense CE license does not permit exposing the web interface directly to the public internet.
- Publishing the live GUI is also a serious security risk (open admin interface to the internet).

Recommended alternatives:
- Host a static portfolio page showing screenshots/recordings and write-ups of the firewall setup.
- For personal remote access to the real GUI, connect through a VPN (WireGuard or OpenVPN) instead of exposing the GUI publicly.

## How to showcase this project in a portfolio (chosen approach)

A home-lab firewall project is best shown through documentation, not a live dashboard. Strong portfolio write-ups cover:

1. Problem statement - what you wanted to build and why (firewall, segmentation, ad-blocking, VPN, etc.).
2. Architecture diagram - use draw.io or Excalidraw: pfSense VM, LAN/WAN/OPT interfaces, switches, VLANs, devices.
3. Configuration details - firewall rules, NAT, DHCP, VLANs, DNS/DHCP reservations, package installs.
4. Screenshots - Dashboard, Firewall Rules, Interfaces, System Logs, traffic/bandwidth graphs. Anonymize any sensitive data.
5. Testing & verification - show evidence: ipconfig within 192.168.1.2-254, ping results, rule-blocked vs allowed traffic, system log entries proving a rule worked.
6. Results & lessons learned - what worked, what broke, how you fixed it.

Format:
- One markdown page per project (portfolios/GitHub accept Markdown well) OR a dedicated portfolio site (GitHub Pages instead of Netlify is a good free option for a static showcase).
- Keep each section short with real screenshots. Recruiters skim.

The key is evidence: screenshots + a written walkthrough prove you built it - a live link is not required.