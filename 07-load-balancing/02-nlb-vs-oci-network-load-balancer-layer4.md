# 02. NLB vs. OCI Network Load Balancer (Layer 4)

## 1. Problem
When systems operate at hyper-scale (processing hundreds of thousands of concurrent connections, streaming real-time IoT sensor telemetry, or running ultra-low-latency financial order books), the processing overhead of Layer 7 load balancing becomes a severe architectural bottleneck. An L7 reverse proxy must terminate TCP, decrypt TLS, buffer HTTP request bodies into memory, and initiate a second internal TCP connection. This introduces 2–5ms of latency, consumes significant memory, and requires complex "pre-warming" to survive instantaneous traffic spikes. Layer 4 Network Load Balancers eliminate this overhead entirely by operating directly at the transport layer.

## 2. Cloud Concept
A Layer 4 Network Load Balancer (NLB) routes packets at the **Transport Layer (OSI Layer 4)** using the IP 5-tuple:
$$\text{5-Tuple} = \langle \text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port}, \text{Protocol} \rangle$$

### Key Architectural Attributes of Layer 4 Load Balancing
1. **Pass-Through / Flow Hashing**:
   - The NLB does not terminate the TCP session or decrypt payloads. It hashes the incoming packet's 5-tuple to select a healthy target server.
   - Once a target is selected, all subsequent packets for that TCP flow are forwarded directly to that same target.
2. **Sub-Millisecond Wire-Speed Performance**:
   - Because packets are not buffered into application memory or parsed as HTTP streams, Layer 4 load balancers deliver deterministic sub-millisecond p99 latency (often $< 100\mu\text{s}$).
3. **Native Client Source IP Preservation**:
   - Unlike an L7 load balancer that overwrites the client IP with its own internal private IP via Source NAT (SNAT), an L4 load balancer preserves the original client IP in the IP packet header, delivering it directly to the target operating system.
4. **Instant Scalability Without Pre-Warming**:
   - Built on distributed data-plane fabrics (AWS Hyperplane, OCI SmartNICs), L4 load balancers can scale from zero to tens of millions of concurrent connections instantly, surviving massive DDoS floods or flash sales without operational intervention.

## 3. Mental Model
Think of Layer 4 vs. Layer 7 routing as railroad switching tracks:
- An **L7 Load Balancer** is an unloading warehouse: a train pulls in, workers unload every box from the train, open the boxes, inspect the labels, sort the items onto new pallets, reload them onto different trains, and dispatch them.
- An **L4 Network Load Balancer** is an **Automated Railroad Switch Track**: the train never stops or unloads. The automated mechanical switch simply flips the rail line based on the train's train car serial number, sending the entire train hurtling down Track 3 at full speed.

## 4. Architecture Diagram
```text
LAYER 4 PASS-THROUGH FLOW (ZERO BUFFERING, SUB-MILLISECOND LATENCY):

[Client: 203.0.113.50:52140]
             │
             ▼ Inbound TCP SYN Packet (Dst: 54.210.10.20:8080)
┌─────────────────────────────────────────────────────────────┐
│  CLOUD LAYER 4 NETWORK LOAD BALANCER                        │
│  AWS: NLB (Hyperplane Fabric) | OCI: Network Load Balancer  │
├─────────────────────────────────────────────────────────────┤
│  * Zero TLS Decryption (Payload untouched)                  │
│  * 5-Tuple Consistent Hashing: Hash(203.0.113.50, 52140...) │
│  * Selects Target Node 2 (10.0.1.25)                        │
│  * Preserves Original Source IP: 203.0.113.50               │
└─────────────────────────────┬───────────────────────────────┘
                              │ Wire-Speed Forwarding (< 100μs)
                              ▼
            [Backend Compute Node: 10.0.1.25:8080]
            * Linux OS socket sees remote IP: 203.0.113.50
            * Completes TCP handshake directly with client
```

