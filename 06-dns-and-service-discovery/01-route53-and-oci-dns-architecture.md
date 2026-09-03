# 01. Route 53 & OCI DNS Architecture

## 1. Problem
DNS is the front door of every cloud application. When architects misunderstand the mechanics of the Domain Name System, applications suffer from severe architectural issues: failure to route apex domains (`example.com`) to cloud load balancers due to RFC 1034 CNAME restrictions, internal microservices leaking private IP addresses to public DNS revolvers, or DNS query latency adding 100ms+ to initial client connections. Senior and Staff engineers must understand how cloud authoritative Anycast DNS platforms operate at the packet and protocol level.

## 2. Cloud Concept
### Authoritative DNS & Anycast Routing
- **Authoritative DNS**: The definitive system of record for a domain. Unlike recursive resolvers (e.g., Google `8.8.8.8` or Cloudflare `1.1.1.1`) that cache records temporarily, authoritative DNS servers hold the actual DNS zone files and provide authoritative answers (`AA` flag set).
- **Anycast BGP Routing**: Both AWS Route 53 and OCI DNS broadcast identical IP addresses from dozens of global edge data centers simultaneously using Border Gateway Protocol (BGP) Anycast. When a client resolver queries Route 53 or OCI DNS, Internet routers automatically route the UDP packet to the geographically closest physical DNS nameserver, delivering sub-20ms DNS query latency worldwide.

### Public vs. Private Hosted Zones & Split-Horizon DNS
- **Public Hosted Zone**: Accessible globally from the public internet. Resolves public service endpoints (CloudFront, Public Load Balancers, API Gateways).
- **Private Hosted Zone**: Associated strictly with one or more designated VPCs or VCNs. Queries originating from inside the VPC resolve private IP addresses; queries originating from the public internet receive `NXDOMAIN` (Non-Existent Domain).
- **Split-Horizon DNS**: Maintaining identical domain names (e.g., `api.company.com`) across both a public and private zone:
  - An internal EC2/Compute instance querying `api.company.com` resolves the internal private IP (`10.0.1.50`).
  - An external internet user querying `api.company.com` resolves the public load balancer IP (`54.210.10.20`).

### The Zone Apex & The ALIAS Record Innovation
- **The RFC 1034 Restriction**: The official DNS specification prohibits a `CNAME` record at the **Zone Apex** (the root domain, e.g., `example.com`), because an apex must contain `SOA` and `NS` records, and RFC 1034 forbids any other record type from coexisting with a `CNAME` on the same node.
- **The Problem in the Cloud**: Cloud load balancers (AWS ALB) and CDNs (CloudFront) dynamically scale their IP addresses and publish only DNS hostnames (e.g., `app-12345.us-east-1.elb.amazonaws.com`), never static IPs. Pointing `example.com` to an ALB using standard DNS was historically impossible.
- **The Cloud Solution: ALIAS Records**:
  - AWS and OCI introduced proprietary **ALIAS Records** (virtual records synthesized at the authoritative nameserver).
  - An ALIAS record points directly to a cloud resource (ALB, S3 bucket, CloudFront distribution).
  - When a resolver queries `example.com`, Route 53 or OCI DNS internally resolves the cloud resource's current IP address and returns a standard **A Record** to the client.
  - **Zero Query Charges**: Route 53 does not bill for DNS queries that resolve to AWS native resources via ALIAS records `[Doc: Amazon Route 53 Pricing, checked 2026-09-03]`.

## 3. Mental Model
Think of DNS resolution as calling a corporate directory:
- **A Record**: Looking up an employee's direct telephone number (returns a concrete IP address).
- **CNAME Record**: An operator saying, *"I don't have John's direct number; please hang up and call Mary instead"* (requires the client to execute a second round-trip DNS query).
- **ALIAS Record**: An intelligent operator who knows Mary is answering for John, transfers your call internally on their private switchboard, and connects you directly to the phone number without requiring you to hang up or make a second call.

