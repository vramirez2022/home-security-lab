# Home Security Lab: pfSense Guest Segmentation + Suricata IDS + WireGuard VPN (Hyper-V)

A home-lab firewall build that proves three security skills:

1. **Network segmentation** — a guest network that reaches the internet but cannot touch my LAN/VMs.
2. **Intrusion detection** — Suricata watching LAN + guest traffic and alerting on scans/attacks.
3. **Secure remote access** — a WireGuard VPN so I can tunnel into the lab from anywhere.

> All IP addresses and MAC addresses below are **placeholders** (RFC1918/RFC5737-style ranges). Real lab addressing has been redacted.

---

## 1. Topology

```
                              [ INTERNET ]
                                   |
                        [ Home Router / NAT ]
                                   |  WAN  10.X.X.X/24
                            [ pfSense VM ]
                 /                 |                  \
                /                  |                   \
   LAN 10.X.X.X/24           WGT 10.X.X.X/24       Guest 10.X.X.X/24
        |                          |                       |
   [ Lab VMs ]              [ VPN clients: phone/Mac ]  [ Guest VM ]
```

- **WAN** = hn0, DHCP from home router (`10.X.X.X`)
- **LAN** = hn1, `10.X.X.X/24` (trusted)
- **Guest Net (OPT1)** = hn2, `10.X.X.X/24` (isolated)
- **WGT (OPT2)** = WireGuard tunnel, `10.X.X.X/24`
- All on separate Hyper-V virtual switches (segmentation mimicked with vSwitches instead of physical VLANs — same firewall skills)

---

## 2. What I built and why

Hypothesis-to-prove: `guests can browse the internet but must never reach my lab dev machines, and I want to (a) detect malicious activity and (b) reach my lab securely from my phone.`

**Answer: three network "rooms" carved out of one pfSense VM.**

| Interface    | Purpose            | Subnet          |
| ------------ | ------------------ | --------------- |
| WAN          | internet           | 10.X.X.X (DHCP) |
| LAN          | trusted lab        | 10.X.X.X0/24    |
| Guest (OPT1) | untrusted visitors | 10.X.X.X/24     |
| WGT (OPT2)   | VPN clients        | 10.X.X.X/24     |

pfSense's default posture is _deny_: nothing crosses between rooms unless a firewall rule says so. Everything below is me writing those rules.

---

## 3. Configuration summary

### Guest isolation (Firewall -> Rules -> Guest Net)

Rules are evaluated top-down, so the isolation **blocks come first** and the allow rules after — otherwise a broad allow rule would match LAN-bound traffic before the block is ever reached.

| #   | Action | Protocol | Source       | Destination | Port  | Purpose                |
| --- | ------ | -------- | ------------ | ----------- | ----- | ---------------------- |
| 1   | Block  | any      | Guest subnet | 10.X.X.X/24 | -     | LAN isolation          |
| 2   | Block  | any      | Guest subnet | 10.X.X.X/24 | -     | block WAN-side lab net |
| 3   | Pass   | UDP      | Guest subnet | firewall    | 67-68 | DHCP                   |
| 4   | Pass   | TCP/UDP  | Guest subnet | Any         | 53    | DNS                    |
| 5   | Pass   | any      | Guest subnet | any         | -     | internet               |

Guest DHCP pool: `10.X.X.X - .100`, with a static reservation for the guest VM -> `10.X.X.X`.

### Suricata IDS (Services -> Suricata)

- Running on **LAN** and **Guest Net** interfaces (IDS mode, alerts-only).
- Rule set: Emerging Threats OPEN (full set), daily auto-update.

### WireGuard VPN (VPN -> WireGuard)

- Tunnel `tun_wg0`: port XXXXX, `10.X.X.X/24`, MTU 1420.
- Peers: iPhone (`10.X.X.X`), Mac (`10.X.X.X`).
- Firewall: WGT pass rule (`10.X.X.X./24 -> any`), WAN pass UDP XXXX.
- **Manual outbound NAT rule** (`10.X.X.X./24 -> WAN`) so VPN clients get internet.
- Home router: port-forward UDP XXXX -> pfSense WAN IP.
- Client configs and private keys are **not committed** (see `.gitignore`).

---

## 4. Proof

| Claim                      | Evidence                                                                               |
| -------------------------- | -------------------------------------------------------------------------------------- |
| Guest can reach internet   | `ping 8.8.8.8` OK from guest                                                           |
| Guest CANNOT reach LAN     | `ping 10.X.X.X` -> blocked, logged in System Logs                                      |
| Suricata detects attacks   | nmap scan from guest shows ET scan alerts                                              |
| VPN connects from cellular | iPhone "Handshake completed", internet via tunnel, pfSense Status shows peer + traffic |

---

## 5. What I learned

- **Firewall rule ordering matters** — block rules must sit before the allow-all, or segmentation silently fails.
- **NAT vs routing vs segmentation** — uploads can flow while downloads die if the outbound NAT mapping is missing.
- **A WireGuard VPN needs a route, not just a tunnel** — clients could handshake but got no reply until the interface had a static IP and pfSense owned the `10.10.10.0/24` route.
- **IDS vs IPS** — Suricata alerts first; blocking (IPS) is a tuning decision made _after_ watching real alerts, to avoid false-positive lockdowns.
- **Home NAT reality** — "connect from anywhere" needs a real router port-forward + public IP (and Dynamic DNS for dynamic IPs).
- **Never commit secrets** — a WireGuard config contains a private key; those files are git-ignored and the key was rotated.

---

## 6. Files

- `pfsense-setup.md` - full build notes (interfaces, DHCP, rules, Suricata, VPN, backups)

> Note: WireGuard client `.conf` files (containing private keys) are intentionally excluded from this repository.

---

_Build run on a Hyper-V lab, pfSense CE, Fedora guest VM, iPhone/macOS WireGuard clients._
