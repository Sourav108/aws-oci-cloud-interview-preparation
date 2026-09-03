# 02. TCP, UDP, TLS & The OSI Model in Cloud Systems

## 1. Problem
When backend systems scale to tens of thousands of requests per second, superficial understanding of the OSI model collapses into severe production incidents. Engineers diagnose application slowness as "database lag" when in reality the microservice fleet is suffering from ephemeral port exhaustion due to millions of sockets trapped in the Linux `TIME_WAIT` state, or packet retransmission storms caused by Maximum Transmission Unit (MTU) mismatches between virtual cloud networks. Furthermore, choosing between Layer 4 and Layer 7 load balancers without understanding TLS termination and TCP handshake latency introduces avoidable round-trip delays into every client interaction.

## 2. Cloud Concept
### Layer 4 (Transport) vs. Layer 7 (Application)
- **Layer 4 (Transport - TCP / UDP)**: Operates strictly at the transport packet layer without decrypting or inspecting payload content. Routing decisions are made solely based on the IP 5-tuple:
  $$\text{5-Tuple} = \langle \text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port}, \text{Protocol} \rangle$$
  - *AWS NLB / OCI Network Load Balancer*: Terminates or passes through TCP/UDP connections with sub-millisecond latency and ultra-high throughput (millions of connections/sec).
- **Layer 7 (Application - HTTP, HTTPS, gRPC, WebSockets)**: Operates at the application layer, parsing HTTP headers, cookies, query strings, and JSON payloads.
  - *AWS ALB / OCI Load Balancer*: Terminates TLS, inspects HTTP paths (`/orders` vs `/users`), evaluates host headers, injects `X-Forwarded-For`, and manages persistent keep-alive connection pools back to targets.

### The TCP Lifecycle in the Cloud
1. **The 3-Way Handshake**:
   $$\text{Client} \xrightarrow{\quad\text{SYN}\quad} \text{Server} \xrightarrow{\quad\text{SYN-ACK}\quad} \text{Client} \xrightarrow{\quad\text{ACK}\quad} \text{Server}$$
   Every new TCP connection requires 1 full round-trip time (1-RTT) before any application data can be sent.
2. **Socket States & TIME_WAIT**:
   When a server or client actively closes a TCP connection, the socket transitions into `TIME_WAIT` for $2 \times \text{MSL}$ (Maximum Segment Lifetime, typically 60 seconds in Linux). This prevents delayed packets from a previous connection from corrupting a new connection using the same 5-tuple.
   - If an application opens a new connection for every HTTP request instead of reusing connections (HTTP Keep-Alive), it will quickly exhaust all $\sim 60,000$ ephemeral ports (`net.ipv4.ip_local_port_range`), crashing the service.

### Maximum Transmission Unit (MTU) & Jumbo Frames
- **Standard Internet MTU**: $1,500\text{ bytes}$. Any packet larger than 1,500 bytes traversing the public internet will be fragmented or dropped if the `Don't Fragment (DF)` bit is set.
- **Cloud Jumbo Frames**: $9,001\text{ bytes}$ `[Doc: AWS EC2 Network MTU, checked 2026-09-03]`. Within an AWS VPC or OCI VCN, network interfaces support 9,001-byte frames, allowing applications to transfer 6x more data per packet, drastically reducing CPU interrupt overhead for high-throughput database replication.

### TLS 1.3 vs. TLS 1.2 Handshake
- **TLS 1.2**: Requires **2 full RTTs** (TCP Handshake + TLS ClientHello/ServerHello/KeyExchange) before transmitting HTTP GET/POST data.
- **TLS 1.3**: Requires only **1 full RTT** by combining cipher negotiation and Diffie-Hellman key exchange into the initial ClientHello. Supports **0-RTT Session Resumption** for returning clients.

## 3. Mental Model
Think of Layer 4 vs. Layer 7 routing as sorting physical letters:
- **Layer 4 (L4)** is an automated conveyor belt scanning only the barcode and zip code on the exterior envelope. It never opens the envelope. It routes 100,000 letters per minute without caring if the letter inside is written in English, French, or binary.
- **Layer 7 (L7)** is an inspector who opens the envelope, reads the letter, verifies the signature (TLS decryption), checks whether the letter requests a billing department or customer service (path-based routing), and rewrites the return address before taping the envelope shut and forwarding it. L7 provides immense intelligence, but processing takes significantly more time and CPU power.

