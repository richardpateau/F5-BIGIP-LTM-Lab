# F5 BIG-IP LTM Enterprise Load Balancing & Traffic Management Lab (Azure)

A 12-module, packet-verified **Application Delivery Controller (ADC)** engineering lab built on **F5 BIG-IP Virtual Edition (TMOS 21.1.0.3)** in **Microsoft Azure**. Every module progresses from architectural design and GUI/`tmsh` configuration to empirical validation using `wrk` load benchmarks, `curl` tests, backend NGINX logs, TMOS kernel telemetry, and **Wireshark `.pcap` analysis** (including decrypted TLS 1.2/1.3 streams and F5 Ethernet Trailer inspection).

---

## Tech Stack & Lab Environment

| Component | Specification |
|---|---|
| **Cloud Platform** | Microsoft Azure (`East US`, `vnet-eastus-1` / `snet-eastus-1` `172.16.0.0/24`) |
| **Load Balancer / ADC** | F5 BIG-IP VE `21.1.0.3` (`Standard_D2as_v4` — 2 vCPU, 8 GB RAM, LTM Licensed) |
| **Backend Servers** | 2x Ubuntu 24.04 LTS running NGINX (`HTTP/80` & `HTTPS/443`, self-signed TLS 1.2/1.3) |
| **Test Clients** | Ubuntu 24.04 LTS (`curl`, `wrk`, `openssl`, `/etc/hosts`) |
| **Analysis & Tooling** | `tmsh`, Wireshark (TLS `SSLKEYLOGFILE` decryption + F5 Ethernet Trailer), Draw.io |

---

## Lab Topology & IP Addressing

![Network Topology & Traffic Flow](screenshots/f5-read-me-draw-io.png)

| Node / Object | IP Address | Azure NIC Config | Role in Architecture |
|---|---|---|---|
| **F5-CLIENT-01** | `172.16.0.8` | Primary NIC | Primary client for `curl`, `wrk` benchmarks, and attack/rate-limit simulations |
| **F5-CLIENT-02** | `172.16.0.9` | Secondary Client | Concurrent client for multi-client Source Address Persistence verification |
| **VS-NGINX-WEB / HTTPS** | `172.16.0.7` | Secondary Static (`Second-ipv4`) | Shared Virtual Server VIP (`TCP/80` and `TCP/443`) |
| **F5 BIG-IP Self-IP** | `172.16.0.4` | Primary (`ipconfig1` + PIP) | Management (`8443`/SSH), active health probes, baseline Auto Map |
| **SNAT-POOL (`SNAT-1`/`2`)** | `172.16.0.10`, `172.16.0.11` | Secondary Static NIC IPs | Dedicated application egress SNAT pool isolating Self-IP `172.16.0.4` |
| **NGINX-WEB01** | `172.16.0.5` | Backend NIC | Backend App 1 (`POOL-NGINX-WEB`, `POOL-NGINX-HTTPS`, `POOL-WEB01`, `POOL-APP1-HTTPS`) |
| **NGINX-WEB02** | `172.16.0.6` | Backend NIC | Backend App 2 (`POOL-NGINX-WEB`, `POOL-NGINX-HTTPS`, `POOL-WEB02`, `POOL-APP2-HTTPS`) |

---

## Lab Modules & Navigation

### 1. SSL/TLS Architecture & Content Switching
| # | Module | Key Objects & Concepts | Verified Engineering Proof |
|---|---|---|---|
| 01 | [**SSL Termination (HTTPS Offloading)**](ssl-termination/README.md) | `CLIENTSSL-F5-LAB` (`f5-lab.local`), `IRULE-URI-ROUTING`, `POOL-WEB01`/`02` | Wireshark proof of **TLS 1.3** on client leg (`172.16.0.8 ↔ .7`) vs. **plaintext HTTP/1.1** on backend leg (`172.16.0.10/.11 → .5`) |
| 02 | [**SSL Bridging (End-to-End TLS)**](ssl-bridging/README.md) | `CLIENTSSL-F5-LAB`, `SERVERSSL-NGINX-LAB`, `MON-NGINX-HTTPS`, `IRULE-URI-ROUTING-HTTPS` | Wireshark proof of independent backend **TLS 1.2** handshakes (`nginx-web01.lab`) & encrypted Application Data (`content_type == 23`) |
| 03 | [**HTTP-to-HTTPS Redirection**](http-to-https-redirection/README.md) | `VS-NGINX-WEB:80`, `IRULE-HTTP-TO-HTTPS`, `HTTP::respond 301` | RFC-compliant `301 Moved Permanently` preserving `[HTTP::host][HTTP::uri]` (`/web01`), synthesized in TMM kernel in **110 µs** with zero backend hits |
| 04 | [**Layer 7 Host-Based Routing**](host-based-routing/README.md) | `IRULE-HOST-BASED-ROUTING`, `POOL-APP1-HTTPS`, `POOL-APP2-HTTPS` | Multi-tenant routing on shared VIP (`app1`/`app2.f5-lab.local`), decrypted TLS 1.3 `Host:` dissection, **47.6K** avg CPU cycles, zero-blast-radius outage isolation |

