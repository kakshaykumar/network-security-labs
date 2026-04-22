# Lab 03 — NAT & VPN: Observing IP Address Behavior

**Tools:** macOS Terminal (`ifconfig`), GMU VPN (OpenVPN-based), whatismyipaddress.com
**Objective:** Document local and public IP addresses before and after connecting to a VPN, and analyze what the changes reveal about NAT and VPN tunnel behavior

---

## Setup

Used the built-in `ifconfig` command on macOS to inspect network interface configuration. Public IP address was verified using `whatismyipaddress.com`, which also returns ISP, ASN, and geolocation data. Connected to George Mason University's VPN and repeated both checks.

---

## Before VPN Connection

### Local Network Configuration (`ifconfig` output)

```
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
     inet 192.168.1.152 netmask 0xffffff00 broadcast 192.168.1.255
     status: active
```

**Extracted values:**
| Parameter | Value |
|---|---|
| Local IP | `192.168.1.152` |
| Subnet Mask | `255.255.255.0` |
| Prefix Length | `/24` |
| Default Gateway | `192.168.1.1` |
| Broadcast | `192.168.1.255` |

`192.168.1.152` is an RFC 1918 private address in the Class C range (`192.168.0.0/16`). It was dynamically assigned by the home router's DHCP server (the same host scanned in Lab 01). The /24 subnet means 254 usable host addresses on this network segment.

### Public IP Address

| Parameter | Value |
|---|---|
| Public IP | `173.66.13.41` |
| ISP | Verizon Business |
| ASN | 701 |
| Location | Fairfax, VA |

`173.66.13.41` is the address that the rest of the internet sees when my device makes a connection. This is the IP assigned to my home router's WAN interface by Verizon. My router is performing NAT — translating my private `192.168.1.152` address to this single public IP for all outbound traffic.

ASN 701 belongs to Verizon Business's network. ASNs are used in BGP routing to identify autonomous systems — large blocks of IP addresses managed by a single organization.

---

## After VPN Connection

### Local Network Configuration (`ifconfig` output)

```
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
     inet 10.151.204.231 netmask 0xffff8000 broadcast 10.151.255.255
     status: active
```

**Extracted values:**
| Parameter | Value |
|---|---|
| Local IP | `10.151.204.231` |
| Subnet Mask | `255.255.128.0` |
| Prefix Length | `/17` |
| Default Gateway | `10.151.255.1` |

`10.151.204.231` is a different RFC 1918 address — now in the `10.0.0.0/8` Class A private range rather than the home network's `192.168.0.0/16`. The VPN server assigned this address from its own internal address pool, placing my device inside GMU's virtual network. The /17 subnet is much larger than my home /24 — it covers 32,766 usable addresses, consistent with a university's VPN pool serving thousands of students and staff.

### Public IP Address

| Parameter | Value |
|---|---|
| Public IP | `129.174.182.103` |
| ISP | George Mason University |
| ASN | GMU's ASN |
| Location | Arlington, VA |

My traffic is now egressing through GMU's network infrastructure. From the perspective of any web server I connect to, I'm coming from George Mason University in Arlington — not from a Verizon residential connection in Fairfax.

---

## Analysis

### What Changed and Why

**Local IP change (`192.168.1.152` → `10.151.204.231`)**

When the VPN connects, it creates a virtual network interface (`utun` on macOS) and the VPN server assigns a new private IP address from its own subnet. My device now has two private IPs simultaneously — the original `192.168.1.152` for local network traffic, and `10.151.204.231` for VPN-tunneled traffic. The VPN client routes internet-bound traffic through the tunnel interface, so applications use the VPN-assigned address for all external connections.

**Public IP change (`173.66.13.41` → `129.174.182.103`)**

Before VPN: `my device (192.168.1.152)` → `home router NAT (173.66.13.41)` → internet

After VPN: `my device (192.168.1.152)` → `encrypted tunnel` → `GMU VPN server (129.174.182.103)` → internet

The VPN encapsulates my traffic in an encrypted tunnel to GMU's VPN server. GMU's server decrypts it and forwards it to the internet using GMU's own public IP. From the destination's perspective, the request came from GMU.

### NAT Implications

NAT is doing two separate jobs in this setup:

1. **Home router NAT** — translates `192.168.1.152` (and every other device on my home network) to the single public IP `173.66.13.41`. This is standard residential NAT.

2. **VPN server NAT** — at the GMU end, the VPN server performs NAT to translate my VPN-assigned `10.151.204.231` to GMU's public IP before forwarding to the internet.

This layered NAT is why the VPN works transparently — the destination server just sees a GMU IP address and never has any visibility into my actual private addresses.

### Privacy and Security Implications

The VPN connection masked:
- My real public IP address (Verizon residential)
- My ISP identity
- My approximate physical location (Fairfax VA → Arlington VA)
- My ASN (Verizon 701 → GMU)

For a student or remote worker accessing university resources, the VPN also means all traffic between my device and the GMU network is encrypted in transit — an eavesdropper on my local Wi-Fi network can see that I'm connected to a VPN endpoint but cannot read the tunnel contents.

What the VPN doesn't hide: GMU's VPN server sees all my traffic in cleartext (after decryption). The VPN shifts trust from my ISP to the VPN provider — in this case, from Verizon to GMU's IT infrastructure.

---

## Key Concepts Demonstrated

| Concept | What the Lab Showed |
|---|---|
| RFC 1918 private addressing | `192.168.x.x` (home) and `10.x.x.x` (VPN) are both private ranges |
| NAT | Single public IP (`173.66.13.41`) represents all devices on home network |
| VPN tunnel | Traffic encrypted between device and GMU VPN server |
| IP reassignment via VPN | New private address (`10.151.204.231`) assigned by VPN server |
| Public IP masking | Destination sees GMU's IP, not Verizon's |
| ASN and BGP | ASN 701 (Verizon) changed to GMU's ASN after VPN — BGP-level identity change |
| Subnet sizing | /24 (home, 254 hosts) vs /17 (VPN pool, 32,766 hosts) |
