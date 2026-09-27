# TCP-IP-Communication-in-Action-Group-Presentation
# How Data Travels Across a Network: HTTPS Web Browsing over TCP/IP

**Scenario:** Opening a secure website over HTTPS from a laptop to an Internet web server

**Assumptions (stated for the team):**
- Laptop: `192.168.1.50`, MAC `AA:AA:AA:AA:AA:01`
- Default gateway (home/enterprise router): `192.168.1.1`, MAC `BB:BB:BB:BB:BB:01`
- DNS server: `192.168.1.1` (forwarding to ISP resolver `8.8.8.8` for this example)
- Destination web server: `203.0.113.10`, listening on TCP port `443`
- Client ephemeral source port: `TCP 51500`
- Network path: Laptop → Wi-Fi access point → Ethernet switch → home/enterprise router (with NAT + firewall) → ISP network → Internet routers → destination web server

---

## 1. TCP/IP and OSI Model Explanation

### The Four TCP/IP Layers

| TCP/IP Layer | Function |
|---|---|
| **Application** | Provides the interface and protocols applications use to communicate — for our scenario, HTTP/HTTPS (with TLS) and DNS. It formats the request and interprets the response. |
| **Transport** | Provides end-to-end delivery between processes using port numbers. TCP is used here to give reliable, ordered, connection-oriented delivery of the HTTPS session. |
| **Internet** | Handles logical addressing (IPv4) and routing of packets between networks, from source IP to destination IP, across multiple routers. |
| **Network Access (Link)** | Handles physical addressing (MAC) and framing on the local network segment — Wi‑Fi and Ethernet — and the actual transmission of bits over the medium. |

### Mapping TCP/IP to OSI (7 Layers)

| OSI Layer | TCP/IP Layer | Role in this scenario |
|---|---|---|
| 7. Application | Application | The browser generates the HTTP request (e.g., `GET / HTTP/1.1`) for the web page. |
| 6. Presentation | Application | TLS handles encryption, certificate validation, and data formatting so the session is confidential and authenticated (HTTPS = HTTP + TLS). |
| 5. Session | Application | TLS also manages the secure session (handshake, session keys, session resumption) between browser and server. |
| 4. Transport | Transport | TCP establishes a connection (three-way handshake), assigns source/destination ports, and guarantees reliable, ordered delivery. |
| 3. Network | Internet | IP addresses the packet with source `192.168.1.50` and destination `203.0.113.10`, and routers forward it toward the destination network. |
| 2. Data Link | Network Access | Ethernet/Wi-Fi frames carry the packet across each physical hop, using MAC addresses that change at every router (each new "leg" of the journey). |
| 1. Physical | Network Access | The actual electrical signals, radio waves (Wi-Fi), or light pulses (fiber) that carry the bits across the medium. |

**Note for the team:** OSI has two layers (Presentation and Session) that TCP/IP does not separate out — TCP/IP folds these into "Application." This is a key mapping point to mention on your OSI slide.

---

## 2. Packet Journey Narrative

### Step 0 – User Action
The user types `https://www.example.com` into the browser and presses Enter.

