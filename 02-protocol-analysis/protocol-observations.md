# Protocol Observations — Wireshark Capture

**Local device:** `192.168.1.152`
**Capture interface:** Wi-Fi en0
**Tool:** Wireshark 4.4.0

---

## ICMP — Internet Control Message Protocol

**Test:** Pinged Google's public DNS server (`8.8.8.8`) from terminal using `ping 8.8.8.8`
**Wireshark filter:** `icmp`

### What I Captured

The capture showed clean echo request/reply pairs between my local device and Google's DNS server. Each outbound request from `192.168.1.152` was matched by a reply from `8.8.8.8` with no packet loss.

**TTL Behavior:**
- Outbound (my device → 8.8.8.8): TTL = 64
- Inbound (8.8.8.8 → my device): TTL = 60

The TTL drop from 64 to 60 on the return path is exactly what you'd expect — each router hop decrements TTL by 1. Four hops between Google's DNS and my machine means four routers in the path. This is standard behavior and confirms end-to-end internet connectivity with no data loss or abnormal latency.

### Key Observations
- All ICMP echo requests were immediately followed by corresponding replies — zero packet loss
- Consistent TTL values across all pairs, indicating a stable routing path
- The sequence numbers incremented correctly across all ping packets

### What This Demonstrates
ICMP is the foundation of basic network diagnostics. Understanding TTL behavior is directly relevant to traceroute analysis and identifying routing anomalies. In a security context, ICMP is also notable as an exfiltration channel — attackers have used ICMP tunneling to smuggle data past firewalls that allow ping traffic. The ICMP Timestamp finding in the Nessus lab connects directly to this.

---

## TLS — Transport Layer Security (v1.2 and v1.3)

**Test:** Browsed the web normally while Wireshark was running
**External hosts:** `17.253.23.201` and `17.57.144.54` (Apple infrastructure)
**Wireshark filter:** `tls`

### What I Captured

Two separate TLS sessions appeared in the capture:

**Session 1 — TLSv1.3 to 17.253.23.201 on port 443**
Standard HTTPS browsing traffic. The PSH (Push) and ACK (Acknowledgment) flags on the packets indicate active data transfer in progress — the client is pushing data to the server and acknowledging received data. All payload content is encrypted; Wireshark can see the TLS handshake metadata and record headers, but not the decrypted content.

**Session 2 — TLSv1.2 to 17.57.144.54 on port 5223**
Port 5223 is Apple's APNs (Apple Push Notification Service) port. This session was TLSv1.2 rather than 1.3, which is worth noting — older TLS versions are still used for some persistent connection services. The PSH+ACK pattern here is the same: encrypted data being exchanged bidirectionally.

### Key Observations
- TLS 1.3 used for standard HTTPS browsing (443) — current standard, strong forward secrecy
- TLS 1.2 used for push notification service (5223) — still widely supported but older
- All application layer content is encrypted — Wireshark shows record type and length but not payload
- The presence of TLS everywhere confirms that modern applications default to encrypted transport

### What This Demonstrates
Being able to identify TLS sessions, distinguish versions, and understand what the metadata reveals (even when content is encrypted) is a fundamental network analysis skill. In security monitoring, TLS metadata — session duration, certificate details, server name indication (SNI) — is often more useful than trying to decrypt content.

---

## TCP — Transmission Control Protocol

**Test:** Opened the Maps application on macOS
**Wireshark filter:** `tcp`

### What I Captured

Opening Maps triggered an initial TCP connection establishment followed by a series of data exchanges. The capture showed:

- **PSH flag (seq 1101):** Client pushing data to server — Maps requesting map tile data
- **ACK flag (seq 1977):** Server acknowledging receipt, client acknowledging server data
- Bidirectional exchange with consistent acknowledgment numbers, indicating reliable delivery

No retransmissions appeared in the capture — all packets were acknowledged cleanly. This is what reliable, healthy TCP communication looks like.