## 4. Architecture Diagram
```text
Layer 4 Load Balancing (Ultra-Fast Pass-Through):
Client ──[SYN]──> [L4 NLB (Preserves Client IP)] ──[Pass-Through]──> [EC2 / OCI VM]
* No TLS Termination at LB. VM handles TLS and TCP termination directly.

Layer 7 Load Balancing (Reverse Proxy & Connection Pooling):
Client ──[TLS 1.3 Handshake]──> [L7 ALB / OCI LB (TLS Terminated)]
                                       │ (Persistent HTTP/2 Keep-Alive Pool)
                                       ▼ (Reused Sockets, Zero Handshake Overhead)
                                [Backend Container / Microservice]
```

## 5. AWS Implementation
In AWS:
- **Application Load Balancer (ALB)**: Layer 7 reverse proxy. Supports HTTP/1.1, HTTP/2, gRPC, and WebSockets. Manages dynamic TLS certificates via AWS Certificate Manager (ACM). Always performs SNAT, replacing the client source IP with the ALB's internal ENI IP (client IP preserved in `X-Forwarded-For` header).
- **Network Load Balancer (NLB)**: Layer 4 proxy built on the AWS **Hyperplane** distributed network fabric `[Doc: AWS Hyperplane Architecture, checked 2026-09-03]`. Capable of handling millions of concurrent connections and sudden volumetric spikes without pre-warming. Supports static Elastic IPs per AZ and preserves client source IP natively without proxy protocol headers.
- **Jumbo Frames in AWS**: EC2 Nitro instances support 9001 MTU within the VPC. Traffic traversing an Internet Gateway, VPC Peering across regions, or Virtual Private Gateway is automatically clamped to 1500 MTU.

## 6. OCI Implementation
In OCI:
- **OCI Load Balancer (Layer 7)**: Enterprise application reverse proxy providing **flexible bandwidth shaping** (e.g., dial in min/max bandwidth from 10 Mbps to 8,000 Mbps) `[Doc: OCI Load Balancing Service, checked 2026-09-03]`. Provides dedicated regional static IP addresses (unlike AWS ALB, which only exposes DNS names). Supports path-based routing, SSL/TLS offloading, and HTTP header modification.
- **OCI Network Load Balancer (Layer 4)**: Non-proxy, ultra-low latency pass-through load balancer. It does not terminate TCP/UDP connections; instead, it uses consistent hashing based on 5-tuple, 3-tuple, or 2-tuple to route packets directly to backend servers with **zero latency overhead** and native source IP preservation.
- **SmartNIC Off-Box Virtualization & MTU**: OCI's off-box architecture offloads all network encapsulation, security list processing, and MTU translation to custom SmartNIC cards, ensuring that 9000-byte jumbo frames achieve maximum wire speed without consuming VM CPU cycles.

## 7. Configuration
Configuring TCP socket reuse and Linux kernel buffer tuning for high-concurrency cloud workloads:

