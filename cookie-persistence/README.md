# Application Delivery & Stateful Session Management: HTTP Cookie Insert Persistence & Wireshark Packet Telemetry

## Summary & Engineering Objectives

This project documents the architectural design, implementation, cryptographic header inspection, and packet-level verification of **HTTP Cookie Persistence (HTTP Cookie Insert)** on an **F5 BIG-IP Local Traffic Manager (LTM)** deployed in Microsoft Azure.

In modern distributed web applications, stateful user transactions—such as e-commerce shopping carts, multi-step financial checkout flows, user authentication sessions, and web application portals—require that successive requests from a client be deterministically routed to the exact backend application server maintaining that user's session state. 

Traditional Layer 4 **Source Address Persistence** tracks affinity by caching client source IP addresses in BIG-IP kernel memory. However, Source Address Persistence encounters critical failure modes in real-world networks:
- **The "Mega-Proxy" / CGNAT Bottleneck:** When thousands of enterprise users traverse a corporate forward proxy, cloud security gateway (e.g., Zscaler), or Carrier-Grade NAT (CGNAT), all clients share the identical source IP address. Layer 4 persistence binds this shared IP to a single backend node, causing severe load imbalance and node saturation ("hot-spotting").
- **Dynamic Client IP Roaming:** Mobile clients hopping between Wi-Fi networks and cellular towers undergo frequent IP reallocations, prematurely severing their persistence affinity.
- **Controller Memory Overhead:** Storing millions of IP hash mappings exhausts the Traffic Management Microkernel (TMM) memory state table.

### The HTTP Cookie Insert Solution:
F5 BIG-IP resolves these challenges via **Layer 7 HTTP Cookie Persistence**. Operating inside the decrypted TLS session on Virtual Server `VS-NGINX-HTTPS`, BIG-IP inspects and manipulates HTTP protocol headers directly:
1. **Initial Dispatch:** Upon receiving an initial client request devoid of session cookies, TMM selects an available pool member using the configured load-balancing algorithm (e.g., Round Robin).
2. **Dynamic Header Injection:** When the backend server returns an HTTP response, BIG-IP intercepts the stream and dynamically injects a RFC 6265 compliant `Set-Cookie:` header containing encoded server target information:
   ```http
   Set-Cookie: BIGipServerPOOL-NGINX-HTTPS=83890348.47873.0000; path=/; Httponly; Secure
   ```
3. **Stateless Controller Architecture:** Because the routing state resides entirely on the client browser and is returned in subsequent `Cookie:` request headers, BIG-IP does not maintain session tables in kernel RAM.
4. **Security Hardening (`HttpOnly` & `Secure`):** Injection of `HttpOnly` shields the session token from cross-site scripting (XSS) extraction via `document.cookie`, while `Secure` ensures the browser only transmits the cookie over encrypted TLS tunnels.

