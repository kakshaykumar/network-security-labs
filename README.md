# Network Security Labs

---

## Overview

This repo contains hands-on lab work from my Cybersecurity Fundamentals course covering three core areas: vulnerability scanning, traffic analysis, and network address management. All three labs were run against live environments — real scan results, real packet captures, real terminal output.

I came into this course with a network engineering background and hands-on experience with tools like Nessus, Wireshark, Qualys, SolarWinds, and NMap. These labs were a chance to use those tools in a more structured, documentation-focused way and think deliberately about what the findings mean from a security standpoint — not just what the tool says, but why it matters.

---

## Labs

| # | Lab | Tools | What It Covers |
|---|---|---|---|
| 01 | [Vulnerability Scanning](01-vulnerability-scanning/) | Tenable Nessus | Scan configuration, CVSS scoring, finding analysis, remediation |
| 02 | [Protocol Analysis](02-protocol-analysis/) | Wireshark | Live packet capture, 5-protocol analysis, traffic interpretation |
| 03 | [NAT & VPN](03-nat-vpn/) | ifconfig, GMU VPN | IP addressing, NAT behavior, VPN tunnel mechanics |

---

## Lab Summaries

### Lab 01 — Vulnerability Scanning with Nessus

Installed Tenable Nessus and ran a Basic Network Scan against my home router (`192.168.1.1`). The scan ran for 6 minutes and returned 34 findings — 3 medium, 2 low, 29 informational. No critical or high findings on the router itself.

The three medium findings were all SSL/TLS related: an untrusted certificate, a self-signed certificate, and IP forwarding enabled — all realistic issues you'd flag in a real network assessment. Each is documented with its CVSS v3.0 score, what the risk actually means, and a proper mitigation path.

→ [Scan findings and analysis](01-vulnerability-scanning/findings/scan-findings.md)

---

### Lab 02 — Protocol Analysis with Wireshark

Captured live traffic from my local device (`192.168.1.152`) across five protocol scenarios:

- **ICMP** — pinged Google's DNS (`8.8.8.8`) from terminal. Watched echo request/reply pairs. TTL dropped from 64 to 60 on return — normal hop reduction, confirms internet connectivity with zero packet loss.
- **TLS 1.2 / 1.3** — browsed the web and observed encrypted sessions to Apple servers on ports 443 and 5223. PSH+ACK flags visible, confirming active data transfer.
- **TCP** — opened Maps app, captured the PSH/ACK sequence and acknowledgment exchange. Stable, no retransmissions.
- **UDP** — streamed social media content to `172.253.63.100`. Connectionless, no handshake, clean session termination.
- **RTP** — initiated a video call and captured real-time transport packets between local device and `192.168.1.168`. UDP underneath, as expected for latency-sensitive media.

→ [Full protocol observations](02-protocol-analysis/protocol-observations.md)

---

### Lab 03 — NAT & VPN: Observing IP Address Behavior

Used `ifconfig` to capture my local network config, verified my public IP via `whatismyipaddress.com`, connected to the GMU VPN, then repeated both.

**Before VPN:**
- Local: `192.168.1.152` — private /24, home network (Verizon)
- Public: `173.66.13.41` — Verizon Business, Fairfax VA, ASN 701

**After VPN:**
- Local: `10.151.204.231` — private /17, VPN-assigned subnet
- Public: `129.174.182.103` — George Mason University, Arlington VA

The results demonstrate NAT and VPN tunneling working exactly as expected. My traffic was now egressing through GMU's infrastructure, my ISP and location were masked, and I was placed in a different RFC 1918 address space managed by the VPN server. Also observed the ASN change in BGP routing — 701 (Verizon) before, GMU's ASN after.

→ [Full NAT/VPN analysis](03-nat-vpn/nat-vpn-analysis.md)

---

## Tools

| Tool | Version | Purpose |
|---|---|---|
| Tenable Nessus Essentials | Latest | Vulnerability scanning |
| Wireshark | 4.4.0 | Packet capture and analysis |
| macOS Terminal / ifconfig | — | Network config inspection |
| GMU VPN | OpenVPN-based | VPN tunnel testing |
| whatismyipaddress.com | — | Public IP verification |

---

## Skills Demonstrated

- Vulnerability assessment end-to-end: scan setup → findings → CVSS scoring → remediation recommendations
- Network traffic capture and multi-protocol layer analysis
- TCP/IP stack behavior across ICMP, TCP, UDP, TLS, and RTP
- NAT mechanics, private vs. public IP addressing, RFC 1918
- VPN tunnel behavior — IP reassignment, ISP masking, BGP/ASN routing
- Translating raw tool output into clear, actionable security analysis

---