## 5. AWS Implementation
In AWS:
- **AWS Network Load Balancer (NLB)**:
  - Built on **AWS Hyperplane**, a massively distributed, internal hardware-accelerated software-defined networking fabric that powers AWS NAT Gateways and PrivateLink `[Doc: AWS Hyperplane Technical Architecture, checked 2026-09-03]`.
  - Capable of processing **tens of millions of requests per second** without pre-warming.
  - Supports TCP, UDP, and TLS listeners.
  - **Static Elastic IP Addresses**: Allocates exactly one dedicated static public Elastic IP per enabled Availability Zone.
  - **Client IP Preservation**:
    - When targeting instances by `instance-id`, client IP is preserved natively.
    - When targeting by `ip` (e.g., EKS pods), client IP preservation can be toggled; if disabled, NLB uses SNAT, but can inject **Proxy Protocol v2** headers.
  - **Zonal Isolation**: NLB is engineered as independent zonal endpoints. If one AZ experiences failure, the remaining AZ endpoints continue operating with zero shared state.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Network Load Balancer (NLB)**:
  - **MAJOR ARCHITECTURAL & FINANCIAL HIGHLIGHT**:
  - In OCI, the **Network Load Balancer is 100% Free** of service management fees `[Doc: OCI Networking Pricing, checked 2026-09-03]`. Customers pay only for standard outbound data transfer.
  - Built directly on OCI's off-box SmartNIC virtualization layer.
  - **Non-Proxy Pass-Through Architecture**:
    - The OCI NLB is a true non-proxy load balancer. It does not act as a middleman socket.
    - Operates with **zero bandwidth throttling**: throughput is bounded only by the physical network interface limits of the backend compute shapes (e.g., up to 100 Gbps on bare metal).
  - **Flexible Hashing Algorithms**:
    - Supports 5-tuple (default: Src IP/Port, Dst IP/Port, Protocol).
    - Supports 3-tuple (Src IP, Dst IP, Protocol) for stateful client session affinity.
    - Supports 2-tuple (Src IP, Dst IP).
  - **Preserves Source IP by Default**: Backend servers always see the genuine client source IP address.
  - **Source/Destination Header Preservation**: Perfect for deploying third-party virtual firewall appliances (e.g., Palo Alto, Fortinet) in high-availability active-active clusters.

## 7. Configuration
Comparing L4 Network Load Balancer provisioning in Terraform across AWS and OCI:

### AWS Network Load Balancer (Terraform)
```hcl
# AWS L4 Network Load Balancer with Static Elastic IPs
resource "aws_eip" "nlb_eip" {
  count  = 3
  domain = "vpc"
}

resource "aws_lb" "l4_nlb" {
  name               = "high-throughput-nlb"
  internal           = false
  load_balancer_type = "network" # L4 primitive!

  # Bind dedicated static IPs per AZ
  subnet_mapping {
    subnet_id     = var.public_subnet_ids[0]
    allocation_id = aws_eip.nlb_eip[0].id
  }
  subnet_mapping {
    subnet_id     = var.public_subnet_ids[1]
    allocation_id = aws_eip.nlb_eip[1].id
  }
  subnet_mapping {
    subnet_id     = var.public_subnet_ids[2]
    allocation_id = aws_eip.nlb_eip[2].id
  }

  tags = { Name = "prod-nlb" }
}

# TCP Pass-through listener on Port 8080
resource "aws_lb_listener" "tcp_listener" {
  load_balancer_arn = aws_lb.l4_nlb.arn
  port              = 8080
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp_targets.arn
  }
}
```

### OCI Network Load Balancer (Terraform)
```hcl
# OCI L4 Network Load Balancer (100% Free Service!)
resource "oci_network_load_balancer_network_load_balancer" "l4_nlb" {
  compartment_id = var.compartment_id
  display_name   = "high-throughput-oci-nlb"
  subnet_id      = var.public_subnet_id # Regional Subnet!

  is_private                     = false
  is_preserve_source_destination = true # Native IP preservation!
}

# Backend Set with 5-Tuple Hashing
resource "oci_network_load_balancer_backend_set" "tcp_backend_set" {
  name                     = "tcp-backend-set"
  network_load_balancer_id = oci_network_load_balancer_network_load_balancer.l4_nlb.id
  policy                   = "FIVE_TUPLE"

  health_checker {
    protocol           = "TCP"
    port               = 8080
    interval_in_millis = 10000
    timeout_in_millis  = 3000
    retries            = 3
  }
}
```