### Key Observations
- Three-way handshake (SYN → SYN-ACK → ACK) visible at session start
- Consistent sequence and acknowledgment number progression
- Zero retransmissions — no packet loss in the capture window
- Clean connection teardown (FIN/ACK sequence) at session end

### What This Demonstrates
TCP's reliability mechanisms — sequence numbers, acknowledgments, retransmission on loss — are fundamental to understanding how most application protocols work. For security analysis, abnormal TCP behavior (SYN floods, RST injection, unexpected FIN packets) are key indicators of network attacks and connection manipulation.

---

## UDP — User Datagram Protocol

**Test:** Streamed social media video content
**Remote host:** `172.253.63.100` (Google/YouTube infrastructure)
**Wireshark filter:** `udp`

### What I Captured

UDP traffic appeared immediately when video playback started. Unlike the TCP sessions, there's no connection establishment — UDP just sends datagrams and doesn't wait for confirmation.

The session showed a continuous stream of UDP datagrams between my device and the streaming server, followed by a cancel/termination segment when I force-closed the stream. No packet loss indicators appeared in the capture window, and the session ended cleanly without retransmission attempts (which UDP doesn't do anyway — that's the application's problem, or in real-time media, just accepted).

### Key Observations
- No SYN/ACK handshake — UDP sessions start immediately with data
- High packet frequency during streaming, confirming real-time data delivery
- Clean session termination without the TCP FIN/ACK ceremony
- Application-layer protocols (QUIC, RTP) handle any reliability needs on top of UDP

### What This Demonstrates
UDP's tradeoff — speed over reliability — is exactly why it's used for streaming and real-time communication. Understanding when protocols choose UDP vs TCP is important for both network design and security analysis. UDP-based protocols are also commonly exploited for amplification attacks (DNS, NTP, SSDP amplification) because of the lack of a handshake.

---

## RTP — Real-Time Transport Protocol

**Test:** Initiated a video call
**Local device:** `192.168.1.152`
**Remote device:** `192.168.1.168` (another device on the local network)
**Wireshark filter:** `rtp`

### What I Captured

RTP packets appeared immediately after the video call connected. RTP runs over UDP (visible in the encapsulation), which makes sense — for real-time audio and video, a dropped frame is better than a delayed one. RTP doesn't request retransmission of lost packets; it just keeps the stream moving.

The capture showed:
- Bidirectional RTP streams between the two devices — separate streams for audio and video
- UDP as the transport layer underneath RTP
- Consistent payload type fields identifying the codec in use
- Sequence numbers incrementing correctly, allowing the receiver to detect any gaps

### Key Observations
- RTP operates over UDP — confirmed by the encapsulation visible in packet details
- Bidirectional streams for full-duplex communication (both devices sending and receiving simultaneously)
- Sequence numbers allow gap detection even without retransmission
- RTP is accompanied by RTCP (Real-Time Control Protocol) for quality monitoring

### What This Demonstrates
RTP is the backbone of VoIP and video conferencing — understanding it is directly relevant to securing unified communications systems. In enterprise security, RTP traffic is a common target for eavesdropping attacks on poorly segmented networks, since the audio/video payload (when not end-to-end encrypted) can be reassembled from captured packets with tools like rtpbreak or VoIPong.

---

## Summary

| Protocol | Transport | Port(s) | Use Case | Security Notes |
|---|---|---|---|---|
| ICMP | — | — | Connectivity testing | Can be used for tunneling/exfiltration |
| TLS 1.3 | TCP | 443 | HTTPS browsing | Gold standard, strong forward secrecy |
| TLS 1.2 | TCP | 5223 | Push notifications | Older version, still widely deployed |
| TCP | — | Various | Reliable app data | SYN floods, RST injection attack vectors |
| UDP | — | Various | Streaming media | Amplification attack vector |
| RTP | UDP | Dynamic | Real-time media | Eavesdropping risk on unencrypted streams |

The capture confirmed clean, healthy communication across all protocols — no anomalies, no retransmissions, no unexpected behavior. From a security monitoring perspective, this represents a baseline of normal network behavior that anomaly detection systems would use as a reference point.
