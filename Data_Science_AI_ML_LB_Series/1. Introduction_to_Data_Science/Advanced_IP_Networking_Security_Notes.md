# Advanced IP Networking & Security — Complete Exam Notes (with Formulas & Examples)

---

# 1. Fundamentals of MPLS

## 1.1 What is MPLS and Why It Was Created
**MPLS (Multi-Protocol Label Switching)** is a technique that forwards packets through a network using **short, fixed-length labels** instead of performing a full destination-IP-address lookup (longest-prefix match) at every single router hop.
- Because it inserts a label **between the Layer 2 header and the Layer 3 (IP) header**, MPLS is often called a **"Layer 2.5"** technology.
- **"Multi-Protocol"** because it can carry many different Layer 3 protocols and Layer 2 payload types (IP, ATM, Frame Relay, Ethernet) over the same MPLS core.

**Why MPLS matters (exam points):**
- **Faster forwarding:** a simple label lookup/swap is computationally simpler and faster in hardware than a full IP routing-table lookup at every hop.
- **Traffic Engineering (MPLS-TE):** allows an operator to explicitly steer traffic along a specific path, rather than being limited to whatever path the IGP's shortest-path calculation happens to choose — enabling better utilization of available bandwidth across multiple links.
- **VPN support:** MPLS is the technology underpinning most carrier-grade **L2VPN and L3VPN** services.
- **QoS support:** labels carry an experimental (EXP)/Traffic Class field usable for per-hop QoS treatment.

## 1.2 Key MPLS Components
| Term | Meaning |
|---|---|
| **Label** | A short, fixed-length (20-bit) identifier inserted into the packet, used for forwarding decisions instead of the IP address |
| **LER (Label Edge Router)** | A router at the **edge** of the MPLS domain — the **ingress LER** classifies incoming packets and *imposes* (pushes) the first label; the **egress LER** *removes* (pops) the label and forwards the original IP packet onward normally |
| **LSR (Label Switch Router)** | A **core** router inside the MPLS domain — forwards packets purely based on the label (label swapping), never inspecting the original IP header |
| **LSP (Label Switched Path)** | The specific, predetermined path a labeled packet follows through the sequence of LSRs — conceptually similar to a virtual circuit |
| **FEC (Forwarding Equivalence Class)** | A group of packets that are all treated identically by the network (same path, same priority) — e.g., "all packets destined to subnet X" or "all packets requiring premium QoS" — the FEC is what determines which label gets assigned at the ingress LER |

## 1.3 MPLS Label Format (32 bits total)
```
┌───────────────────────┬─────────┬───┬──────────┐
│   Label Value (20 bits) │ EXP (3) │ S │ TTL (8)  │
└───────────────────────┴─────────┴───┴──────────┘
```
| Field | Size | Purpose |
|---|---|---|
| Label Value | 20 bits | The actual label used for forwarding decisions |
| **EXP / Traffic Class** | 3 bits | Used for QoS marking (similar role to DSCP, but at the MPLS layer) |
| **S (Bottom of Stack)** | 1 bit | Set to `1` if this is the **last (innermost)** label in a stack, `0` otherwise — MPLS supports **stacking multiple labels** on one packet |
| **TTL** | 8 bits | Same purpose as IP's TTL — decremented at each LSR hop, prevents infinite forwarding loops |