## 4. Architecture Diagram
```text
SPLIT-HORIZON DNS & ALIAS RESOLUTION:

[External Internet Client]
          │
          ▼ 1. Query: api.corp.com
[Public Route 53 / OCI DNS Zone] ──► Returns ALIAS -> A Record: 54.210.10.20 (Public ALB)
                                      (Client connects to Public Ingress)

[Internal EC2 / OCI Compute Instance (10.0.1.5)]
          │
          ▼ 2. Query: api.corp.com
[VPC / VCN Private DNS Resolver (169.254.169.253 / 10.0.0.1)]
          │
          ▼ 3. Intercepted by Private Hosted Zone
[Private Route 53 / OCI Private Zone] ──► Returns A Record: 10.0.1.50 (Internal Private IP)
                                           (Traffic never leaves private cloud backbone!)
```

## 5. AWS Implementation
In AWS:
- **Amazon Route 53**:
  - 100% SLA for authoritative DNS resolution (one of the few 100% SLAs in AWS) `[Doc: Amazon Route 53 SLA, checked 2026-09-03]`.
  - Global Anycast network across dozens of edge locations.
  - Supports ALIAS records for: Application Load Balancers, Network Load Balancers, CloudFront distributions, S3 static website endpoints, and API Gateways.
  - **Route 53 Resolver (AmazonProvidedDNS)**: Resolves at the VPC base $+2$ address or the link-local IP `169.254.169.253`. Enforces a strict quota of **1,024 packets per second (PPS) per ENI** `[Doc: Amazon Route 53 Resolver Quotas, checked 2026-09-03]`.
  - **Route 53 Resolver Endpoints**: Inbound endpoints allow on-premises resolvers to query Route 53 Private Hosted Zones; Outbound endpoints allow VPC workloads to resolve on-premises corporate DNS domains over Direct Connect.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI DNS Service**:
  - Globally distributed Anycast authoritative DNS platform providing public and private DNS management.
  - Supports primary, secondary, and split-horizon private DNS views.
  - **OCI ALIAS Record Support**: OCI DNS natively supports ALIAS records, allowing zone apex domains (`example.com`) to map directly to OCI Load Balancers or OCI Object Storage endpoints without CNAME RFC violations `[Doc: OCI DNS Overview, checked 2026-09-03]`.
- **OCI VCN Private DNS & Resolvers**:
  - Every OCI VCN includes a private DNS resolver by default.
  - Subnets inherit a default DNS domain (e.g., `subnetname.vcnname.oraclevcn.com`), automatically assigning internal hostnames to launched instances.
  - **Private Views & Zones**: OCI separates private DNS into **Views** and **Zones**. A Private View can be associated with multiple VCNs across different compartments, allowing centralized internal name resolution across the entire tenancy.
  - **DNS Resolver Endpoints**: Like AWS, OCI Private Resolvers support **Listening Endpoints** (inbound from on-prem) and **Forwarding Endpoints** (outbound to on-prem corporate DNS).

## 7. Configuration
Comparing ALIAS record and Private Zone configuration in Terraform across AWS and OCI:

### AWS Route 53 Apex ALIAS Record (Terraform)
```hcl
resource "aws_route53_zone" "primary" {
  name = "mycompany.com"
}

# Apex domain pointing to Application Load Balancer via ALIAS
resource "aws_route53_record" "apex" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = "mycompany.com"
  type    = "A"

  alias {
    name                   = aws_lb.public_alb.dns_name
    zone_id                = aws_lb.public_alb.zone_id
    evaluate_target_health = true # Automatic health check integration!
  }
}
```

### OCI DNS Apex ALIAS Record (Terraform)
```hcl
resource "oci_dns_zone" "primary" {
  compartment_id = var.compartment_id
  name           = "mycompany.com"
  zone_type      = "PRIMARY"
}

# Apex domain pointing to OCI Load Balancer via ALIAS
resource "oci_dns_rrset" "apex" {
  zone_name_or_id = oci_dns_zone.primary.id
  domain          = "mycompany.com"
  rtype           = "ALIAS"

  items {
    domain = "mycompany.com"
    rdata  = oci_load_balancer.public_lb.ip_addresses[0] # Maps directly to LB IP
    rtype  = "A"
    ttl    = 300
  }
}
```

## 8. Data Flow
```text
Step-by-Step ALIAS Resolution Flow:
1. Client Browser queries local ISP resolver: "What is example.com?"
2. ISP Resolver queries Route 53 / OCI DNS Anycast Nameserver via UDP port 53.
3. Authoritative Nameserver inspects ALIAS record:
   * Recognizes target is an internal cloud Load Balancer.
   * Resolves current healthy IP addresses of the Load Balancer (e.g., 54.210.10.20, 54.210.10.21).
   * Generates synthetic standard 'A' record response.
4. Nameserver returns 'A' record to ISP resolver.
5. ISP resolver caches 'A' record and returns it to Client Browser.
6. Client Browser establishes TCP/TLS handshake directly with Load Balancer IP.
```