### 2. High Availability, Load Balancing & Session Persistence
| # | Module | Key Objects & Concepts | Verified Engineering Proof |
|---|---|---|---|
| 05 | [**HTTP Health Monitoring & Failover**](http-health-monitoring/README.md) | `MON-NGINX-HTTP` (`5s` interval / `16s` timeout, recv `"NGINX WEB"`) | Stopped NGINX on `WEB01` → evicted in ≤16s; `wrk` sustained **266,212 reqs @ 4,435 req/s** with **0 connect errors** on `WEB02` + automatic recovery |
| 06 | [**Least Connections Load Balancing**](least-connections/README.md) | `POOL-NGINX-WEB` (`Least Connections (node)`), Persistence `None` | Maintained **501 vs. 499** active socket equilibrium across 1,000 concurrent connections (even after 7.54M direct reqs skewed `WEB02`), shifting **995** active conns on failure |
| 07 | [**Source Address Persistence**](source-address-persistence/README.md) | `PERSIST-SOURCE-IP` (`source_addr`, `180s` sliding timeout) | `tmsh show ltm persistence persist-records` proved `172.16.0.8` pinned to `.6` while `172.16.0.9` pinned independently to `.5` |
| 08 | [**HTTP Cookie Insert Persistence**](cookie-persistence/README.md) | `PERSIST-COOKIE-LAB` (`HttpOnly`, `Secure`), Cookie Decoding | Decoded `83890348.47873.0000` → `172.16.0.5:443`; 100/100 requests stuck to `WEB02`; `persistence.pcap` proved **1,849 pkts** on `WEB02` vs. **50 monitor-only pkts** on `WEB01` |

### 3. Security Controls, Traffic Shaping & Cloud Networking
| # | Module | Key Objects & Concepts | Verified Engineering Proof |
|---|---|---|---|
| 09 | [**Layer 7 Security Controls (LTM Policies)**](security-controls/README.md) | `POLICY-HTTPS-SECURITY` (`BLOCK-SENSITIVE-URLS`, `BLOCK-UNSAFE-METHODS`) | Eliminated `/admin`, `/private`, `TRACE`, and `CONNECT` at TMM edge via TCP `RST` (**5 Invoked / 5 Succeeded** in `tmsh` & Wireshark), replacing `405` banner/cookie leaks |
| 10 | [**Connection & HTTP Rate Limiting**](http-rate-limiting/README.md) | `connection-limit 50` (L4), `IRULE-HTTP-RATE-LIMIT` (`table` 20 req/10s) | L4 clamped `wrk -c100` to **50** (`25`/`25`) with **13,261** `RST` rejects; L7 synthesized `HTTP 429` (`Retry-After: 10`) in **670 µs** on `tmm1` (83 execs, 0 failures) |
| 11 | [**SNAT Pools & Azure Multi-IP NIC**](snat/README.md) | Azure Secondary Static IPs, `SNAT-POOL` (`172.16.0.10`, `172.16.0.11`) | Migrated off Auto Map (`172.16.0.4`); Wireshark verified app egress exclusively on `.10`/`.11` while isolating `.4` to 1.5% health/SSH traffic |
| 12 | [**TMSH Operational Monitoring & Forensics**](f5-operational-monitoring/README.md) | `show ltm virtual`, `show ltm pool members`, CMP `all-cpus` | Audited **4.2M** HTTP & **494.4K** HTTPS reqs, **49.97% / 50.03%** symmetry across 638K reqs, and exact monitor failure timestamp (`@2026/11/19 02:16:09`) |

---

## Key Quantitative & Packet-Level Highlights

| Engineering Capability | Measurement / Evidence |
|---|---|
| **Edge 301 Redirect Turnaround (`IRULE-HTTP-TO-HTTPS`)** | **110 µs** (`0.000110s`) in TMM kernel (`s1/tmm0`), `0` backend packets |
| **Edge 429 Rate-Limit Synthesis (`IRULE-HTTP-RATE-LIMIT`)** | **670 µs** (`0.000670s`) in TMM kernel (`s1/tmm1`), `55.8K` avg CPU cycles |
| **Host-Based Routing Overhead (`IRULE-HOST-BASED-ROUTING`)** | **47.6K** avg CPU cycles (`0` failures / aborts) |
| **Live Socket Equilibrium (`Least Connections`)** | **501 vs. 499** concurrent sockets under 1,000-conn `wrk` load (`995` on failover) |
| **Failover Availability Under Outage (`MON-NGINX-HTTP`)** | **266,212 reqs @ 4,435.49 req/s** with **0** connect errors during `WEB01` outage |
| **Stateless Cookie Stickiness (`persistence.pcap`)** | **100/100** `curl` hits & **1,849 vs. 50** backend packets (`WEB02` vs. `WEB01`) |
| **SNAT Pool Isolation (`snat-post-change.pcap`)** | `100%` app traffic moved to `.10`/`.11`; Self-IP `.4` reduced to `13/844` pkts (`1.5%`) |

