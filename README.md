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

<img width="563" height="310" alt="image" src="https://github.com/user-attachments/assets/aa07edb3-8669-4a45-86fe-27500b700d8e" />


| TCP/IP Layer | Function |
|---|---|
| **Application** | Provides the interface and protocols applications use to communicate — for our scenario, HTTP/HTTPS (with TLS) and DNS. It formats the request and interprets the response. |
| **Transport** | Provides end-to-end delivery between processes using port numbers. TCP is used here to give reliable, ordered, connection-oriented delivery of the HTTPS session. |
| **Internet** | Handles logical addressing (IPv4) and routing of packets between networks, from source IP to destination IP, across multiple routers. |
| **Network Access (Link)** | Handles physical addressing (MAC) and framing on the local network segment — Wi‑Fi and Ethernet — and the actual transmission of bits over the medium. |

### Mapping TCP/IP to OSI (7 Layers)

<img width="647" height="346" alt="image" src="https://github.com/user-attachments/assets/a7e8abb0-36e0-4f34-b929-286dbc660e0c" />


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
## 7. Slide Outline (19 Slides)

The final deck contains a title slide, 17 content slides (including 5 lab-evidence slides built from our Wireshark and terminal captures) and a references slide.

**Slide 1 — Title**
- How Data Travels Across a Network
- A TCP/IP and OSI walkthrough of opening a secure website (HTTPS)
- Presenters and roles
*Speaker note: Introduce the group and the topic in one or two sentences, and explain that the talk follows one HTTPS request from the moment the URL is typed to the moment the page loads.*

**Slide 2 — Scenario Overview and Objectives**
- Scenario: Opening a secure website (HTTPS) from a laptop to an Internet web server
- Journey at a glance: laptop/browser → Wi-Fi access point → switch → router (NAT + firewall) → Internet (ISP routers) → web server (`203.0.113.10`)
- Objectives: trace data with the TCP/IP model mapped to OSI, show encapsulation and decapsulation, identify protocols, addresses and ports, and diagnose a realistic failure
*Speaker note: State the scenario in one sentence, preview the four parts of the talk (model, journey, protocols, troubleshooting) and mention that real lab captures back up each step.*

**Slide 3 — TCP/IP Model: Four Layers**
- Application, Transport, Internet, Network Access
- One-line function of each layer
- Applied to HTTPS: browser (Application), TCP (Transport), IP (Internet), Ethernet/Wi-Fi (Network Access)
*Speaker note: Walk through the four layers top to bottom, giving a concrete HTTPS example for each so the model isn't abstract.*

**Slide 4 — OSI Model and Mapping to TCP/IP**
- 7 OSI layers listed with their TCP/IP equivalents and their role in this scenario
- Key point: Application, Presentation and Session all fold into TCP/IP's Application layer; Data Link and Physical fold into Network Access
- Note: Presentation and Session are not separate layers in TCP/IP
*Speaker note: Emphasize that OSI is a more granular teaching and reference model, while TCP/IP is what is actually implemented on the Internet.*

**Slide 5 — Packet Journey: DNS Resolution**
- Browser needs an IP address for the domain name
- DNS query sent over UDP to port 53, possibly forwarded to an upstream resolver
- Response returns the IP address; UDP is used because the exchange is small and can simply be retried
- DNS query encapsulation: message → UDP datagram → IP packet → Ethernet/Wi-Fi frame
*Speaker note: Explain that before any web traffic flows, the device must turn the human-readable name into an IP address, and that UDP suits this small, fast exchange.*

**Slide 6 — Lab Evidence: DNS Lookup in Practice**
- Terminal: `nslookup wikipedia.org` against DNS server `10.0.2.3` (port 53) returns `195.200.68.224` (IPv4) and `2a02:ec80:700:ed1a::1` (IPv6)
- Wireshark `dns` filter: two queries (A and AAAA) and two responses between `10.0.2.15` and `10.0.2.3`
- First query is 73 bytes on the wire, sent over UDP
*Speaker note: Point at the nslookup output first, then at the Wireshark list, showing that one lookup actually produces an A query and an AAAA query.*

**Slide 7 — Packet Journey: Connection Setup (TCP + TLS)**
- TCP three-way handshake: SYN, SYN-ACK, ACK
- TLS handshake on top of TCP: ClientHello, ServerHello and certificate, key exchange
- Result: a reliable, secure and authenticated channel to port 443
*Speaker note: Describe these as two separate handshakes stacked on each other — TCP guarantees reliable delivery, TLS guarantees confidentiality and authentication.*

**Slide 8 — Lab Evidence: TCP Handshake and TLS in Wireshark**
- `tcp.port == 443` capture: SYN, SYN-ACK and ACK between `10.0.2.15` (port 50992) and `151.101.129.91` (port 443)
- TLS 1.3 Client Hello (SNI `ads.mozilla.org`), then Change Cipher Spec
- After the handshake, the payload appears only as encrypted Application Data
- Note: this capture is a Firefox connection to ads.mozilla.org; the handshake sequence is identical for any HTTPS site, including Wikipedia
*Speaker note: Walk down the Info column line by line, and stress that once the handshake finishes nothing readable remains in the packets.*