### Linux Kernel TCP Optimization (`/etc/sysctl.conf`)
```ini
# Increase ephemeral port range to prevent socket exhaustion
net.ipv4.ip_local_port_range = 1024 65535

# Enable TCP SYN Cookies to protect against SYN Flood DDoS attacks
net.ipv4.tcp_syncookies = 1

# Allow reuse of sockets in TIME_WAIT state for new outgoing connections
net.ipv4.tcp_tw_reuse = 1

# Decrease TIME_WAIT timeout from default 60s to 30s
net.ipv4.tcp_fin_timeout = 30

# Maximize socket listen backlog queue for bursty HTTP traffic
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 3240000

# Optimize TCP send and receive socket buffer sizes for high BDP networks
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

### ALB TLS 1.3 Security Policy (Terraform)
```hcl
resource "aws_lb_listener" "https_listener" {
  load_balancer_arn = aws_lb.app_alb.arn
  port              = "443"
  protocol          = "HTTPS"

  # Enforce modern TLS 1.3 / 1.2 with secure PFS ciphers
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}
```

## 8. Data Flow
```text
Step-by-Step TLS 1.3 Ingress Handshake:
1. Client ──[TCP SYN]──────────────────────────────────────────► [ALB / OCI LB]
2. Client ◄─[TCP SYN-ACK]────────────────────────────────────── [ALB / OCI LB]
3. Client ──[TCP ACK + TLS 1.3 ClientHello (Key Share)]────────► [ALB / OCI LB]
4. Client ◄─[TLS ServerHello + EncryptedExtensions + Cert]───── [ALB / OCI LB]
5. (1-RTT Completed: Encrypted Tunnel Established)
6. Client ──[Encrypted HTTP/2 GET /api/v1/checkout]────────────► [ALB / OCI LB]
7. [ALB / OCI LB decrypts, checks path /api/v1/checkout]
8. [ALB forwards over pre-warmed keep-alive TCP socket]────────► [App Container]
```

## 9. Security
- **Perfect Forward Secrecy (PFS)**: Modern TLS policies enforce Diffie-Hellman ephemeral key exchanges (`ECDHE`). Even if an attacker records encrypted network traffic for years and subsequently steals the server's private SSL key, they cannot retroactively decrypt past sessions.
- **ALPN (Application-Layer Protocol Negotiation)**: Negotiated inside the TLS ClientHello extension, allowing the browser and cloud load balancer to securely agree on `h2` (HTTP/2) or `http/1.1` without an extra round-trip.

## 10. Reliability
- **Connection Draining (Deregistration Delay)**:
  When an EC2 or OCI Compute instance is scheduled for termination during a scale-in event or deployment, the load balancer stops forwarding **new** TCP connections immediately, while keeping existing TCP sockets open for a configured window (e.g., 30–60 seconds) to allow in-flight transactions to conclude cleanly.
- **TCP Keep-Alive to Prevent Silent Drops**: Cloud NAT Gateways and stateful firewalls drop idle TCP connections after **350 seconds** in AWS `[Doc: Amazon VPC NAT Gateway Quotas, checked 2026-09-03]`. Backend applications communicating over long-lived sockets (e.g., database connection pools) must configure TCP keep-alive probes every 60 seconds to prevent firewalls from silently tearing down the connection.

## 11. Scaling
- **HTTP/2 Multiplexing**: Unlike HTTP/1.1 where browsers open 6–8 parallel TCP connections to load page assets, HTTP/2 multiplexes hundreds of concurrent requests over a **single persistent TCP connection**, drastically reducing connection overhead and memory usage on backend servers.
- **Bandwidth Delay Product (BDP)**:
  $$\text{BDP} = \text{Bandwidth (bits/sec)} \times \text{Round-Trip Latency (sec)}$$
  In long-distance cross-region cloud links (e.g., 10 Gbps pipe across 80ms latency), TCP socket buffers must be tuned to at least $\text{BDP} = 10^9 \times 0.08 = 100\text{ MB}$ to allow TCP to fully saturate the physical bandwidth.

## 12. Observability
- **TCP Socket State Metrics**: Run `ss -s` on production hosts to monitor socket allocation:
  ```text
  Total: 1250
  TCP:   850 (estab 420, closed 210, orphaned 0, timewait 180)
  ```
- **Load Balancer Reset Packets**: Monitor CloudWatch metrics `HTTPCode_ELB_5XX_Count` and `TargetConnectionErrorCount` to identify connection drops occurring before HTTP parsing.

## 13. Cost
- **L4 vs. L7 Cost Economics**:
  - AWS ALB charges based on **LCUs (Load Balancer Capacity Units)**, billing for new connections, active connections, processed bytes, and rule evaluations.
  - AWS NLB charges based on **NCLUs (Network Capacity Units)**, which are substantially cheaper for pure high-volume data streams (e.g., video streaming or log ingestion).
  - In OCI, the **Network Load Balancer is 100% free** of service management charges `[Doc: OCI Networking Pricing, checked 2026-09-03]`, making it an ideal, zero-cost choice for internal Kubernetes pod ingress.

## 14. Failure Modes
- **The Ephemeral Port Starvation Crash**: An application service calls an external microservice inside a high-throughput loop without connection reuse (`Keep-Alive: false`). Within 3 minutes, 60,000 sockets enter `TIME_WAIT`. The operating system throws `java.net.NoRouteToHostException: Cannot assign requested address`, crashing all outbound network calls.
- **The Blackhole MTU Mismatch**: An EC2 instance with 9001 MTU sends an 8,000-byte packet to an on-premises data center via IPsec VPN. The VPN tunnel has an MTU of 1400 bytes. If Path MTU Discovery (PMTUD) is broken because an over-zealous firewall blocked ICMP Type 3 Code 4 ("Fragmentation Needed"), the router silently drops the packet. Small ping packets succeed, but large HTTP POST requests hang indefinitely.

## 15. Troubleshooting
Diagnosing a hanging connection caused by Path MTU Discovery failure:
1. **Send Ping with Don't Fragment (DF) Bit Set**:
   ```bash
   # Test MTU limits incrementally
   ping -M do -s 1472 <target-ip>   # 1472 + 28 bytes ICMP/IP header = 1500 MTU
   ping -M do -s 8972 <target-ip>   # 8972 + 28 = 9000 Jumbo Frame
   ```
2. **Inspect ICMP Packet Filtering**: Verify that Security Groups and NACLs allow inbound ICMP Type 3 (Destination Unreachable) to enable automatic MTU negotiation.
3. **Inspect Sockets**: Run `netstat -natp | grep TIME_WAIT | wc -l` to verify if socket leakage is consuming the local port range.

## 16. Common Mistakes
- **Terminating TLS on Backend Application Servers**: Placing an L4 load balancer in front of microservices and terminating TLS inside Java or Node.js containers consumes significant container CPU and complicates certificate rotation. Terminate TLS at the cloud load balancer (ALB / OCI LB) to offload cryptographic compute to cloud hardware.
- **Using ALB for Non-HTTP TCP Protocols**: Attempting to route MQTT, raw binary streaming, or gaming UDP packets through an AWS ALB or standard OCI Load Balancer. ALB and standard OCI LB only support HTTP, HTTPS, gRPC, and WebSockets. Non-HTTP protocols mandate an L4 NLB.

## 17. Trade-offs
| Feature / Characteristic | Layer 4 (AWS NLB / OCI NLB) | Layer 7 (AWS ALB / OCI LB) |
| :--- | :--- | :--- |
| **Throughput / Latency** | Ultra-high throughput, sub-millisecond | High throughput, 1–3ms processing latency |
| **Payload Routing** | Limited to IP and Port | Deep inspection (Path, Host, Query, Headers) |
| **TLS Offloading** | Pass-through or TLS termination | Comprehensive TLS offloading + ACM integration |
| **Client IP Handling** | Preserved natively in TCP packet | Injected into `X-Forwarded-For` HTTP header |
| **Resource Overhead** | Minimal (Zero payload buffering) | Moderate (Buffers HTTP requests, terminates SSL) |

## 18. Interview Questions
1. *In an ultra-high-throughput payment API processing 50,000 RPS, why might you choose an AWS NLB over an ALB, and what trade-offs do you accept regarding header inspection and WAF integration?*
2. *Explain what happens at the network layer when an application logs `Cannot assign requested address`. How do you permanently eliminate this failure in a high-traffic microservices architecture?*
3. *Why does OCI Network Load Balancer operate with zero service management fees, and how does its pass-through architecture differ from an AWS Application Load Balancer?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Choosing an AWS Network Load Balancer (NLB) over an Application Load Balancer (ALB) for a 50,000 RPS payment engine is driven by performance, scalability, and deterministic latency:
>
> 1. **Why NLB Wins at 50,000 RPS**:
>    - **Zero Pre-Warming**: NLB is built on the AWS Hyperplane architecture, capable of handling instant, massive traffic spikes (e.g., flash sales) scaling to millions of connections per second without requiring pre-warming or scaling delays.
>    - **Sub-Millisecond Latency**: Operating at Layer 4, NLB does not parse HTTP headers, buffer request bodies, or decode JSON payloads. It routes raw TCP packets with sub-millisecond p99 latency.
>    - **Native Client IP Preservation**: NLB preserves the client's source IP address directly in the TCP packet header without requiring `X-Forwarded-For` header injection, simplifying fraud detection and audit logging.
>    - **Static Anycast IP Addresses**: NLB provides dedicated static Elastic IPs per AZ, allowing financial enterprise partners to whitelist static IPs in their firewalls.
>
> 2. **Accepted Trade-offs**:
>    - **Loss of L7 Routing**: NLB cannot route traffic based on URL paths (`/v1/charge` vs `/v1/refund`) or host headers; routing must be handled downstream by reverse proxies (Envoy / NGINX) or Kubernetes Ingress.
>    - **WAF Limitations**: AWS WAF cannot be attached directly to an NLB in standard mode (AWS WAF integrates natively with ALB or CloudFront). To enforce WAF inspection, we would need to terminate TLS downstream or place ALB behind NLB."

## 20. Hands-on Exercise
**Objective**: Trace TCP connection states and demonstrate ephemeral port exhaustion mitigation.

### Verification Steps
1. On an EC2 / OCI Linux instance, write a Python script that opens 5,000 sequential HTTP requests using `requests.get()` without an active `requests.Session()` (forcing new TCP handshakes per request).
2. Monitor socket exhaustion in real time:
   ```bash
   watch -n 1 'cat /proc/net/sockstat'
   ```
3. Observe `TCP: inuse` and `tw` (TIME_WAIT) surging to thousands of sockets.
4. Refactor the script to use `session = requests.Session()` with an HTTP Keep-Alive pool. Re-run and observe that socket count remains flat at 1 active connection, completely eliminating socket exhaustion.