## 8. Data Flow
```text
Layer 4 Packet Flow (Pass-Through vs. Proxy Protocol):
1. Inbound Packet: [Src: 203.0.113.50:52140] ──> [Dst: 54.210.10.20:8080]
2. NLB Hyperplane / OCI SmartNIC evaluates 5-tuple hash.
3. Target 10.0.1.25 selected.
4. Target Delivery Mode:
   - Mode A (Direct Pass-Through): Packet delivered to 10.0.1.25 with IP header untouched!
   - Mode B (Proxy Protocol v2): NLB prepends 16-byte binary header:
     [PROXY TCP4 203.0.113.50 54.210.10.20 52140 8080] + [Original TCP Payload]
5. Target OS reads original client IP directly from socket or Proxy Protocol header.
```

## 9. Security
- **Target Security Group Ingress Requirements**:
  - In an L7 ALB, the target security group allows traffic **only from the ALB's Security Group ID** (because the ALB replaces the client IP with its own private IP).
  - In an L4 NLB with client IP preservation, **the backend target security group must allow traffic from the client's actual public IP CIDR** (e.g., `0.0.0.0/0`) on the target port, because the packet retains the client's real source IP when arriving at the instance!
- **WAF Absence at Layer 4**: L4 load balancers cannot run AWS WAF or OCI WAF directly because WAF requires decrypting TLS and parsing HTTP payloads. If WAF inspection is mandatory, traffic must flow through an ALB, CloudFront, or an inspection proxy fleet.

## 10. Reliability
- **Cross-Zone Load Balancing on NLBs**:
  - By default, an AWS NLB distributes traffic **only to targets located in the same Availability Zone** where the client connection landed.
  - If AZ-1 has 2 healthy targets and AZ-2 has 10 healthy targets, AZ-1 targets will receive 5x more load!
  - *Reliability Rule*: Enable **Cross-Zone Load Balancing** on AWS NLB (`cross_zone_load_balancing.enabled = true`) to balance connections evenly across all targets in all AZs.
  - In OCI, the regional subnet architecture automatically eliminates zonal imbalance by default.

## 11. Scaling
- **Handling Flash Sales & DDoS Surges**:
  - Because an NLB does not maintain application-level buffers or proxy state, it can handle traffic surging from 1,000 to 5,000,000 requests/sec within seconds without dropping packets.
  - Ideal for IoT telemetry gateways, real-time gaming UDP servers, VoIP call signaling, and financial order routing.

## 12. Observability
- **NLB CloudWatch Metrics**:
  - `ActiveFlowCount`: Number of concurrent active TCP/UDP sessions tracked by Hyperplane.
  - `ProcessedBytes`: Total volume of payload bytes routed.
  - `TCP_Client_Reset_Count` / `TCP_Target_Reset_Count`: Identifies whether connection aborts are initiated by clients or backend servers.

## 13. Cost
- **AWS NLB Pricing**:
  - Hourly fee: **\$0.0225 per hour** ($\approx \$16.20/\text{month}$) `[Doc: AWS ELB Pricing, checked 2026-09-03]`.
  - Usage fee: **\$0.006 per NCLU-hour (Network Capacity Unit)**. An NCLU measures 800 new TCP connections/sec, 100,000 active connections, or 1 GB of data per hour. Substantially cheaper than ALB LCUs.
- **OCI NLB Pricing**:
  - **\$0.00 (100% Free)**. OCI charges zero hourly fees and zero capacity fees for the Network Load Balancer service `[Doc: OCI Networking Pricing, checked 2026-09-03]`.

## 14. Failure Modes
- **The Client IP Security Group Lockout**: An engineer migrates an application from ALB to NLB. They leave the target EC2 security group configured to allow traffic only from `sg-alb`. Because the NLB preserves the client source IP, incoming packets arrive from public IP addresses (e.g., `203.0.113.50`). The security group drops all incoming packets, causing a total application blackout.
- **Proxy Protocol Desynchronization**: Enabling Proxy Protocol v2 on an NLB target group without configuring the backend NGINX/Envoy server to accept Proxy Protocol. The backend server attempts to parse the binary Proxy Protocol header as a raw HTTP request, throwing `400 Bad Request` on every transaction.