**Slide 9 — Packet Journey: Encapsulation of the Request**
- HTTP request → TLS encrypted → TCP segment → IP packet → Ethernet/Wi-Fi frame
- PDU names at each layer: Data, Segment, Packet, Frame
- Addressing added at each layer: ports, IP addresses, MAC addresses
*Speaker note: Walk through each encapsulation step in order, pointing out what is added at each layer and naming the PDU at each stage.*

**Slide 10 — Lab Evidence: Encapsulation in a Real Packet**
- Ethernet II header (source and destination MAC) wrapping the IPv4 header (`10.0.2.15` → `10.0.2.3`, protocol 17 = UDP, TTL 64)
- UDP header (port 40486 → 53) wrapping the DNS standard query (transaction ID `0x1c67`)
- Wireshark's protocol nesting: `eth:ethertype:ip:udp:dns`
*Speaker note: Use the two screenshots to show the layers literally nested inside one another, from the frame down to the DNS message.*

**Slide 11 — Packet Journey: Across the Network and Back**
- Local network: AP → switch → router (NAT + firewall)
- Internet: multiple ISP/backbone routers; MAC changes at each hop while the IP stays the same
- Response follows the reverse path back to the laptop
*Speaker note: Stress that IP addresses stay constant end to end (aside from NAT) while MAC addresses change at every router hop — this is the most important concept of the section.*

**Slide 12 — Lab Evidence: MAC Changes, IP Stays the Same**
- Reply packet from `151.101.129.91` (the web server) to `10.0.2.15` (the laptop), TCP port 443 → 50992
- Ethernet source MAC `52:55:0a:00:02:02` is the local gateway, flagged as a locally administered address, not the server's own MAC
- The earlier DNS query used a different destination MAC, showing the Layer 2 address depends on the next hop
*Speaker note: Contrast the IP header, which names the real endpoints, with the Ethernet header, which only names the next hop.*

**Slide 13 — Packet Journey Diagram (Explained)**
- Diagram of devices, request and response arrows, and layer labels
- Highlights where encapsulation and decapsulation occur (laptop and web server)
- Highlights address changes along the path
*Speaker note: Talk the audience through the diagram left to right, following the request out and the response back, pointing at each device as you describe what it does.*

**Slide 14 — Lab Evidence: The Page Loads**
- Browser shows `https://www.wikipedia.org` with the padlock icon
- DNS, TCP and TLS all completed before the first byte of the page arrived
- Journey complete: Application → Transport → Internet → Network Access outbound, reversed on the way back
*Speaker note: Close the packet journey by showing the end result, and tie it back to the earlier captures.*

**Slide 15 — Protocol Analysis Table**
- 4-layer protocol table (protocol, PDU, addressing, purpose)
- Application: HTTP/HTTPS (TLS), DNS; Transport: TCP and UDP; Internet: IPv4; Network Access: Ethernet, Wi-Fi, ARP
- Brief explanation of why each protocol is used
*Speaker note: Don't read the whole table aloud — pick two or three rows (for example TCP and IP) and explain them in depth as examples.*

**Slide 16 — TCP vs UDP Comparison**
- Key differences: connection-oriented vs connectionless, reliability, ordering, overhead
- TCP examples: HTTPS, email
- UDP examples: DNS, VoIP and video conferencing
*Speaker note: Explain why HTTPS needs TCP's reliability, then contrast with DNS's use of UDP for speed, so the comparison feels concrete.*

**Slide 17 — Communication Failure Scenario**
- Problem: user cannot reach `https://www.example.com`; the page times out while other sites work fine
- Symptoms: only one site fails, no certificate warning, so the connection never completes
- Root cause (revealed at the end): a firewall rule change blocking outbound TCP 443 to that server's IP range
*Speaker note: Present this like a mini mystery — describe the symptom first, then let the next slide show how it was diagnosed.*

**Slide 18 — Troubleshooting Steps**
- Flow: define the problem → check connectivity (ping the default gateway) → verify DNS (`nslookup`) → check the port (`Test-NetConnection` to TCP 443) → inspect devices and firewall logs → fix → confirm
- Fix: add an allow rule for outbound TCP 443; connection restored
*Speaker note: Walk through the flow as a logical, repeatable process that works up the layers and applies to other problems, not just this one.*

**Slide 19 — References**
- IETF RFC 9110 (HTTP Semantics) and RFC 9293 (TCP)
- Cisco Networking Academy course materials
- CompTIA Network+ study guide
- Microsoft Learn: TCP/IP protocol architecture
*Speaker note: Thank the audience, summarize in one sentence that every page load passes through four layers and is wrapped and unwrapped at each hop, and invite questions.*

---

---
## 8. References

1. **RFC 9110 – HTTP Semantics** (Internet Engineering Task Force, IETF) — the official standard defining HTTP.
2. **RFC 793 / RFC 9293 – Transmission Control Protocol (TCP)** (IETF) — the official standard defining TCP's connection and reliability mechanisms.
3. **Cisco Networking Academy – "Networking Basics" / CCNA Introduction to Networks course materials** (Cisco Systems) — covers TCP/IP and OSI models, encapsulation, and addressing in an educational format.
4. **CompTIA Network+ Certification Exam Objectives and Study Guide** (CompTIA) — covers TCP vs UDP, protocol layering, and troubleshooting methodology.
5. **Microsoft Learn – "TCP/IP Protocol Architecture"** (Microsoft) — vendor documentation on TCP/IP implementation and NAT/firewall behavior in enterprise networks.

*(The team should look these up directly and format citations per their required style, e.g., APA or IEEE.)*