## 9. Security
- **DNSSEC (Domain Name System Security Extensions)**:
  - Both Route 53 and OCI DNS support DNSSEC signing.
  - Uses public-key cryptography to cryptographically sign DNS records, protecting clients against DNS spoofing, cache poisoning, and man-in-the-middle attacks.
- **Preventing Split-Brain DNS Leaks**:
  - Private Hosted Zones must never be exposed publicly.
  - Enforce VPC/VCN association rules to prevent internal hostnames (e.g., `db-primary.prod.internal`) from leaking to external resolvers.

## 10. Reliability
- **100% Availability SLA in Route 53**: Route 53 achieves 100% availability through redundant global Anycast IP blocks distributed across physically independent Autonomous System Numbers (ASNs).
- **Evaluate Target Health in ALIAS Records**: When `evaluate_target_health = true` is set on a Route 53 ALIAS record pointing to an ALB, Route 53 automatically inherits the health of the backend targets. If all targets behind an ALB in Region A fail, Route 53 marks the ALIAS record unhealthy and stops returning that ALB's IP.

## 11. Scaling
- **Overcoming the 1,024 PPS DNS Quota in AWS**:
  - Every EC2 instance network interface (ENI) enforces a hard limit of **1,024 packets per second** to the `AmazonProvidedDNS` resolver `[Doc: EC2 Quotas, checked 2026-09-03]`.
  - In high-throughput Kubernetes clusters (EKS), hundreds of pods making unbuffered DNS lookups will saturate this quota, causing random `UnknownHostException` errors.
  - *Remediation*: Deploy **NodeLocal DNSCache** as a DaemonSet in Kubernetes to cache DNS lookups locally on each node's loopback interface (`169.254.20.10`), absorbing 95%+ of DNS query traffic.

## 12. Observability
- **Route 53 Query Logging**: Streams 100% of DNS queries to CloudWatch Logs. Analyze query volume, top requested domains, and NXDOMAIN error rates.
- **OCI DNS Metrics**: Track `QueryVolume` and `ResponseLatency` per zone in OCI Monitoring.

## 13. Cost
- **Hosted Zone Fees**:
  - AWS Route 53 charges **\$0.50 per hosted zone per month** for the first 25 zones `[Doc: Amazon Route 53 Pricing, checked 2026-09-03]`.
  - Queries cost **\$0.40 per million queries** for standard queries, and **\$0.70 per million** for latency-based queries.
  - *Free ALIAS Queries*: Queries to Route 53 ALIAS records that map to AWS resources (ALB, CloudFront, S3) are **100% free of query charges**.
- **OCI DNS Pricing**:
  - OCI provides 1 million queries free per month, billing \$0.85 per million queries thereafter `[Doc: OCI Networking Pricing, checked 2026-09-03]`.

## 14. Failure Modes
- **The Negative Caching (SOA Minimum TTL) Storm**: A developer queries a newly created internal service hostname before the DNS record is published. The resolver receives `NXDOMAIN`. The resolver caches this negative response based on the zone's `SOA Minimum TTL` (often default 300 to 900 seconds). Even after the record is created, the application cannot connect for 15 minutes due to negative caching.
- **The Client-Side JVM TTL Freeze**: The Java Virtual Machine historically cached DNS lookups **forever** by default (`networkaddress.cache.ttl = -1`). If a cloud load balancer changes its IP during an autoscaling or failover event, the Java application continues sending traffic to the defunct IP address indefinitely until the JVM is restarted.

## 15. Troubleshooting
When DNS resolution fails or returns unexpected IPs:
1. **Bypass Local Caching with Direct Authoritative Queries**:
   ```bash
   dig @ns-1234.awsdns-50.org api.company.com +trace
   ```
2. **Inspect Negative Caching TTL**: Look at the SOA record TTL returned during `NXDOMAIN` responses.
3. **Verify Private Zone VPC Association**: Run `aws route53 get-hosted-zone --id <zone-id>` and confirm that the client's VPC ID is listed in the `VPCs` array.