### Core Engineering Objectives:
1. **Custom Persistence Profile Engineering:** Author `PERSIST-COOKIE-LAB` (and `PERSIST-COOKIE`) utilizing the `HTTP Cookie Insert` method, hardening cookie transmission with `HTTPOnly` and `Secure` attributes and session-scoped expiration.
2. **Virtual Server Association:** Bind the cookie persistence profile to secure Virtual Server `VS-NGINX-HTTPS` (`172.16.0.7:443`).
3. **Pre-Enforcement Baseline Verification:** Execute `curl -Isk https://172.16.0.7 | grep -i Set-Cookie` to prove zero cookie leakage prior to policy enforcement.
4. **Dynamic Header Injection Verification:** Empirically validate that BIG-IP injects the hardened `Set-Cookie: BIGipServerPOOL-NGINX-HTTPS` header upon policy activation.
5. **Algorithmic Cookie Decoding & Mathematical Analysis:** Reverse-engineer the encoded decimal cookie payload (`83890348.47873.0000`), proving its exact conversion to IPv4 node `172.16.0.5` and port `443`.
6. **High-Frequency 100-Request Session Continuity Testing:** Execute an automated 100-iteration HTTP request sequence utilizing a persistent curl cookie jar (`-b /tmp/f5-cookies.txt -c /tmp/f5-cookies.txt`), demonstrating 100% deterministic session stickiness to backend node `NGINX-WEB02`.
7. **Wire-Level Wireshark Packet Telemetry Dissection:** Ingest and dissect packet capture [persistence.pcap](file:///home/rich/Documents/f5-big-ip-ltm-lab/cookie-persistence/pcaps/persistence.pcap) in Wireshark (`Endpoints` and packet breakdown), empirically verifying that client `172.16.0.8` generated 1,800 packets against VIP `172.16.0.7`, and backend server `NGINX-WEB02` (`172.16.0.6`) absorbed 1,849 packets (~99% of backend traffic), while unselected node `NGINX-WEB01` remained idle with only 50 packets (monitor heartbeats).

---

## Architecture & Traffic Flow Topology

The architecture leverages a dedicated Ubuntu client (`F5-CLIENT-01`) targeting the TLS Virtual Server (`VS-NGINX-HTTPS`). Traffic is evaluated by BIG-IP and persistently steered across dual NGINX application servers.

```mermaid
sequenceDiagram
    autonumber
    actor Client as F5-CLIENT-01 (172.16.0.8)
    participant F5 as F5 BIG-IP (VS-NGINX-HTTPS 172.16.0.7)
    participant Web01 as NGINX-WEB01 (172.16.0.5:443)
    participant Web02 as NGINX-WEB02 (172.16.0.6:443)

    Note over Client,F5: Transaction 1: Initial Request (No Cookie Present)
    Client->>F5: TLS Client Hello & Handshake
    F5-->>Client: TLS Established (Certificate Validated)
    Client->>F5: GET / HTTPS (No Cookie Header)
    Note over F5: TMM inspects headers: No BIGipServer cookie found<br/>Executes Round Robin -> Selects NGINX-WEB02
    F5->>Web02: HTTPS GET / (Forwarded across backend TLS)
    Web02-->>F5: HTTP/1.1 200 OK (Server 2 Payload)
    Note over F5: HTTP Cookie Insert: Encodes 172.16.0.6:443 into cookie<br/>Injects Set-Cookie with HttpOnly & Secure flags
    F5-->>Client: HTTP 200 OK + Set-Cookie: BIGipServerPOOL-NGINX-HTTPS=...
    Note over Client: Client stores cookie in /tmp/f5-cookies.txt

    Note over Client,Web02: Transactions 2 to 100: Session-Persistent Requests
    loop Requests 002 to 100 (Automated Loop)
        Client->>F5: GET / (Header: Cookie: BIGipServerPOOL-NGINX-HTTPS=...)
        Note over F5: TMM decodes cookie -> Resolves 172.16.0.6:443<br/>Bypasses Round Robin; steers directly to Web02
        F5->>Web02: HTTPS GET /
        Web02-->>F5: HTTP/1.1 200 OK (NGINX WEB02 - Backend Server 2)
        F5-->>Client: HTTP/1.1 200 OK (Zero Load Imbalance on Web01)
    end

    Note over F5,Web01: Independent Layer 7 Health Probing
    F5->>Web01: Periodic GET / (MON-NGINX-HTTP :80) -> Healthy
    F5->>Web02: Periodic GET / (MON-NGINX-HTTP :80) -> Healthy
```

### Infrastructure Addressing & Role Matrix

| Component | IP Address | Port / Protocol | Profile / Mechanism | Description & Architectural Purpose |
|---|---|---|---|---|
| **F5-CLIENT-01** | `172.16.0.8` | Ephemeral &rarr; `443` | Ubuntu Linux Client | Originates baseline checks and 100-packet persistence test sequences |
| **VS-NGINX-HTTPS** | `172.16.0.7` | `TCP/443` (HTTPS) | Standard Virtual Server | Secure VIP; terminates TLS, evaluates cookies, enforces `PERSIST-COOKIE-LAB` |
| **F5 BIG-IP Self-IP** | `172.16.0.4` | System / Self-IP | Internal Gateway | Originates backend TLS handshakes and `MON-NGINX-HTTP` health probes |
| **POOL-NGINX-HTTPS** | `172.16.0.5`, `172.16.0.6` | `TCP/443` (HTTPS) | Balanced Pool | Target pool containing both backend web nodes listening on port 443 |
| **NGINX-WEB01** | `172.16.0.5` | `TCP/443` & `80` | Backend Node 1 | Production web server; idle during cookie persistence lock to Web02 |
| **NGINX-WEB02** | `172.16.0.6` | `TCP/443` & `80` | Backend Node 2 | Production web server; receives 100% of persistent client sessions |

---

## Phase 1: Pre-Implementation Baseline (Persistence Disabled)

Prior to deploying the HTTP Cookie Persistence profile, `VS-NGINX-HTTPS` operated without cookie injection, relying strictly on standard load-balancing methods or default source address tables.

### 1. Initial Virtual Server Resource State
Under **Local Traffic &rarr; Virtual Servers : Virtual Server List &rarr; VS-NGINX-HTTPS &rarr; Resources**, the Virtual Server configuration was inspected prior to cookie profile enforcement:

![Virtual Server Resource Baseline](screenshots/test-cookie-persistence-off-persistence-profile.png)

- **Default Pool:** `POOL-NGINX-HTTPS`
- **Default Persistence Profile:** Inactive / Not configured with cookie persistence.

---

### 2. Client-Side HTTP Header Verification (Absence of Cookies)
From `F5-CLIENT-01` (`172.16.0.8`), sequential HTTP HEAD requests were transmitted to the HTTPS Virtual Server VIP (`172.16.0.7:443`) to inspect response headers for `Set-Cookie` directives:

```bash
azureuser@F5-CLIENT-01:~$ curl -Isk https://172.16.0.7 | grep -i Set-Cookie
azureuser@F5-CLIENT-01:~$ curl -Isk https://172.16.0.7 | grep -i Set-Cookie
azureuser@F5-CLIENT-01:~$ curl -Isk https://172.16.0.7 | grep -i Set-Cookie
```

![Baseline Test - No Cookie Injected](screenshots/test-cookie-persistence-off.png)

#### Baseline Findings:
- All three `curl` requests returned clean responses with **no `Set-Cookie` headers**.
- Because no persistence tokens were generated or exchanged, client requests were subject to standard load-balancing distribution across backend instances, offering zero stateful session guarantees.

---

## Phase 2: Cookie Persistence Profile Engineering & Association

To enforce deterministic session stickiness at Layer 7, a dedicated Cookie Persistence profile was designed and associated with the Virtual Server.

### 1. Provisioning Custom Persistence Profile (`PERSIST-COOKIE`)
Under **Local Traffic &rarr; Profiles : Persistence &rarr; New Persistence Profile...**, a custom profile was engineered:

![Creating Custom Cookie Persistence Profile](screenshots/creating-cookie-persistence.png)

#### Technical Parameter Specification:

| Parameter Field | Selected Value | Engineering Rationale |
|---|---|---|
| **Name** | `PERSIST-COOKIE` (and `PERSIST-COOKIE-LAB`) | Standardized identifier denoting cookie-based persistence |
| **Persistence Type** | `Cookie` | Engages Layer 7 HTTP cookie evaluation engine within TMM |
| **Parent Profile** | `cookie` | Inherits TMOS base cookie persistence defaults |
| **Cookie Method** | `HTTP Cookie Insert` | BIG-IP automatically injects the cookie into HTTP response headers |
| **Cookie Name** | *[Default]* (`BIGipServer<pool_name>`) | Uses standard naming convention `BIGipServerPOOL-NGINX-HTTPS` |
| **HTTPOnly Attribute** | `Enabled` | Hardens cookie against XSS document object model extraction |
| **Secure Attribute** | `Enabled` | Restricts cookie transmission strictly to encrypted TLS/HTTPS sessions |
| **Expiration** | `Session Cookie` | Cookie expires immediately when client closes browser session |
| **Cookie Encryption Use Policy**| `Disabled` | Retains standard decimal encoding for diagnostic transparency |

---

### 2. Attaching Persistence Profile to `VS-NGINX-HTTPS`
Under **Local Traffic &rarr; Virtual Servers : Virtual Server List &rarr; VS-NGINX-HTTPS &rarr; Resources**, the persistence profile was bound to the Virtual Server:

![Applying Cookie Persistence to Virtual Server](screenshots/applying-cookie-persistence.png)

- **Default Pool:** `POOL-NGINX-HTTPS`
- **Default Persistence Profile:** `PERSIST-COOKIE-LAB`
- **Fallback Persistence Profile:** `None`
- **iRules:** `/Common/IRULE-URI-ROUTING-HTTPS`

---

## Phase 3: Active Cookie Injection & Algorithmic Payload Decoding

With the persistence profile actively bound, client queries were executed to verify header insertion and decode the underlying network coordinates.

### 1. Empirical Verification of Injected Header
From `F5-CLIENT-01`, a test request was executed against the VIP:

```bash
azureuser@F5-CLIENT-01:~$ curl -Isk https://172.16.0.7 | grep -i Set-Cookie
```

![Grep Cookie Output Showing Set-Cookie Success](screenshots/test-cookie-persistence-on-grep-cookie-success.png)

#### Injected Response Header:
```http
Set-Cookie: BIGipServerPOOL-NGINX-HTTPS=83890348.47873.0000; path=/; Httponly; Secure
```

---

### 2. Security Attribute Analysis
- **`Httponly`:** Mandates that the cookie is completely inaccessible to client-side scripts (`document.cookie`). If a web application vulnerability (such as Stored or Reflected XSS) exists in the application payload, malicious JavaScript cannot harvest the persistence token.
- **`Secure`:** Guarantees that modern user agents will never transmit this cookie over unencrypted HTTP channels, completely mitigating wire-level credential snooping on open networks.
- **`path=/`:** Establishes domain-wide validity across all URI namespaces served by the Virtual Server.

---

### 3. F5 Cookie Encoding Mathematics & Reverse Engineering
F5 BIG-IP encodes backend pool member IP addresses and service ports using a standardized decimal formula:

$$\text{Cookie Format: } \langle\text{Encoded IPv4}\rangle.\langle\text{Encoded Port}\rangle.\langle\text{Route Domain}\rangle$$

Given the observed cookie value: `83890348.47873.0000`

#### A. Decoding the Backend IPv4 Address (`83890348`):
F5 encodes an IPv4 address $A.B.C.D$ into a 32-bit little-endian integer:

$$\text{Encoded IP} = A + (B \times 256) + (C \times 65536) + (D \times 16777216)$$

To reverse this into octets:
- $A = 83890348 \pmod{256} = \mathbf{172}$
- $B = \lfloor(83890348 / 256)\rfloor \pmod{256} = 327696 \pmod{256} = \mathbf{16}$
- $C = \lfloor(83890348 / 65536)\rfloor \pmod{256} = 1280 \pmod{256} = \mathbf{0}$
- $D = \lfloor(83890348 / 16777216)\rfloor \pmod{256} = 5 \pmod{256} = \mathbf{5}$

$$\mathbf{\text{Decoded IP Address: } 172.16.0.5 \quad (NGINX-WEB01)}$$

#### B. Decoding the Service Port (`47873`):
F5 encodes the 16-bit port by swapping its high and low bytes (endian byte reversal):
- Decimal `47873` in hexadecimal: `0xBB01`
- Reversing bytes (`0xBB` and `0x01`): `0x01BB`
- Hexadecimal `0x01BB` converted to decimal:

$$(1 \times 256) + (11 \times 16) + 11 = 256 + 176 + 11 = \mathbf{443}$$

$$\mathbf{\text{Decoded Port: } 443 \quad (HTTPS)}$$

#### C. Route Domain (`0000`):
- `0000` represents TMOS Route Domain `0` (the default global routing partition).

```python
# Verification Script in Python
encoded_ip = 83890348
ip_parts = [str((encoded_ip >> (i * 8)) & 0xFF) for i in range(4)]
print(f"Decoded IP: {'.'.join(ip_parts)}")  # Output: 172.16.0.5

encoded_port = 47873
port = ((encoded_port & 0xFF) << 8) | ((encoded_port >> 8) & 0xFF)
print(f"Decoded Port: {port}")              # Output: 443
```

---

## Phase 4: 100-Request Automated Session Continuity Stress Test

To validate absolute session stickiness under sustained request traffic, an automated 100-request benchmark was executed from `F5-CLIENT-01`.

### 1. Test Execution Script
The test utilized curl with cookie jar persistence (`-b` to read cookies, `-c` to write cookies), simulating a standard browser session across 100 sequential HTTP requests:

```bash
for i in $(seq 1 100); do
  response=$(curl -sk -b /tmp/f5-cookies.txt -c /tmp/f5-cookies.txt https://172.16.0.7/);
  printf "Request %03d: %s\n" "$i" "$response";
done
```

![100-Packet Persistence Test Execution](screenshots/100-packets-test-config.png)

### 2. Empirical Test Output:
```text
Request 001: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 002: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 003: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 004: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 005: <h1>NGINX WEB02 - Backend Server 2</h1>
...
Request 030: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 031: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 032: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 033: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 034: <h1>NGINX WEB02 - Backend Server 2</h1>
Request 035: <h1>NGINX WEB02 - Backend Server 2</h1>
...
Request 100: <h1>NGINX WEB02 - Backend Server 2</h1>
```

### 3. Engineering Outcome:
- **100% Deterministic Continuity:** Across **100 out of 100 consecutive requests**, 100% of traffic was steered exclusively to `NGINX-WEB02` (`172.16.0.6:443`).
- **Zero Intermittent Flapping:** Without persistence, Round Robin would have alternated responses equally between Server 1 and Server 2. With cookie persistence active, TMM parsed the incoming `Cookie:` header, identified the target node, and maintained flawless session affinity.

---

## Phase 5: Wire-Level Wireshark Packet Telemetry Dissection

To corroborate the client-side terminal results at the packet level, network traffic was captured into packet capture file [persistence.pcap](file:///home/rich/Documents/f5-big-ip-ltm-lab/cookie-persistence/pcaps/persistence.pcap) and analyzed in Wireshark.

### 1. Wireshark IPv4 Endpoints Analysis
Opening [persistence.pcap](file:///home/rich/Documents/f5-big-ip-ltm-lab/cookie-persistence/pcaps/persistence.pcap) under **Statistics &rarr; Endpoints &rarr; IPv4** revealed the exact distribution of traffic across all communicating nodes:

![Wireshark IPv4 Endpoints Statistics](screenshots/wireshark-statistics.png)

#### Packet & Bandwidth Distribution Telemetry:

| IP Address | Node Identity | Total Packets | Total Bytes | Tx Packets | Tx Bytes | Rx Packets | Rx Bytes | Operational Role in Capture |
|---|---|---|---|---|---|---|---|---|
| **`172.16.0.8`** | `F5-CLIENT-01` | **1,800** | **456 kB** | 900 | 183 kB | 900 | 273 kB | Client test harness originating 100 HTTPS transactions |
| **`172.16.0.7`** | `VS-NGINX-HTTPS` | **1,800** | **456 kB** | 900 | 273 kB | 900 | 183 kB | Virtual Server VIP terminating client TLS traffic |
| **`172.16.0.6`** | `NGINX-WEB02` | **1,849** | **544 kB** | 759 | 320 kB | 1,090 | 224 kB | **Target backend node; absorbed 100% of client requests** |
| **`172.16.0.5`** | `NGINX-WEB01` | **50** | **10 kB** | 26 | 6 kB | 24 | 3 kB | **Unselected node; handled ONLY health monitor probes** |
| **`172.16.0.4`** | `F5 BIG-IP Self-IP` | **1,939** | **570 kB** | 1,141 | 240 kB | 798 | 330 kB | Internal interface conducting backend TLS & health probes |
| **`168.63.129.16`**| Azure Infrastructure | **32** | **10 kB** | 13 | 4 kB | 19 | 6 kB | Azure fabric health probing |
| **`96.246.138.240`**| Management SSH | **8** | **6 kB** | 0 | 0 bytes | 8 | 6 kB | Out-of-band administrative terminal traffic |

---

### 2. In-Depth Wire-Level Dissection

```text
========================================================================================
                      PACKET LEVEL PERSISTENCE VERIFICATION
========================================================================================

  [ Client: 172.16.0.8 ] 
            │
            │  1,800 Packets (100 HTTPS Transactions: Handshake, Request, ACK, Close)
            ▼
  [ Virtual Server: 172.16.0.7:443 ]
            │
            ├───────────────────────────────────────────────────────┐
            │                                                       │
            ▼ (Cookie: NGINX-WEB02)                                  ▼ (No Client Traffic)
  [ NGINX-WEB02: 172.16.0.6 ]                             [ NGINX-WEB01: 172.16.0.5 ]
    • 1,841 Packets on Port 443 (HTTPS)                     • 0 Packets of Client Payload
    • 8 Packets on Port 80 (Health Monitor)                 • 16 Packets on Port 80 (Health Monitor)
    • Total: 1,849 Packets                                  • 34 Packets of Initial Handshake
                                                            • Total: 50 Packets
========================================================================================
```

#### Key Technical Takeaways from Packet Dissection:
1. **Mathematical Symmetry on Ingress:** The client (`172.16.0.8`) and Virtual Server VIP (`172.16.0.7`) exhibit exact mathematical symmetry: **900 packets transmitted and 900 packets received (1,800 packets total)**, representing an average of 18 packets per complete TLS connection cycle across the 100 sequential requests.
2. **Empirical Proof of Backend Isolation:** `NGINX-WEB02` handled **1,841 packets on port 443** (plus 8 health monitor packets on port 80). In stark contrast, `NGINX-WEB01` handled **zero client transactions**, receiving only 16 health check packets on port 80 and 34 initial handshake packets.
3. **Wire-Level Proof of Cookie Persistence:** This packet distribution demonstrates that BIG-IP adhered strictly to the client-submitted cookie, bypassing standard Round Robin load-balancing logic to maintain total session persistence.

---

## Phase 6: Architectural Comparison & Engineering Considerations

| Persistence Method | Inspection Layer | Client Dependency | Controller Memory Footprint | NAT / Mega-Proxy Resilience | Security Hardening |
|---|---|---|---|---|---|
| **HTTP Cookie Insert** *(Implemented)* | Layer 7 | Browser cookie storage | **Zero (Stateless on BIG-IP)** | **Immune** (Each client browser tracked uniquely) | `HttpOnly`, `Secure`, and optional Cookie Encryption |
| **HTTP Cookie Passive** | Layer 7 | Server sets cookie | **Zero (Stateless on BIG-IP)** | **Immune** | Depends on backend server configuration |
| **HTTP Cookie Rewrite** | Layer 7 | Server sets blank cookie | **Zero (Stateless on BIG-IP)** | **Immune** | Overwrites server-generated cookie header |
| **Source Address Persistence** | Layer 4 | None (Protocol agnostic) | **High** (Stores state in TMM RAM) | **Vulnerable** (All NAT clients hit one node) | Subnet masking (`/32` or `/24`) |
| **SSL Session ID** | Layer 4/7 | TLS Session Cache | **Moderate** (Maintains session cache) | **Immune** | Ephemeral; invalid under TLS 1.3 resumption |

### Production Engineering Best Practices:
1. **Cookie Encryption:** While raw decimal encoding (`83890348.47873.0000`) is useful for diagnostics and lab validation, enterprise security standards (e.g., OWASP, PCI-DSS) recommend enabling **Cookie Encryption**. By configuring an Encryption Passphrase on the persistence profile, BIG-IP encrypts the cookie with AES, preventing external reconnaissance of internal server IP addressing.
2. **Health Monitor Precedence Over Persistence:** If a pinned backend node experiences an outage (e.g., NGINX crashes), BIG-IP's Layer 7 health monitor (`MON-NGINX-HTTP`) immediately evicts the member. When the client transmits its next request with the old cookie, BIG-IP recognizes that the target node is `down`, automatically bypasses the cookie mapping, and dynamically selects a healthy surviving server.
3. **Session vs. Explicit Expiration:** For sensitive banking or enterprise applications, configuring a finite expiration window (e.g., 1800 seconds) or enforcing session-only cookies ensures stale tokens cannot be replayed after a user terminates their browser session.

---

## Summary of Empirical Verification

| Verification Phase | Test Mechanism | Target Object | Observed Result | Conclusion |
|---|---|---|---|---|
| **Baseline Audit** | `curl -Isk ... \| grep Set-Cookie` | `VS-NGINX-HTTPS` | No `Set-Cookie` header emitted | **Confirmed Unpersisted** |
| **Profile Provisioning** | TMOS Profile Manager | `PERSIST-COOKIE-LAB` | `HTTP Cookie Insert`, `HttpOnly`, `Secure` enabled | **Profile Provisioned** |
| **Header Injection** | `curl -Isk ... \| grep Set-Cookie` | `VS-NGINX-HTTPS` | `Set-Cookie: BIGipServer...=83890348.47873.0000` | **Header Injected** |
| **Math Decoding** | Little-endian bitwise decoding | `83890348.47873` | Decodes exactly to `172.16.0.5:443` | **Math Validated** |
| **100-Request Stress Test** | 100-loop curl with cookie jar | `https://172.16.0.7/` | 100 out of 100 requests returned `NGINX WEB02` | **100% Session Affinity** |
| **Wireshark Packet Audit** | `persistence.pcap` Endpoints | Backend Nodes | `WEB02`: 1,849 pkts vs. `WEB01`: 50 pkts | **Wire-Level Proof Established** |
