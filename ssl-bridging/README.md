# F5 BIG-IP LTM — SSL Bridging & Layer 7 URI Routing

## Overview

This lab demonstrates **SSL Bridging** (TLS termination and backend re-encryption) on **F5 BIG-IP Local Traffic Manager (LTM)** running in Azure.

In an SSL Bridging architecture, BIG-IP acts as a full TLS proxy:
1. **Client-Side TLS Termination:** BIG-IP terminates the incoming HTTPS connection from the client using a **Client SSL Profile** (`CLIENTSSL-F5-LAB`).
2. **Layer 7 Inspection & Routing:** Once decrypted, BIG-IP inspects the HTTP URI via an **iRule** (`IRULE-URI-ROUTING-HTTPS`) to select a specific backend pool member.
3. **Server-Side Re-Encryption:** BIG-IP initiates a **separate TLS 1.2 session** to the backend NGINX server using a **Server SSL Profile** (`SERVERSSL-NGINX-LAB`), preserving end-to-end encryption across the internal network.

---

## Architecture & IP Addressing

![Palo Alto](screenshots/ssl-bridging-draw-io.png)

| Component | IP Address | Port / Protocol | Role |
|---|---|---|---|
| **F5-CLIENT-01** | `172.16.0.8` | Ephemeral → `443` | HTTPS test client (`curl`) |
| **VS-NGINX-HTTPS** | `172.16.0.7` | `TCP/443` (HTTPS) | Virtual Server (Client & Server SSL profiles) |
| **F5 BIG-IP (Self-IP)** | `172.16.0.4` | Source NAT / Monitors | Health probes & backend TLS connections |
| **NGINX-WEB01** | `172.16.0.5` | `TCP/443` (HTTPS) | Backend App 1 (`/web01`) |
| **NGINX-WEB02** | `172.16.0.6` | `TCP/443` (HTTPS) | Backend App 2 (`/web02`) |

---

## Configuration Summary

### 1. HTTPS Health Monitor (`MON-NGINX-HTTPS`)
Created an active HTTPS monitor that performs a TLS handshake and validates an HTTP `GET` response containing `NGINX` every 5 seconds (16-second timeout):
- **Send String:** `GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n`
- **Receive String:** `NGINX`

![HTTPS Health Monitor Configuration](screenshots/creating-https-monitor.png)

---

### 2. HTTPS Pool (`POOL-NGINX-HTTPS`)
Configured both backend servers on port `443` using **Least Connections (member)** load balancing and attached `MON-NGINX-HTTPS`.

![HTTPS Pool Configuration](screenshots/creating-https-pool.png)

---

### 3. SSL Bridging Profiles & Virtual Server (`VS-NGINX-HTTPS`)
Configured both SSL profiles on `VS-NGINX-HTTPS` along with the `http` profile (required for L7 HTTP inspection):
- **Client SSL Profile:** `CLIENTSSL-F5-LAB` (presents certificate to `F5-CLIENT-01` and decrypts inbound TLS).
- **Server SSL Profile:** `SERVERSSL-NGINX-LAB` (parent `serverssl`, `Server Certificate: ignore` for self-signed lab backend certificates).

**Server SSL Profile**

![Server SSL Profile](screenshots/creating-ssl-profile.png)

**Virtual Server SSL Profiles**
 
![Virtual Server SSL Profiles](screenshots/virtual-server-adding-server-profile.png)

**Default Pool**

![Virtual Server Default Pool](screenshots/configuring-virtual-server-default-pool.png)

---

### 4. Layer 7 URI Routing iRule (`IRULE-URI-ROUTING-HTTPS`)
Because BIG-IP decrypts the client's TLS traffic, it can inspect `[HTTP::path]` and route requests to specific pool members before re-encrypting the traffic to the backend:

```tcl
when HTTP_REQUEST {
    if { [HTTP::path] starts_with "/web01" } {
        pool POOL-NGINX-HTTPS member 172.16.0.5 443
    } elseif { [HTTP::path] starts_with "/web02" } {
        pool POOL-NGINX-HTTPS member 172.16.0.6 443
    }
}
```

**iRule Definition**

![iRule Definition](screenshots/creating-i-rule.png)

**Attached to Virtual Server**

![Applying iRule](screenshots/applying-i-rule.png)

**Network Map**

![Network Map](screenshots/network-map.png)


---

## Validation & Packet Analysis

### 1. Client Functional Verification (`curl`)
Requests sent from `F5-CLIENT-01` (`172.16.0.8`) to `https://172.16.0.7/web01` and `/web02` were decrypted by BIG-IP, routed by URI path, and served over backend HTTPS:

```bash
curl -kL https://172.16.0.7/web01
# <h1>Welcome to NGINX-WEB01 (App 1)</h1>

curl -kL https://172.16.0.7/web02
# <h1>Welcome to NGINX-WEB02 (App 2)</h1>
```

![Successful Client Curl Output](screenshots/client-curl-successful.png)

---

### 2. Backend TLS 1.2 Handshake & Certificate Inspection (Wireshark)
Capturing traffic on BIG-IP (`ssl.pcap`) confirmed that BIG-IP (`172.16.0.4`) establishes independent **TLSv1.2** handshakes with both `172.16.0.5:443` and `172.16.0.6:443`. Inspecting the `Server Hello` and `Certificate` packet confirmed the backend presented `id-at-commonName=nginx-web01.lab`:

**Backend TLSv1.2 Handshakes** 

![PCAP Overview](screenshots/pcap-overview.png)

**Backend Server Certificate (`nginx-web01.lab`)** 

![PCAP Server Hello & Certificate](screenshots/pcap-server-hello-details.png)|

---

### 3. Encrypted Backend Application Data (`tls.record.content_type == 23`)
Filtering `ssl.pcap` for TLS Application Data records (`tls.record.content_type == 23 && tcp.port == 443`) confirmed that payload traffic between BIG-IP (`172.16.0.4`) and both backend servers (`172.16.0.5` / `172.16.0.6`) remained encrypted on the wire:

![Encrypted TLS 1.2 Application Data](screenshots/pcap-application-data-encrpyted-tls-1.2.png)

**Web 01 (172.16.0.5) IP Verification**

![PCAP Overview](screenshots/web-01-172.16.0.6-ip-verification.png)

**Web 02 (172.16.0.6) IP Verification**

![PCAP Overview](screenshots/web-01-172.16.0.5-ip-verification.png)

---

### 4. Backend NGINX Access Logs
Checking `/var/log/nginx/access.log` on `NGINX-WEB01` confirmed requests arrived from BIG-IP's Self-IP (`172.16.0.4`) after TLS decryption:

![NGINX-WEB01 Access Log](screenshots/sudo-tail-web-01.png)

---

## Key Engineering Takeaways
- **Why SSL Bridging vs. SSL Offloading:** SSL Bridging maintains encryption between BIG-IP and backend servers (required for zero-trust / compliance environments) while still allowing BIG-IP to inspect HTTP headers/URIs for iRules, WAF, or cookie persistence.
- **Two Separate TLS Sessions:** The client negotiates TLS with `VS-NGINX-HTTPS` (`CLIENTSSL-F5-LAB`), while BIG-IP negotiates a completely independent TLS 1.2 session with the backend server (`SERVERSSL-NGINX-LAB`).
- **Lab vs. Production Note:** In this lab, `SERVERSSL-NGINX-LAB` uses `Server Certificate: ignore` for self-signed backend certs. In production, the backend CA certificate should be imported into BIG-IP's Trusted Certificate Authorities to enforce backend certificate validation.