### Step 1 – DNS Resolution (Application Layer, before the HTTPS session)
- The browser needs the IP address for `www.example.com`.
- It sends a **DNS query** (UDP port 53, or TCP 53 if the response is large) to the configured DNS server (`192.168.1.1`, which forwards to `8.8.8.8` if it doesn't have it cached).
- The DNS server responds with the resolved IP: `203.0.113.10`.
- **Encapsulation for this step:** DNS message → UDP segment (source port ephemeral, destination port 53) → IP packet (src `192.168.1.50`, dst DNS server) → Ethernet/Wi-Fi frame (src laptop MAC, dst gateway MAC).

### Step 2 – TCP Three-Way Handshake (Transport Layer)
- Before any HTTP data is sent, the laptop establishes a TCP connection to `203.0.113.10:443`.
- **SYN** → **SYN-ACK** → **ACK** exchanged between client port `51500` and server port `443`.
- This guarantees a reliable, ordered channel exists before data is exchanged.

### Step 3 – TLS Handshake (Application Layer, "on top of" TCP)
- Client and server negotiate a TLS version and cipher suite.
- The server presents its **digital certificate**; the browser validates it against trusted Certificate Authorities.
- A shared session key is derived (e.g., via TLS 1.3's key exchange), and from this point the session is encrypted.

### Step 4 – HTTP Request Sent (Application → Transport → Internet → Network Access)
This is where full **encapsulation** happens for the actual page request:

1. **Application Layer:** Browser builds `GET / HTTP/1.1` with headers (Host, User-Agent, etc.). This is encrypted by TLS before being handed down. **PDU: Data/Message.**
2. **Transport Layer:** TCP wraps the encrypted HTTP data in a **segment**, adding source port `51500` and destination port `443`, sequence/acknowledgment numbers, and flags. **PDU: Segment.**
3. **Internet Layer:** IP wraps the segment in a **packet**, adding source IP `192.168.1.50` and destination IP `203.0.113.10`, TTL, and other header fields. **PDU: Packet.**
4. **Network Access Layer:** Ethernet (and then Wi-Fi, or vice versa depending on the client's connection type) wraps the packet in a **frame**, adding source MAC (laptop) and destination MAC (the *default gateway's* MAC, not the web server's — this is a key teaching point). **PDU: Frame**, converted to bits for transmission.

### Step 5 – Traversing the Local Network and Intermediate Devices
- The frame leaves the laptop's Wi-Fi adapter and is received by the **access point**, which bridges it onto the wired network.
- The **Ethernet switch** forwards the frame based on MAC address (Layer 2) toward the router's LAN interface.
- The **router** (acting as default gateway) receives the frame, **strips the Layer 2 header (decapsulates to the packet)**, examines the destination IP, and determines it must be forwarded out to the Internet.
- The router (often combined with a **firewall/NAT device** in home/enterprise setups) performs **NAT** — translating the private source IP `192.168.1.50:51500` to a public IP and port — and checks firewall rules (outbound HTTPS on 443 is normally allowed).
- The router **re-encapsulates** the packet into a *new* frame with new source/destination MAC addresses (its own MAC as source, the next hop router's MAC as destination) and forwards it to the ISP.

### Step 6 – Routing Across the Internet
- The packet passes through multiple **ISP and backbone routers**. At each hop:
  - The frame is decapsulated to the packet (Layer 2 header removed).
  - The router reads the destination IP and TTL, decrements TTL, and looks up the next hop in its routing table.
  - The packet is re-encapsulated into a new frame for the next physical link.
- The IP addresses (source/destination) **remain unchanged** across this journey (aside from the NAT translation done once at the edge); only the **MAC addresses change at every hop**.

### Step 7 – Arrival at the Destination Server
- The final router/switch in the server's network delivers the frame to the web server's network interface.
- The server decapsulates: Frame → Packet → Segment → Data, verifying IP addresses, TCP port 443, and completing the TLS/HTTP processing.
- The server processes the HTTP request and prepares an **HTTP response** (the requested web page).

### Step 8 – The Response Journey (Reverse Path)
- The exact same encapsulation process happens in reverse at the server: HTTP response → TLS-encrypted → TCP segment (src port 443, dst port 51500) → IP packet (src `203.0.113.10`, dst public IP of the router) → Ethernet frame.
- The response is routed back across the Internet, arrives at the home/enterprise router, which **reverses the NAT translation** (mapping the public IP:port back to `192.168.1.50:51500`) and forwards it to the laptop via the switch and access point.
- The laptop decapsulates each layer in turn, TLS decrypts the payload, and the browser renders the web page.

### Step 9 – Connection Teardown
- Once the page (and any additional resources) has been fully transferred, the TCP connection is closed via a **FIN/ACK** exchange (or the connection may be kept alive briefly for further requests, common with HTTP/1.1 keep-alive or HTTP/2 multiplexing).

**Key teaching point for the team:** IP addresses stay constant from source to destination (except where NAT translates them once), but **MAC addresses change at every Layer 2 hop** — this is one of the most commonly misunderstood concepts and worth its own slide emphasis.

---

## 3. Packet Journey Diagram — Step-by-Step Drawing Guide

Draw this as a **horizontal diagram** with devices left to right, and **layer labels along the bottom**.

**Row of devices (left to right):**
1. **Laptop** (`192.168.1.50`, MAC `AA:AA:AA:AA:AA:01`)
2. **Wi-Fi Access Point**
3. **Ethernet Switch**
4. **Router/Firewall (Default Gateway)** (`192.168.1.1`, MAC `BB:BB:BB:BB:BB:01`, performs NAT)
5. **ISP Router(s)** (draw as a cloud labeled "Internet")
6. **Destination Web Server** (`203.0.113.10`, TCP port 443)

**Arrows:**
- Draw a **solid arrow above the device row**, pointing left→right, labeled "**Request: HTTPS GET →**".
- Draw a **dashed arrow below the device row**, pointing right→left, labeled "**← Response: HTTP 200 + page content**".

**Layer labels (place as a horizontal strip beneath the whole diagram, or as small stacked boxes at each device):**
- At the **Laptop** and **Server** ends, draw a small stack of 4 boxes (bottom to top): `Network Access → Internet → Transport → Application`, showing full encapsulation/decapsulation happens at the endpoints.
- At the **Switch**, show only the `Network Access` box highlighted (Layer 2 only — switches don't process IP).
- At the **Router**, show `Network Access` and `Internet` boxes highlighted, with a small icon/arrow showing "decapsulate → re-encapsulate" (routers rebuild the Layer 2 frame at every hop, but keep the IP packet mostly intact except TTL and, at this router, NAT translation).

**Annotations to add:**
- Next to the Laptop: label the outgoing frame `Src MAC: AA:AA...01 → Dst MAC: BB:BB...01 (gateway)`.
- Next to the Router-to-ISP link: label `Src IP: 192.168.1.50 (NAT'd to public IP) → Dst IP: 203.0.113.10`, and note "MAC addresses change here; IP addresses do not (except NAT)."
- Next to the Server: label `Dst Port: TCP 443`, `Src Port: TCP 51500`.
- Add a small **lock icon** on the arrow between Laptop and Server to represent TLS encryption, labeled "TLS-encrypted payload."
- Add a small callout box: "**Encapsulation** happens top-to-bottom at the sender; **decapsulation** happens bottom-to-top at the receiver."

This produces a clean, exam-friendly diagram that shows device topology, direction of traffic, address changes, and where encapsulation/decapsulation occurs.

---

## 4. Protocol Analysis Table

| TCP/IP Layer | Example Protocol(s) | PDU Name | Addressing Used | Purpose/Role in This Scenario |
|---|---|---|---|---|
| Application | HTTP, HTTPS (TLS), DNS | Data / Message | Domain names (resolved to IPs); no layer-specific address of its own | HTTP/TLS builds and encrypts the web page request/response; DNS resolves `www.example.com` to `203.0.113.10` |
| Transport | TCP (also UDP for DNS) | Segment (TCP) / Datagram (UDP) | Port numbers (src 51500, dst 443 for HTTPS; dst 53 for DNS) | TCP provides reliable, ordered delivery of the HTTPS session; UDP carries the quick DNS query/response |
| Internet | IPv4 | Packet | IP addresses (src `192.168.1.50`, dst `203.0.113.10`) | Routes the packet across networks from the laptop to the web server |
| Network Access (Link) | Ethernet, Wi-Fi (802.11), ARP | Frame | MAC addresses (change at every hop) | Delivers the frame across each physical/local segment; ARP resolves the next-hop IP to a MAC address |

---

## 5. TCP vs UDP Comparison

| Characteristic | TCP | UDP |
|---|---|---|
| Connection model | Connection-oriented (three-way handshake) | Connectionless (no handshake) |
| Reliability | Reliable — lost segments are retransmitted | Unreliable — no retransmission |
| Ordering | Guarantees in-order delivery via sequence numbers | No ordering guarantee |
| Error checking | Checksum + acknowledgments + retransmission | Checksum only, no recovery |
| Congestion control | Yes (e.g., slow start, congestion avoidance) | None |
| Overhead | Higher (handshake, ACKs, header ~20 bytes) | Lower (minimal header, 8 bytes) |
| Performance implication | Slower to start, but guarantees data integrity — ideal when correctness matters more than speed | Faster, lower latency — ideal when speed matters more than occasional loss |

**TCP examples (this scenario and general):**
1. HTTPS web browsing (this scenario) — the page must arrive complete and in order.
2. Email transfer (SMTP) — messages must not be corrupted or lost.

**UDP examples:**
1. DNS queries (used in this scenario) — small, quick request/response; the client just retries if no answer comes.
2. VoIP / video conferencing media streams — a lost or delayed packet is worse to wait for (causes lag) than to simply skip, so speed is prioritized over perfect reliability.

---

## 6. Communication Failure and Troubleshooting

### Invented Failure Scenario
A user reports: **"I can't open `https://www.example.com` — the page just times out, but other websites work fine."**

**Root cause (to be revealed at the end):** The company firewall was recently reconfigured and is now blocking outbound TCP port 443 to that specific server's IP range, while DNS and general Internet access work fine.

### Troubleshooting Flow

**Step 1 – Observe and Define the Problem**
- Confirm exactly what fails: Is it this one site, or all HTTPS sites? Browser error message (timeout vs. "connection refused" vs. certificate error) gives clues.
- Confirmed: only `www.example.com` fails; other HTTPS sites work.

**Step 2 – Check Basic Connectivity**
- Run `ipconfig` / `ifconfig` to confirm the laptop has a valid IP, subnet mask, and default gateway (`192.168.1.1`).
- `ping 192.168.1.1` (gateway) — succeeds, confirming local network connectivity is fine.
- `ping 203.0.113.10` — may fail or succeed depending on whether ICMP is filtered; not conclusive on its own for HTTPS issues.

**Step 3 – Verify Name Resolution (DNS)**
- Run `nslookup www.example.com` or `dig www.example.com`.
- Confirms DNS correctly resolves to `203.0.113.10` — so the problem is **not** DNS.

**Step 4 – Check Ports and Protocols**
- Attempt `Test-NetConnection 203.0.113.10 -Port 443` (Windows) or `telnet 203.0.113.10 443` / `nc -zv 203.0.113.10 443` (Linux/macOS).
- Result: the TCP connection to port 443 **times out** — no SYN-ACK is received, pointing to something blocking or dropping the traffic rather than the server being down (a "connection refused" would suggest the server itself is rejecting it).

**Step 5 – Inspect Devices**
- Check the **switch** — port status is up, no VLAN misconfiguration (rules out Layer 2 issue).
- Check the **router/firewall logs** — this is where the administrator finds **explicit deny log entries** for outbound TCP port 443 traffic to `203.0.113.0/24`, added during a recent rule change.
- Check the **server status** (via an external monitoring tool or a colleague on a different network) — the server responds fine to others, ruling out a server-side outage.

**Step 6 – Identify the Root Cause**
- The firewall rule change is the root cause: an overly broad "deny" rule targeting that IP range was applied, unintentionally blocking legitimate HTTPS traffic to `www.example.com`.

**Step 7 – Apply a Fix and Confirm**
- The administrator adds an explicit **allow rule** for outbound TCP port 443 to `203.0.113.10` (or narrows the deny rule that caused the conflict), applies the change, and reloads the firewall configuration.
- Re-test: `Test-NetConnection 203.0.113.10 -Port 443` now succeeds; the user reloads the browser and the page loads correctly.
- Document the change and the cause for future reference.

---

## 7. Slide Outline (12 Content Slides)

**Slide 1 — Scenario Overview and Objectives**
- Scenario: Opening a secure website (HTTPS) from a laptop to an Internet web server
- Objective: Trace how data travels using the TCP/IP model, mapped to OSI
- What the audience will learn: encapsulation, addressing, protocols, troubleshooting
*Speaker note: Introduce the scenario in one sentence, state that the presentation will follow a single HTTPS request from click to page-load, and preview the four main parts of the talk (model, journey, protocols, troubleshooting).*

**Slide 2 — TCP/IP Model Overview**
- Application, Transport, Internet, Network Access layers
- One-line function of each layer
- Applied to HTTPS: browser (App), TCP (Transport), IP (Internet), Ethernet/Wi-Fi (Network Access)
*Speaker note: Walk through the four layers top to bottom, giving a concrete example from the HTTPS scenario for each one so it isn't abstract.*

**Slide 3 — OSI Model and Mapping to TCP/IP**
- 7 OSI layers listed
- Mapping table: which OSI layers combine into each TCP/IP layer
- Key point: Presentation and Session are folded into TCP/IP's Application layer
*Speaker note: Emphasize that OSI is more granular and mainly used as a teaching/reference model, while TCP/IP is what's actually implemented on the Internet.*

**Slide 4 — Packet Journey: DNS Resolution**
- Browser needs an IP for the domain name
- DNS query sent over UDP port 53
- Response: IP address returned (`203.0.113.10`)
*Speaker note: Explain that before any web traffic happens, the device must resolve the human-readable domain name into an IP address, and this uses UDP because it's a small, fast exchange.*

**Slide 5 — Packet Journey: Connection Setup (TCP + TLS)**
- TCP three-way handshake (SYN, SYN-ACK, ACK)
- TLS handshake: certificate validation, key exchange
- Result: secure, reliable channel established
*Speaker note: Describe these as two separate handshakes stacked on top of each other — TCP guarantees reliable delivery, TLS guarantees confidentiality and authentication.*

**Slide 6 — Packet Journey: Encapsulation of the Request**
- HTTP request → TLS encrypted → TCP segment → IP packet → Ethernet/Wi-Fi frame
- Show PDU names at each layer
- Addressing added at each layer (ports, IPs, MACs)
*Speaker note: Walk through each encapsulation step in order, pointing out what gets added at each layer and using the PDU name at each stage.*

**Slide 7 — Packet Journey: Across the Network and Back**
- Local network: AP → switch → router (NAT + firewall)
- Internet: multiple ISP/backbone routers, MAC changes each hop, IP stays the same
- Response follows the reverse path back to the laptop
*Speaker note: Stress that IP addresses stay constant end-to-end (aside from NAT) while MAC addresses change at every router hop — this is the most important concept on this slide.*

**Slide 8 — Packet Journey Diagram (Explained)**
- Show the diagram (devices, arrows, layer labels)
- Highlight where encapsulation/decapsulation occurs
- Highlight address changes along the path
*Speaker note: Talk the audience through the diagram left to right, following the request arrow out and the response arrow back, pointing at each device as you describe what happens there.*

**Slide 9 — Protocol Analysis Table**
- Show the 4-layer protocol table (protocol, PDU, addressing, purpose)
- Brief explanation of why each protocol was chosen
*Speaker note: Don't read the whole table aloud — pick two or three rows (e.g., TCP and IP) and explain them in more depth as examples.*

**Slide 10 — TCP vs UDP Comparison**
- Key differences: connection-oriented vs connectionless, reliability, overhead
- TCP examples: HTTPS, email
- UDP examples: DNS, VoIP/video streaming
*Speaker note: Explain why HTTPS specifically needs TCP's reliability, then contrast with DNS's use of UDP for speed, to make the comparison concrete rather than abstract.*

**Slide 11 — Communication Failure Scenario**
- Problem: user can't reach the website (times out), other sites work fine
- Root cause (revealed at end): firewall rule blocking outbound TCP 443 to that server
*Speaker note: Present this like a mini mystery — describe the symptom first, then let the next slide show how it was diagnosed.*

**Slide 12 — Troubleshooting Steps**
- Flow: check connectivity → check DNS → check ports (TCP 443) → check devices/firewall logs → identify cause → fix → confirm
- Fix: firewall rule corrected; connection restored
*Speaker note: Walk through the troubleshooting flow as a logical, repeatable process the audience could apply to other problems, not just this one specific case.*

---

## 8. References

1. **RFC 9110 – HTTP Semantics** (Internet Engineering Task Force, IETF) — the official standard defining HTTP.
2. **RFC 793 / RFC 9293 – Transmission Control Protocol (TCP)** (IETF) — the official standard defining TCP's connection and reliability mechanisms.
3. **Cisco Networking Academy – "Networking Basics" / CCNA Introduction to Networks course materials** (Cisco Systems) — covers TCP/IP and OSI models, encapsulation, and addressing in an educational format.
4. **CompTIA Network+ Certification Exam Objectives and Study Guide** (CompTIA) — covers TCP vs UDP, protocol layering, and troubleshooting methodology.
5. **Microsoft Learn – "TCP/IP Protocol Architecture"** (Microsoft) — vendor documentation on TCP/IP implementation and NAT/firewall behavior in enterprise networks.

*(The team should look these up directly and format citations per their required style, e.g., APA or IEEE.)*