**Label Stacking:** MPLS allows multiple labels to be pushed onto a single packet (e.g., an outer "transport" label to route across the core LSPs, plus an inner "VPN" label identifying which customer's VPN the traffic belongs to) — this stacking is exactly what makes MPLS VPNs (Section 1.6) possible.

## 1.4 MPLS Operation — Label Push / Swap / Pop
```
   Ingress LER          LSR (core)          LSR (core)          Egress LER
  (Push Label)        (Swap Label)         (Swap Label)         (Pop Label)
        │                    │                    │                    │
 IP Packet   ──►  [L=100|IP]  ──►  [L=200|IP]  ──►  [L=300|IP]  ──►  IP Packet
 (unlabeled)         │                    │                    │      (unlabeled,
                      │                    │                    │       normal IP
                      └── label lookup ────┴── label lookup ────┘       routing resumes)
                          & swap                 & swap
```
1. **Ingress LER:** classifies the incoming IP packet into a **FEC** (based on destination, policy, QoS needs), and **pushes** the appropriate initial label.
2. **Core LSRs:** each looks up the **incoming label** in its Label Forwarding Information Base (LFIB), **swaps** it for the corresponding **outgoing label**, and forwards to the next hop — critically, **never examines the original IP header** at all.
3. **Egress LER:** **pops** (removes) the final label and forwards the now-bare IP packet using standard IP routing to its final destination.

### Penultimate Hop Popping (PHP)
As an optimization, the **second-to-last router** in the LSP (the "penultimate hop") often pops the label itself **before** forwarding to the egress LER — this saves the egress LER from having to do both a label lookup *and* an IP lookup; it only needs to do the IP lookup, since the label is already gone by the time the packet arrives.

## 1.5 Label Distribution
LSRs need a way to agree on which label means what — this is handled by a **signaling/distribution protocol**:
| Protocol | Purpose |
|---|---|
| **LDP (Label Distribution Protocol)** | The standard protocol LSRs use to exchange label bindings with their neighbors, building the LSPs largely following the IGP's existing shortest-path routes |
| **RSVP-TE (Resource Reservation Protocol – Traffic Engineering)** | An extension of RSVP used to signal **explicit paths with reserved bandwidth guarantees** — this is what actually enables MPLS Traffic Engineering, allowing operators to set up LSPs along paths *other* than the default IGP shortest path |

## 1.6 MPLS VPNs
| Type | Description | Key Concepts |
|---|---|---|
| **L3VPN** | Provider offers Layer 3 (IP routing) VPN service to customers, keeping each customer's routes completely separate | **VRF** (Virtual Routing and Forwarding — a separate, isolated routing table per customer on the PE router), **RD** (Route Distinguisher — makes potentially-overlapping customer IP prefixes globally unique when carried in MP-BGP), **RT** (Route Target — controls which VRFs import/export a given route), **MP-BGP** (Multiprotocol BGP, carries the VPN-IPv4/IPv6 routes between Provider Edge routers) |
| **L2VPN** | Provider offers Layer 2 (Ethernet/other L2) connectivity across the MPLS core | **Pseudowire** (point-to-point L2 circuit emulation, e.g. carrying Ethernet or TDM over MPLS), **VPLS** (Virtual Private LAN Service — provides multipoint Ethernet LAN emulation across the MPLS network, making geographically separate sites appear to be on one shared LAN) |

## 1.7 MPLS Traffic Engineering (MPLS-TE)
Without TE, traffic simply follows the IGP's shortest path — which can lead to some links being heavily congested while others sit underutilized. **MPLS-TE**, using RSVP-TE, lets an operator explicitly establish LSPs along **specific chosen paths with guaranteed bandwidth**, achieving much better overall network utilization and allowing fast automatic re-routing (via **Fast Reroute, FRR**) if a link along the path fails.

## 1.8 Summary of MPLS Benefits
- Faster, hardware-simple forwarding (label swap vs. full routing lookup).
- Enables carrier-grade **L2VPN/L3VPN** services.
- **Traffic Engineering** for optimal bandwidth utilization.
- Native **QoS support** via the EXP field.
- **Protocol-independent** core — can carry virtually any Layer 3/Layer 2 payload.

---

# 2. Network Design for Next Generation IP Cloud Network

## 2.1 The Shift to Cloud-Native Network Design
Modern telecom/service-provider networks are evolving away from dedicated, purpose-built hardware appliances toward **cloud-native, software-driven architectures**, mirroring the same virtualization/cloud trends seen in general IT — driven by the need for **faster service rollout, lower cost, and elastic scalability**.

## 2.2 NFV (Network Functions Virtualization)
**NFV** replaces dedicated physical network appliances (routers, firewalls, load balancers, and even mobile core elements like the EPC/IMS components covered earlier) with **software running as Virtual Network Functions (VNFs)** on standard, commodity servers.

### NFV Reference Architecture (ETSI MANO Framework)
| Component | Role |
|---|---|
| **NFVI (NFV Infrastructure)** | The underlying physical + virtual **compute, storage, and network** resources that host the VNFs |
| **VNF (Virtualized Network Function)** | The actual software-based network function itself (e.g., vRouter, vFirewall, virtualized EPC/IMS components) |
| **NFVO (NFV Orchestrator)** | Orchestrates resources across the *entire* infrastructure and manages complete **network services** (chains of multiple VNFs working together) |
| **VNFM (VNF Manager)** | Manages the **lifecycle** of specific VNF instances — instantiation, scaling up/down, healing after failure |
| **VIM (Virtualized Infrastructure Manager)** | Controls and manages the NFVI resources directly (e.g., **OpenStack** is a common VIM implementation) |

## 2.3 SDN (Software Defined Networking)
**SDN** separates the network's **control plane** (decision-making about where traffic should go) from the **data plane** (the actual forwarding of packets), centralizing control-plane intelligence into a **software controller** that can programmatically manage the entire network.

### SDN Architecture
```
   Application Layer   (Business apps, network services)
          │  Northbound API (often REST)
   Control Layer        (SDN Controller — e.g., OpenDaylight, ONOS)
          │  Southbound API (commonly OpenFlow)
 Infrastructure Layer   (Physical/virtual switches — simple forwarding only)
```
- **Northbound API:** interface between the Controller and applications/services above it — lets applications request specific network behavior.
- **Southbound API:** interface between the Controller and the underlying switches — **OpenFlow** is the most well-known southbound protocol, letting the controller directly program each switch's forwarding table.

**Why SDN+NFV matters for telecom (exam point):** together they enable **rapid deployment of new network functions/services** (minutes, via software instantiation, instead of months of hardware procurement), flexible placement of functions (including at the network **edge via MEC**, as covered in the 5G notes), and are core enablers of **5G Core's Service-Based Architecture** and **network slicing**.

## 2.4 Data Center Network Design — Spine-Leaf Architecture
Modern cloud/telecom data centers have largely replaced the older 3-tier design (Core–Aggregation–Access) with a flatter, more scalable **Spine-Leaf (Clos) architecture**:

```
        Spine 1      Spine 2      Spine 3      Spine 4
          │  ╲    ╱   │  ╲    ╱    │  ╲    ╱    │
          │    ╲╱     │    ╲╱      │    ╲╱      │
          │    ╱╲     │    ╱╲      │    ╱╲      │
   Leaf 1 ─┴────┴───  Leaf 2 ──────  Leaf 3 ─────── Leaf 4
     │                    │                             │
  Servers               Servers                      Servers
```
- **Leaf switches:** connect directly to the servers/end devices.
- **Spine switches:** interconnect all the leaf switches — **every leaf connects to every spine**, so any leaf-to-leaf path is always exactly **two hops** (leaf→spine→leaf), giving predictable, low, and consistent latency.
- **ECMP (Equal-Cost Multi-Path)** routing is used across all the leaf-spine links simultaneously, providing both high aggregate bandwidth and automatic load distribution/redundancy.
- **Easy horizontal scalability:** need more capacity? Simply add more spine switches (increasing leaf-to-leaf bandwidth) or more leaf switches (increasing server-facing ports) — without redesigning the whole topology.

### Oversubscription Ratio (a key data center design metric)
```
Oversubscription Ratio = (Total downlink bandwidth to servers) : (Total uplink bandwidth to spine)
```
**Example:** A leaf switch with 48× 10Gbps server-facing ports (480 Gbps total) and 4× 40Gbps uplinks to the spine (160 Gbps total) has an oversubscription ratio of `480:160 = 3:1` — meaning, in a worst case where all servers transmit simultaneously at full rate, only 1/3 of that traffic can actually reach the spine at once. Lower ratios (closer to 1:1) mean less potential congestion but cost more (more uplink capacity needed).

## 2.5 Benefits of the Cloud-Native NGN Approach
- **Elasticity/Scalability:** capacity added/removed on demand via software, matching real-time load.
- **Faster time-to-market:** new services/network functions deployed in minutes via orchestration, not months of hardware procurement and installation.
- **Reduced CAPEX:** commodity server hardware replaces multiple types of expensive, purpose-built appliances.
- **Automation & self-healing:** orchestration platforms can automatically detect a failed VNF instance and spin up a replacement without manual intervention.
- Directly enables modern architectures covered elsewhere in these notes: **5G Core (SBA), network slicing, MEC/edge computing.**

---

# 3. IP Network QoS

## 3.1 Why QoS is Needed
Modern converged IP networks carry **voice, video, and data simultaneously**, each with very different sensitivity to network impairments. Without QoS, the network treats all traffic as **"best effort"** — first-come-first-served, with no differentiation — which is a serious problem when the network is congested, because delay-sensitive voice/video traffic gets stuck behind bulk data transfers.

## 3.2 Core QoS Parameters
| Parameter | Description | Most Sensitive Traffic |
|---|---|---|
| **Bandwidth/Throughput** | Amount of data that can be transferred per unit time | Video streaming, large file transfers |
| **Latency (Delay)** | Time for a packet to travel from source to destination | Voice, real-time gaming |
| **Jitter** | **Variation** in latency between successive packets | Voice, video (causes choppy audio/video if not buffered/compensated) |
| **Packet Loss** | Percentage of packets that fail to arrive | Voice, video (visible/audible artifacts); less critical for bulk data (TCP retransmits) |

**ITU-T G.114 recommended targets for acceptable voice quality (frequently examined numbers):**
| Metric | Target |
|---|---|
| One-way delay | **< 150 ms** (acceptable for most users); 150–400ms degrades interactivity; >400ms generally unacceptable |
| Jitter | **< 30 ms** |
| Packet Loss | **< 1%** |

## 3.3 QoS Models
| Model | Approach | Scalability |
|---|---|---|
| **Best Effort** | No QoS mechanisms at all — every packet treated identically | N/A (default, but poor for real-time traffic) |
| **IntServ (Integrated Services)** | **Per-flow** reservation using **RSVP** — each individual flow explicitly reserves guaranteed bandwidth/QoS end-to-end | **Poor** — every router must maintain per-flow state, which doesn't scale on large/core networks |
| **DiffServ (Differentiated Services)** | **Class-based** — packets are marked (via **DSCP**) into a small number of traffic classes; routers apply a defined **Per-Hop Behavior (PHB)** to each class, without needing to track individual flows | **Good** — the dominant QoS model in real-world IP/MPLS networks today |

## 3.4 DSCP (Differentiated Services Code Point) Marking
DSCP is a **6-bit field** (part of the IP header's old ToS byte, or IPv6's Traffic Class field), giving 64 possible marking values (0–63):

| DSCP Name | Decimal Value | Typical Use |
|---|---|---|
| **Default (BE)** | 0 | Best-effort traffic, no special treatment |
| **AF (Assured Forwarding)** classes AF11–AF43 | Various (e.g., AF11=10, AF21=18, AF31=26, AF41=34) | Data traffic requiring some priority, with defined drop-precedence sub-levels within each class |
| **CS5** | 40 | Voice **signaling** (e.g., SIP call setup messages) |
| **EF (Expedited Forwarding)** | **46** | Voice/video **bearer** traffic — the highest-priority class, designed for minimum latency/jitter/loss |
| **CS6 / CS7** | 48 / 56 | Network control traffic (e.g., routing protocol packets) |

## 3.5 Queuing/Scheduling Mechanisms (how routers decide what to transmit next when the link is busy)
| Mechanism | How it Works |
|---|---|
| **FIFO (First In, First Out)** | Simplest — packets transmitted strictly in arrival order, **no prioritization at all** |
| **PQ (Priority Queuing)** | Strict priority — higher-priority queues are *always* emptied completely before lower-priority queues get any service — risk of **starving** low-priority traffic entirely under heavy load |
| **WFQ (Weighted Fair Queuing)** | Allocates link bandwidth **proportionally** among flows/classes based on assigned weights — fairer than strict PQ, but doesn't guarantee true minimum latency for any one class |
| **CBWFQ (Class-Based WFQ)** | WFQ applied to defined **traffic classes** (rather than individual flows) — each class gets a guaranteed minimum bandwidth percentage |
| **LLQ (Low Latency Queuing)** | **The most common real-world choice** — combines a **strict-priority queue reserved specifically for voice/real-time traffic** (bounded, so it can't starve everything else) with **CBWFQ for all other classes** — gives voice the guaranteed low latency it needs while still fairly sharing remaining bandwidth among other traffic types |

## 3.6 Congestion Avoidance — RED / WRED
- **RED (Random Early Detection):** proactively and *randomly* drops a small percentage of packets **before** a queue becomes completely full, rather than waiting for a "tail drop" (dropping everything once the queue overflows). This avoids **TCP global synchronization** — a phenomenon where many TCP flows all back off (slow down) simultaneously in response to a tail-drop event, then all speed back up together, creating repeating waves of congestion.
- **WRED (Weighted RED):** applies RED with **different drop thresholds/probabilities per traffic class/DSCP marking** — so lower-priority traffic gets dropped earlier/more aggressively than higher-priority traffic as the queue fills.

## 3.7 Policing vs Shaping
Both mechanisms enforce a traffic contract (e.g., a customer's contracted bandwidth), but respond differently when traffic exceeds the allowed rate:

| | Traffic Policing | Traffic Shaping |
|---|---|---|
| Action on excess traffic | **Drops** (or re-marks to a lower priority) packets exceeding the rate | **Buffers/delays** excess packets, smoothing the flow out to conform to the rate over time |
| Effect | Can cause abrupt packet loss for bursty traffic | Adds some delay/jitter, but avoids outright dropping |
| Typical use | At network **ingress** — enforce a customer's contracted rate immediately | At network **egress** — smooth outgoing traffic to match a downstream link's capacity |

### The Token Bucket Algorithm (used by BOTH policing and shaping)
```
Tokens are added to a "bucket" at a defined rate (CIR).
A packet can be sent only if enough tokens are available (tokens ≥ packet size);
sending a packet consumes tokens equal to its size.
If insufficient tokens: policing → DROP the packet; shaping → QUEUE/DELAY the packet.
```
**Key formula:**
```
Tc (time interval) = Bc / CIR
```
where **CIR** = Committed Information Rate (the contracted average bit rate, bits/sec), **Bc** = Committed Burst Size (the maximum burst — in bits — allowed to be sent at once, filling the bucket), **Tc** = the time interval over which that burst is measured/replenished.

**Worked Example:** If CIR = 1 Mbps and Bc = 8,000 bits (1 KB), then:
```
Tc = Bc / CIR = 8,000 bits / 1,000,000 bps = 0.008 seconds = 8 ms
```
This means the traffic contract allows a burst of 8,000 bits within any 8ms window, while averaging to 1Mbps over time.

*(Advanced variant, often mentioned alongside: the **Two-Rate Three-Color Marker (trTCM)** uses both a CIR and a **PIR — Peak Information Rate** — to classify/color packets as Green (conforming), Yellow (exceeding CIR but within PIR), or Red (exceeding PIR), enabling more nuanced marking than simple pass/drop.)*

### Bandwidth-Delay Product (related buffer-sizing concept)
```
Bandwidth-Delay Product (BDP) = Bandwidth (bps) × Round-Trip Time (RTT, seconds)
```
Represents the amount of data that can be "in flight" on a network path at any given moment — used to size TCP receive windows and network buffers for maximum achievable throughput on long-distance/high-bandwidth links.

---

# 4. Proxy Services

## 4.1 What is a Proxy Server?
A **proxy server** is an intermediary that sits between a client and a destination server, forwarding requests and responses **on behalf of one side or the other** — the destination (or the client) may never directly interact with the true endpoint.

## 4.2 Types of Proxies
| Type | Sits Near... | Represents | Typical Use |
|---|---|---|---|
| **Forward Proxy** | The **clients** | The client side | Content filtering, caching, access control, hiding the client's real identity from external servers |
| **Reverse Proxy** | The **server(s)** | The server side | Load balancing across backend servers, SSL/TLS termination, caching, hiding backend server details/topology from external clients — this is the model used by **CDNs** |
| **Transparent Proxy** | Anywhere in the path | Neither explicitly — intercepts traffic automatically | ISP/gateway-level content filtering or caching, without requiring any client-side configuration |
| **Open Proxy** | Anywhere | Anyone who connects to it | Publicly accessible to any user — often a **security risk** if unintentionally exposed |

## 4.3 Functions of a Proxy Server
- **Caching:** stores frequently-requested content to reduce latency and backend/backbone load (see the Caching notes for hit-ratio formulas, replacement policies, etc.).
- **Content Filtering/Security:** blocks access to malicious or policy-violating websites/content.
- **Access Control & Authentication:** enforces which users/devices may access which resources.
- **Anonymity:** hides the client's real IP address from the destination server (forward proxy).
- **Load Balancing:** distributes incoming requests across multiple backend servers (reverse proxy).
- **SSL/TLS Termination:** a reverse proxy can decrypt incoming HTTPS traffic itself, **offloading** the computationally expensive cryptographic work from the backend application servers.
- **Logging & Monitoring:** central point to log/audit all traffic passing through.

## 4.4 HTTP Proxy vs SOCKS Proxy
| | HTTP Proxy | SOCKS Proxy |
|---|---|---|
| Protocol awareness | Understands HTTP/HTTPS specifically (can inspect/modify headers, cache content) | **Protocol-agnostic** — operates at a lower level (Session layer), can relay virtually any TCP/UDP traffic |
| Flexibility | Limited to web traffic | More general-purpose (email, FTP, gaming, any TCP/UDP application) |

## 4.5 Proxy vs VPN
| | Proxy | VPN |
|---|---|---|
| Scope of encryption | Typically encrypts/relays only **specific application traffic** configured to use it (e.g., just the browser) | Encrypts **all** traffic from the device at the OS/network level |
| Level of privacy/security | Lower — often no encryption at all (just relaying/hiding IP) | Higher — full tunnel encryption |
| Typical use case | Quick, application-specific IP masking, caching, content filtering | Full secure remote access / site-to-site connectivity (as covered in the Secured IP Network notes) |

---

# 5. DNS Services

## 5.1 What is DNS?
**DNS (Domain Name System)** is a hierarchical, distributed database system that translates human-readable **domain names** (e.g., `www.example.com`) into machine-usable **IP addresses**.

### DNS Hierarchy
```
                    "." (Root)
                       │
        ┌──────────────┼──────────────┐
      .com            .org           .in (ccTLD)      ← Top-Level Domain (TLD) servers
        │
   example.com                                          ← Authoritative Name Server
        │
  www.example.com                                        ← the actual record requested
```
- **Root servers:** the top of the hierarchy (13 logical root server clusters worldwide, labeled A–M), know only where to find the TLD servers.
- **TLD (Top-Level Domain) servers:** handle `.com`, `.org`, `.net`, country-code TLDs (`.in`, `.uk`), etc. — know where to find the authoritative servers for specific domains under them.
- **Authoritative Name Servers:** hold the actual, definitive DNS records for a specific domain (e.g., `example.com`).

## 5.2 Common DNS Record Types
| Record | Purpose |
|---|---|
| **A** | Maps a domain name to an **IPv4** address |
| **AAAA** | Maps a domain name to an **IPv6** address |
| **CNAME** | An **alias** — maps one domain name to another (canonical) domain name |
| **MX** (Mail Exchange) | Specifies the mail server(s) responsible for receiving email for the domain, with a priority value |
| **NS** | Specifies the **authoritative name servers** for a domain |
| **PTR** | Reverse DNS — maps an **IP address back to a domain name** (used in the special `in-addr.arpa` zone) |
| **TXT** | Arbitrary text data — commonly used for domain verification, SPF/DKIM email-authentication records |
| **SOA** (Start of Authority) | Administrative info about a zone: primary name server, admin contact, serial number, refresh/retry/expire/minimum-TTL timers |

## 5.3 DNS Resolution Process (Worked Example)
**Scenario:** A user's browser looks up `www.example.com` for the first time (nothing cached anywhere yet).

1. Client sends the query to its configured **recursive resolver** (e.g., the ISP's DNS server, or a public resolver like `8.8.8.8`).
2. Resolver has nothing cached, so it queries a **Root server** → Root server replies: *"I don't know the IP, but ask the `.com` TLD servers."*
3. Resolver queries the **`.com` TLD server** → TLD server replies: *"I don't know the IP, but `example.com`'s authoritative name servers are at these addresses."*
4. Resolver queries the **Authoritative Name Server for `example.com`** → it replies with the actual **A record** — the IP address for `www.example.com`.
5. Resolver **caches** this result (for the duration specified by the record's **TTL**) and finally returns the IP address to the client's browser.

## 5.4 Recursive vs Iterative Queries
| Query Type | Behavior |
|---|---|
| **Recursive** | The queried server does **all the work** itself and returns only the **final answer** (or a definitive "doesn't exist") to the requester — this is how a **client asks its resolver** |
| **Iterative** | The queried server gives back **the best answer it currently has, or a referral** to another server that might know more — the querier must then follow up itself — this is how a **resolver walks the hierarchy** (Root → TLD → Authoritative) |

## 5.5 DNS Zones & Zone Transfer
- A **zone** is a portion of the DNS namespace managed by a specific administrative entity, defined by a **zone file** on the **Primary (Master)** DNS server.
- **Secondary (Slave)** DNS servers hold a **read-only replica** of the zone, obtained via a **Zone Transfer** (**AXFR** for a full transfer, **IXFR** for an incremental/differential transfer) — providing redundancy and load distribution for DNS query handling.

## 5.6 DNS Security
| Mechanism | Purpose |
|---|---|
| **DNSSEC (DNS Security Extensions)** | Adds **digital signatures** to DNS records, allowing a resolver to cryptographically verify that a response genuinely came from the authoritative source and wasn't tampered with — protects against spoofing/cache poisoning |
| **DoT (DNS over TLS)** | Encrypts DNS queries/responses using TLS, over a dedicated port (853) |
| **DoH (DNS over HTTPS)** | Encrypts DNS queries/responses by tunneling them inside regular HTTPS traffic (port 443) — harder for a network operator to even distinguish from normal web traffic |

## 5.7 Common DNS Attacks
| Attack | Description |
|---|---|
| **DNS Spoofing / Cache Poisoning** | Injecting a **false DNS response** into a resolver's cache, redirecting users toward a malicious server instead of the legitimate one |
| **DNS Amplification Attack** | A **DDoS technique** exploiting the fact that a small DNS query can trigger a much larger response — attacker spoofs the victim's IP as the query source, so the large responses flood the victim |
| **DNS Tunneling** | Encoding and exfiltrating data (or establishing a covert command-and-control channel) hidden inside DNS query/response traffic, which is often allowed through firewalls with little scrutiny |

**Round Robin DNS (brief, related concept):** a simple load-balancing technique where a domain has **multiple A records** pointing to different server IPs; the DNS server cycles through returning them in different order to successive queries, spreading client load roughly evenly across the servers (though it has no awareness of actual server health/load).

---

# 6. Cyber Security and Firewall

## 6.1 CIA Triad (recap)
**Confidentiality** (only authorized access), **Integrity** (data not tampered with), **Availability** (reliable access when needed) — the foundation for evaluating any security control (see the Secured IP Network notes for the full breakdown of encryption, hashing, and redundancy mechanisms that achieve each).

## 6.2 Common Cyber Threat Categories
| Threat | Description |
|---|---|
| **Malware** | Malicious software — includes **viruses** (attach to & spread via files), **worms** (self-replicate across networks without needing a host file), **trojans** (disguise as legitimate software), **ransomware** (encrypts victim's data, demands payment), **spyware** (covertly monitors/exfiltrates data) |
| **Phishing / Social Engineering** | Manipulating a human into revealing sensitive information or taking a harmful action, rather than attacking a technical vulnerability directly |
| **DoS / DDoS** | Overwhelming a target to deny service to legitimate users (see the IP/Data Networks notes) |
| **MITM (Man-in-the-Middle)** | Secretly intercepting/altering communication between two parties |
| **SQL Injection** | Inserting malicious database query code via an application's unsanitized input fields, to manipulate/extract data from the backend database |
| **XSS (Cross-Site Scripting)** | Injecting malicious script code into a trusted website, which then executes in *other users'* browsers when they visit the compromised page |
| **Zero-Day Exploit** | An attack exploiting a vulnerability that is **not yet publicly known or patched** — no defense/signature exists yet |
| **APT (Advanced Persistent Threat)** | A prolonged, stealthy, targeted intrusion campaign (often nation-state or organized-crime sponsored) aiming for long-term undetected access rather than a quick smash-and-grab |

## 6.3 Firewall Generations (deeper look, building on Section 5 of the IP/Data Networks notes)
| Generation | Type | Key Capability |
|---|---|---|
| **1st Gen** | Packet Filtering | Stateless — examines header fields (IP, port, protocol) against static rules only |
| **2nd Gen** | Stateful Inspection | Tracks the **state/context** of active connections — far more secure than pure packet filtering |
| **3rd Gen** | Application-Layer / Proxy Firewall | Inspects actual **application-layer content** (Layer 7), acts as a full intermediary |
| **Next-Gen (NGFW)** | Modern integrated firewall | Combines stateful inspection + built-in **IPS**, application awareness (can identify specific apps regardless of port used), user-identity awareness, **SSL/TLS inspection** (decrypts and inspects encrypted traffic), and integrated threat intelligence feeds |

## 6.4 Firewall Deployment Architectures
| Architecture | Description |
|---|---|
| **Screened Host** | A single firewall/router filters traffic to/from a protected internal "bastion host" |
| **Screened Subnet (DMZ)** | The most common modern design — a dedicated **DMZ** segment (see the IP/Data Networks notes for the full DMZ discussion) sits between two firewall boundaries, hosting public-facing services while isolating the internal network |
| **Default Deny Policy** | Best-practice rule ordering: **deny everything by default**, then explicitly permit only the specific traffic that's genuinely required — far more secure than a "default allow, block known-bad" approach |

## 6.5 Modern Security Frameworks
### Defense in Depth
A **layered security strategy** — using multiple, independent security controls at different points (perimeter firewall, network segmentation, host-based antivirus/EDR, application-level security, data encryption, user security-awareness training) — so that if any **one** layer is breached or fails, the others still provide meaningful protection. No single control is relied upon exclusively.

### Zero Trust Architecture
A modern security model built on the principle **"Never trust, always verify"** — it explicitly **rejects the old assumption that anything "inside" the corporate network perimeter can be implicitly trusted**. Instead:
- Every access request is **authenticated and authorized individually**, regardless of whether it originates from inside or outside the traditional network boundary.
- Employs **micro-segmentation** — dividing the network into many small, isolated zones so that even if an attacker breaches one segment, lateral movement to others is blocked.
- Applies the **Principle of Least Privilege** — users/systems are granted only the minimum access necessary for their function.

## 6.6 Security Operations — SOC & SIEM
| Term | Meaning |
|---|---|
| **SOC (Security Operations Center)** | A centralized team/facility responsible for **continuous monitoring, detection, and response** to security incidents across an organization's entire IT/network environment |
| **SIEM (Security Information and Event Management)** | A software platform that **aggregates and correlates log/event data** from many different sources (firewalls, servers, applications, IDS/IPS) across the network, enabling the SOC team to detect patterns indicating an attack in real time that might be invisible looking at any single log source alone |

## 6.7 Telecom-Specific Security Considerations
- **SS7 Vulnerabilities:** SS7 (covered in the Signaling Fundamentals notes) was originally designed for a small, closed, mutually-trusted community of telecom operators. As interconnection has opened up globally, SS7's lack of built-in authentication has made it exploitable for **unauthorized subscriber location tracking, call/SMS interception, and fraud** — a well-documented, real-world telecom security concern.
- **GTP (GPRS Tunneling Protocol) Security:** used to tunnel user data through the mobile core (3G/4G), GTP has faced similar interconnect-related security exposure, especially at **roaming interfaces** between different operators' networks.
- **Diameter Protocol Security:** the signaling protocol that largely replaced older signaling in 4G/5G core networks; interconnect/roaming security remains an active area of concern, analogous to the SS7/GTP issues above.
- **DDoS Protection for Telecom Infrastructure:** given that telecom networks carry critical national infrastructure traffic (emergency services, financial transactions, general connectivity), dedicated large-scale **DDoS scrubbing/mitigation capacity** is a standard requirement for carrier-grade network security.

---

# Quick Revision Summary (Exam Cheat-Sheet)

- **MPLS:** "Layer 2.5," forwards via labels not IP lookups; components = Label/LER/LSR/LSP/FEC; label = 20(value)+3(EXP)+1(S)+8(TTL)=32 bits; operation = Push(ingress)→Swap(core)→Pop(egress); PHP saves the egress router a lookup; LDP=basic label distribution, RSVP-TE=explicit bandwidth-reserved paths for MPLS-TE; L3VPN uses VRF+RD+RT+MP-BGP, L2VPN uses Pseudowires/VPLS.
- **Cloud NGN Design:** NFV (VNF+NFVI+MANO: NFVO/VNFM/VIM) virtualizes network functions; SDN separates control/data plane (Northbound=apps, Southbound=OpenFlow); Spine-Leaf data center architecture gives predictable 2-hop latency + ECMP scalability; Oversubscription ratio = downlink BW : uplink BW.
- **QoS:** Best Effort vs IntServ (per-flow RSVP, doesn't scale) vs DiffServ (class-based DSCP, scales well — the real-world standard); DSCP: Default=0, EF=46(voice), CS5=40(voice signaling); queuing: FIFO→PQ(can starve)→WFQ/CBWFQ(proportional)→LLQ(priority for voice+CBWFQ for rest, the common real choice); RED/WRED avoid TCP global sync; Policing=drops excess, Shaping=buffers excess; Token Bucket: Tc=Bc/CIR; ITU-T G.114: voice delay<150ms, jitter<30ms, loss<1%.
- **Proxy:** Forward proxy=near clients (filtering/anonymity), Reverse proxy=near servers (load balancing/SSL termination/CDN model), Transparent=no client config needed; HTTP proxy=web-only, SOCKS=protocol-agnostic; Proxy≠VPN (VPN encrypts everything at OS level, proxy typically doesn't).
- **DNS:** Hierarchy = Root→TLD→Authoritative; Records: A/AAAA/CNAME/MX/NS/PTR/TXT/SOA; Recursive (client→resolver, full answer) vs Iterative (resolver→other servers, referrals); DNSSEC=signs records against spoofing; DoT/DoH=encrypt queries; watch for DNS amplification (DDoS) & cache poisoning attacks.
- **Cybersecurity/Firewall:** CIA triad recap; malware/phishing/DoS/MITM/SQLi/XSS/zero-day/APT threat types; Firewall gens: Packet Filter→Stateful→Application/Proxy→NGFW (adds IPS+app-awareness+SSL inspection); Default-Deny is best practice; Defense in Depth = multiple independent layered controls; Zero Trust = "never trust, always verify" + micro-segmentation + least privilege; SOC=monitoring team, SIEM=log correlation platform; Telecom-specific risk = SS7/GTP/Diameter interconnect vulnerabilities.