---

## Packet Capture (`.pcap`) Artifact Index

All supporting Wireshark captures are preserved in the repository for independent verification:

| Capture File | Module | Key Verification |
|---|---|---|
| [`http-termination.pcap`](ssl-termination/pcaps/http-termination.pcap) | SSL Termination | TLS 1.3 client leg vs. cleartext `HTTP/1.1` backend leg + SNAT `.10`/`.11` |
| [`persistence.pcap`](cookie-persistence/pcaps/persistence.pcap) | Cookie Persistence | 1,800 client packets pinned to `NGINX-WEB02` (1,849 pkts vs. 50 on `WEB01`) |
| [`redirection.pcap`](http-to-https-redirection/pcaps/redirection.pcap) | HTTP-to-HTTPS Redirect | `HEAD /` → `HTTP/1.0 301` (`Location: https://...`) in `110 µs` |
| [`host-based.pcap`](host-based-routing/pcaps/host-based.pcap) | Host-Based Routing | Decrypted TLS 1.3 `Host: app1.f5-lab.local` & `app2.f5-lab.local` + F5 trailer |
| [`policies.pcap`](security-controls/pcaps/policies.pcap) | L7 Security Controls | Immediate TCP `[RST]` termination on blocked URI/method probes |
| [`snat.pcap`](snat/pcaps/snat.pcap) | SNAT Baseline | Auto Map translation using BIG-IP Self-IP `172.16.0.4` |
| [`snat-post-change.pcap`](snat/pcaps/snat-post-change.pcap) | SNAT Pool | Dedicated source translation via `172.16.0.10` & `172.16.0.11` |

---

## Repository Structure

```text
f5-big-ip-ltm-lab/
├── README.md                       # Portfolio overview & module index (this file)
├── ssl-termination/                # HTTPS offloading, certs, URI routing & pcaps
├── ssl-bridging/                   # End-to-end TLS termination & backend re-encryption
├── http-health-monitoring/         # L7 HTTP probing (MON-NGINX-HTTP) & failover benchmarks
├── least-connections/              # Dynamic socket balancing & uneven background load tests
├── source-address-persistence/     # L4 IP affinity & dual-client tmsh persist-records
├── cookie-persistence/             # L7 Cookie Insert, decoding analysis & 100-req pcap proof
├── http-to-https-redirection/      # Edge 301 redirect iRule & 110 us turnaround pcap
├── host-based-routing/             # Multi-tenant Host-header routing & fault isolation
├── security-controls/              # TMM LTM Policy blocking /admin, /private, TRACE, CONNECT
├── http-rate-limiting/             # L4 connection-limit 50 & L7 table 429 rate limiting
├── snat/                           # Azure multi-IP NIC, Auto Map vs. SNAT-POOL pcaps
└── f5-operational-monitoring/      # tmsh telemetry, load symmetry audit & outage forensics
```
````

---

### Quick Cleanup Heads-Up Before Pushing Your Latest Edits
1. **Fix 3 remaining local `file:///` links** in your subfolder READMEs so `.pcap` links work on GitHub:
   * `cookie-persistence/README.md` → change `file:///home/rich/Documents/f5-big-ip-ltm-lab/cookie-persistence/pcaps/persistence.pcap` to `pcaps/persistence.pcap` (3 places).
   * `host-based-routing/README.md` → change `file:///home/rich/Documents/f5-big-ip-ltm-lab/host-based-routing/pcaps/host-based.pcap` to `pcaps/host-based.pcap` (2 places).
   * `http-to-https-redirection/README.md` → change `file:///home/rich/Documents/f5-big-ip-ltm-lab/http-to-https-redirection/pcaps/redirection.pcap` to `pcaps/redirection.pcap` (1 place).
2. **Double-check two tiny image filenames** in `ssl-bridging/README.md`: your `tree` shows `web-01-172.16.0.5-ip-verification.png` and `web-01-172.16.0.6-ip-verification.png`, and your pasted `ssl-bridging/README.md` has them swapped under the `Web 01` / `Web 02` captions (plus a stray `|` at the end of `![PCAP Server Hello & Certificate](...png)|`).
3. **Check `http-rate-limiting/README.md`:** your `tree` shows `creating-i-rule.png`, `applying-i-rule.png`, etc., and it references `connection-limit.pcap` / `irule-rate-limit.pcap` (which aren't in `http-rate-limiting/pcaps/` in `tree`), which is totally fine if you just refer to them as local captures during testing rather than links.