## 16. Common Mistakes
- **Using CNAME for Zone Apex**: Attempting to create a `CNAME` for `mycompany.com` pointing to an ALB. Standard DNS servers reject this. You must use an **ALIAS** record.
- **Setting Excessively High TTLs for Dynamic Endpoints**: Setting a TTL of 86,400 seconds (24 hours) on a service endpoint that might fail over to another region. During an outage, clients will take 24 hours to receive the updated failover IP.

## 17. Trade-offs
| Record Type | Pros | Cons | Best Use Case |
| :--- | :--- | :--- | :--- |
| **A Record** | Instant, single-step resolution | Tied to static IP; requires manual updates | Static bare-metal hosts |
| **CNAME Record** | Flexible; maps domain to domain | Prohibited at Zone Apex; requires extra DNS RTT | Subdomains (`www`, `mail`) |
| **ALIAS Record** | Works at Zone Apex; Free queries; Auto-tracks cloud IPs | Proprietary to cloud provider (Route 53 / OCI DNS) | CloudFront, ALB, OCI Load Balancers |

## 18. Interview Questions
1. *Why does the DNS specification forbid a CNAME record at the zone apex (`example.com`), and how do AWS Route 53 and OCI DNS solve this problem technically?*
2. *How do you configure a split-horizon DNS architecture using Route 53 or OCI DNS, and what security advantage does it provide to internal microservices?*
3. *A Java microservice fleet running on Amazon EKS intermittently throws `UnknownHostException` during peak load. You verify that the external service is online and healthy. What is the root cause at the network layer, and how do you resolve it?*

## 19. Interview Answer
**Exemplary Answer to Question 3**:
> "This issue is a classic symptom of **AmazonProvidedDNS (Route 53 Resolver) PPS Quota Saturation** combined with **Java JVM DNS Caching behavior**:
>
> 1. **The Root Cause**:
>    - Every EC2 instance network interface (ENI) enforces a hard, non-adjustable quota of **1,024 packets per second (PPS)** for DNS queries to the link-local resolver (`169.254.169.253`).
>    - When an EKS node runs dozens of high-throughput Java pods, each opening connections to Redis, PostgreSQL, Kafka, and external APIs, aggregate DNS queries exceed 1,024 PPS.
>    - The Nitro hypervisor silently drops packets exceeding this limit. Java's DNS resolver times out and throws `java.net.UnknownHostException`.
>
> 2. **The JVM Negative Caching Trap**:
>    - By default, Java caches negative DNS lookup responses (`networkaddress.cache.negative.ttl`) for 10 seconds. Once a single dropped packet causes an `UnknownHostException`, Java refuses to retry for 10 seconds, cascading the failure.
>
> 3. **The Architectural Fix**:
>    - **Deploy NodeLocal DNSCache**: Deploy the Kubernetes `NodeLocal DNSCache` DaemonSet. It runs a lightweight CoreDNS instance on every worker node listening on a local loopback IP (`169.254.20.10`). Pods query NodeLocal DNSCache locally over Unix sockets/loopback without hitting the ENI network quota. Cache hit rates typically exceed 98%, completely eliminating hypervisor DNS drops.
>    - **Tune JVM DNS Settings**: In `$JAVA_HOME/jre/lib/security/java.security`, set:
>      ```ini
>      networkaddress.cache.ttl=30
>      networkaddress.cache.negative.ttl=2
>      ```
>      This enforces a 30-second cache for healthy lookups and caps negative failure caching to 2 seconds."

## 20. Hands-on Exercise
**Objective**: Demonstrate split-horizon DNS resolution differences between internal VPC instances and external internet queries.

### Verification Steps
1. Create a Public Route 53 zone for `test.corp` with an A record: `api.test.corp -> 54.210.10.20`.
2. Create a Private Route 53 zone for `test.corp` associated with your VPC, with an A record: `api.test.corp -> 10.0.1.50`.
3. From your local development laptop (public internet), run:
   ```bash
   dig api.test.corp +short
   ```
   *Result*: Returns public IP `54.210.10.20`.
4. SSH into an EC2 instance inside the VPC and run:
   ```bash
   dig api.test.corp +short
   ```
   *Result*: Returns internal private IP `10.0.1.50`, proving transparent split-horizon resolution.