## 15. Troubleshooting
When clients cannot connect through an NLB:
1. **Check Target Health**: Run `aws elbv2 describe-target-health` or `oci nlb backend-health get`.
2. **Inspect Backend Firewall Rules**: Does the backend target security group / NSG permit ingress from `0.0.0.0/0` (or client CIDRs) on the application port?
3. **Verify Health Check Port**: If health checking on TCP port 8080, ensure the application process is bound to `0.0.0.0:8080` and not localhost (`127.0.0.1:8080`).

## 16. Common Mistakes
- **Using NLB When Path-Based Routing is Needed**: Attempting to route `/api` to one cluster and `/static` to another using an NLB. NLB operates at Layer 4; it has zero concept of HTTP paths or URLs.
- **Forgetting Health Check Target Delays**: NLB TCP health checks only verify that the TCP port is open. If an application's database connection pool deadlocks, the TCP port remains open, and the NLB continues forwarding traffic to the broken node. Use **HTTP health checks** on NLB target groups to validate true application readiness.

## 17. Trade-offs
| Engineering Dimension | Layer 4 (AWS NLB / OCI NLB) | Layer 7 (AWS ALB / OCI LB) |
| :--- | :--- | :--- |
| **Latency** | Sub-millisecond (< 100μs) | 1–3ms (HTTP parsing overhead) |
| **Throughput / Scalability** | Millions of RPS; No pre-warming | Hundreds of thousands of RPS; Pre-warming required for massive surges |
| **Protocol Support** | TCP, UDP, TLS pass-through | HTTP, HTTPS, gRPC, WebSockets |
| **Routing Capability** | IP and Port only | Path, Host, Headers, Cookies, Query parameters |
| **Cost** | Free in OCI; Low NCLU fee in AWS | Hourly fee + LCU fee |

## 18. Interview Questions
1. *When architecting an ultra-low latency trading API processing 200,000 requests per second, why is an L4 NLB superior to an L7 ALB? What security group configuration change is mandatory when switching to an NLB?*
2. *Explain what Proxy Protocol v2 is, why it was invented, and when it must be enabled on a cloud load balancer.*
3. *Why does OCI offer its Network Load Balancer completely free of charge, and what architectural advantages does its pass-through design offer for third-party firewall appliances?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "An L4 Network Load Balancer (NLB) is superior to an L7 Application Load Balancer (ALB) for a 200,000 RPS trading API across three critical dimensions:
>
> 1. **Latency & Throughput**: An ALB must buffer incoming HTTP packets, parse the complete HTTP request stream, and establish a second TCP connection to the backend, adding 2–5ms of latency and consuming significant memory. The NLB operates at Layer 4 using hardware flow hashing on Hyperplane/SmartNICs, routing packets at wire speed with sub-millisecond p99 latency without payload buffering.
> 2. **Instant Spike Scaling**: An ALB scales out by launching internal proxy nodes behind DNS, which requires pre-warming to survive instant bursts. The NLB handles millions of concurrent flows instantly with zero pre-warming.
>
> **The Mandatory Security Group Change**:
> - With an ALB, the ALB performs Source NAT (SNAT), replacing the client's public IP with its own internal ENI IP. Thus, backend targets configure their Security Groups to allow traffic **strictly from the ALB's Security Group ID**.
> - With an NLB, client IP addresses are **preserved natively in the packet header**. When a packet arrives at the backend target, its source IP is the client's public IP (`203.0.113.50`).
> - Therefore, the backend target's Security Group **must be updated to allow traffic directly from the client's IP range (or `0.0.0.0/0`) on the application port**. If you leave the security group locked to an internal security group reference, the target will drop 100% of the NLB traffic."

## 20. Hands-on Exercise
**Objective**: Deploy a Layer 4 Network Load Balancer and observe native client IP preservation in backend logs.

### Verification Steps
1. Deploy an NLB forwarding TCP port 80 to an NGINX container.
2. In NGINX configuration, log `$remote_addr`.
3. Send an HTTP request from your external developer laptop: `curl http://<nlb-static-ip>/`.
4. Inspect NGINX access logs:
   *Expected Result*: The log reflects your actual home/office public IPv4 address, proving that the Layer 4 load balancer passed the packet through without SNAT rewriting.
