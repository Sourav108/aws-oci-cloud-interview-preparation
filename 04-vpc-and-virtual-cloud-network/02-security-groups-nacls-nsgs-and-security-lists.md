# 02. Security Groups, NACLs, NSGs & Security Lists

## 1. Problem
In production cloud engineering, conflating network security primitives is one of the leading causes of phantom outages and severe security vulnerabilities. Many engineers mistakenly treat an **OCI Security List** as the direct equivalent of an **AWS Security Group**, failing to realize that Security Lists apply at the **Subnet** boundary, exposing every instance in that subnet to the same rules. Furthermore, engineers frequently deploy stateless Network ACLs (NACLs) without understanding TCP ephemeral return port mechanics, causing total communication blackouts while connection tracking tables silently exhaust under high-concurrency traffic bursts.

## 2. Cloud Concept
Cloud firewall primitives differ across two architectural axes: **Statefulness** (stateful vs. stateless) and **Attachment Boundary** (network interface vs. subnet boundary).

### The Four Cloud Firewall Primitives
| Primitive | Cloud Provider | Attachment Target | Default Statefulness | Scope & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Security Group (SG)** | AWS | Elastic Network Interface (ENI) | Strictly Stateful | Microsegmentation; instance-level virtual firewall |
| **Network ACL (NACL)** | AWS | Subnet Boundary | Strictly Stateless | Subnet perimeter defense; coarse CIDR blocking |
| **Network Security Group (NSG)** | OCI | Virtual Network Interface (VNIC) | Configurable (Default: Stateful) | Microsegmentation; separates security policy from network topology |
| **Security List (SL)** | OCI | Subnet Boundary | Configurable (Default: Stateful) | Baseline subnet-wide firewall rules |

### Stateful Connection Tracking (`conntrack`) vs. Stateless Packet Evaluation
- **Stateful Firewalls (AWS SG, OCI NSG/SL Stateful)**:
  - When an inbound packet is permitted, the hypervisor creates an entry in an internal connection tracking table (`conntrack`):
    $$\text{Session Key} = \langle \text{Proto}, \text{SrcIP}, \text{SrcPort}, \text{DstIP}, \text{DstPort} \rangle$$
  - Outbound return traffic matching this session key is **automatically permitted**, bypassing outbound rule checks.
  - *Risk*: Hardware connection tracking tables have fixed limits. If an instance handles millions of concurrent micro-connections (e.g., DNS servers, NTP, or SYN floods), the tracking table exhausts, causing the hypervisor to drop packets (`conntrack: table full, dropping packet`).
- **Stateless Firewalls (AWS NACL, OCI Stateless Rules)**:
  - Zero connection tracking overhead. Every packet is evaluated independently against an ordered rule list.
  - If an inbound rule allows TCP port 443, the response packet **will be blocked on egress** unless an explicit outbound rule allows traffic to the client's **ephemeral port range** (`1024–65535`).
  - *Advantage*: Immune to connection tracking table exhaustion; perfect for mitigating volumetric Layer 4 DDoS attacks.

## 3. Mental Model
Think of cloud firewalls as security at a corporate headquarters:
- **Stateless Subnet Firewalls (AWS NACL / OCI Security List)** are the **Armed Guard at the Front Gate**. The guard checks everyone entering and leaving the parking lot against a strict written manifest. If you enter with an authorized visitor pass, but the guard has no explicit rule letting visitors drive out through the gate, you are trapped in the parking lot.
- **Stateful Interface Firewalls (AWS Security Group / OCI NSG)** are the **Biometric Badge Reader at your Office Door**. When you swipe in with your badge, the system unlocks the door and remembers you are inside. When you leave, the door unlocks automatically from the inside without requiring a second badge swipe.

## 4. Architecture Diagram
```text
PACKET INGRESS EVALUATION ORDER:

Incoming Internet Packet (Dst: 10.0.1.5:443)
        │
        ▼ 1. Subnet Boundary Check
┌─────────────────────────────────────────────────────────┐
│ AWS: Subnet NACL (Evaluated in numbered order 100, 200) │
│ OCI: Subnet Security List (Evaluated as a union set)    │
└───────────────────────────┬─────────────────────────────┘
                            │ Pass (If explicitly allowed)
                            ▼
        ▼ 2. Interface Boundary Check (Hypervisor / SmartNIC)
┌─────────────────────────────────────────────────────────┐
│ AWS: Security Group (Attached to ENI, Stateful)         │
│ OCI: Network Security Group (Attached to VNIC, Stateful)│
└───────────────────────────┬─────────────────────────────┘
                            │ Pass (Logged in conntrack table)
                            ▼
        [Compute Instance OS: Port 443 Listener]
                            │
                            ▼ Return Packet (Src: 10.0.1.5:443 -> Dst: Client Ephemeral Port)
┌─────────────────────────────────────────────────────────┐
│ AWS SG / OCI NSG: Automatically Allowed by conntrack!   │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│ AWS NACL: MUST have outbound rule allowing 1024-65535!  │
│ OCI SL:   If stateless, MUST have outbound allow rule!  │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼ Out to Internet
```

## 5. AWS Implementation
In AWS:
- **Security Groups (SGs)**:
  - Attached directly to the Elastic Network Interface (ENI). Up to 5 Security Groups per ENI by default `[Doc: Amazon VPC Security Group Limits, checked 2026-09-03]`.
  - **All-Deny Default**: SGs contain no allow rules by default; all inbound traffic is blocked until an explicit allow rule is added.
  - **No Deny Rules**: SGs do **not** support explicit `DENY` rules; everything not explicitly permitted is dropped.
  - **Security Group Referencing**: SGs can reference other Security Group IDs as source/destination (e.g., allow port 5432 ingress where source is `sg-app12345`). This allows dynamic microsegmentation across instances regardless of their private IP.
  - **Rule Quotas**: 60 inbound rules and 60 outbound rules per Security Group by default.
- **Network ACLs (NACLs)**:
  - Attached to the Subnet boundary. Every subnet must be associated with exactly 1 NACL.
  - **Rule Number Ordering**: Evaluated in strict numerical order (e.g., Rule 100 before Rule 200). First matching rule decides `ALLOW` or `DENY`.
  - **Supports Explicit DENY**: NACLs support explicit deny rules, making them ideal for instantly blacklisting malicious IP ranges during an active DDoS.
  - **Default Rule Quota**: 20 rules per NACL (adjustable up to 40 max) `[Doc: Amazon VPC Quotas, checked 2026-09-03]`.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **CRITICAL ARCHITECTURAL DISTINCTION**:
  - **Do NOT equate AWS Security Groups with OCI Security Lists**.
  - An AWS Security Group attaches to an **ENI**. An OCI Security List attaches to an **entire Subnet**.
  - The true architectural equivalent of an AWS Security Group in OCI is the **Network Security Group (NSG)** `[Doc: OCI Network Security Groups, checked 2026-09-03]`.
- **OCI Network Security Groups (NSGs)**:
  - Attached directly to the Virtual Network Interface (VNIC).
  - Can be attached to compute instances, load balancers, database systems, and mount targets. Up to 5 NSGs can be attached to a single VNIC.
  - **Decouples Security from Network Topology**: Allows instances with different security postures to share the same regional subnet without inheriting unwanted firewall rules.
  - Supports up to 120 security rules per NSG `[Doc: OCI Networking Service Limits, checked 2026-09-03]`.
  - Can reference another NSG ID as the traffic source or destination.
- **OCI Security Lists (SLs)**:
  - Attached to the Subnet level. All VNICs created in that subnet automatically inherit the Security List rules.
  - Evaluated as a **union set**: all rules in all associated Security Lists are evaluated together. There is no rule priority or numerical ordering.
  - Can contain up to 50 ingress and 50 egress rules by default.
- **Stateful vs. Stateless Flag in OCI**:
  - Unlike AWS where SGs are strictly stateful and NACLs are strictly stateless, in OCI, **both NSGs and Security Lists allow marking individual rules as stateful or stateless via a single boolean flag** (`is_stateless = true/false`) `[Doc: OCI Security Rules, checked 2026-09-03]`.
  - Marking high-throughput rules (e.g., Big Data replication, DNS lookups) as stateless bypasses connection tracking on the SmartNIC.

## 7. Configuration
Comparing microsegmentation configuration in Terraform across AWS and OCI:

### AWS Security Group Referencing (Terraform)
```hcl
# App Tier SG
resource "aws_security_group" "app" {
  name        = "app-tier-sg"
  vpc_id      = var.vpc_id
  description = "App server security group"

  ingress {
    description     = "HTTP from internal ALB only"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [var.alb_security_group_id]
  }
}

# DB Tier SG: Only permits traffic from App Tier SG
resource "aws_security_group" "db" {
  name        = "db-tier-sg"
  vpc_id      = var.vpc_id
  description = "Database security group"

  ingress {
    description     = "PostgreSQL from App Tier only"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id] # Sg-to-Sg referencing!
  }
}
```

### OCI Network Security Group (NSG) Referencing (Terraform)
```hcl
# App Tier NSG
resource "oci_core_network_security_group" "app_nsg" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  display_name   = "app-tier-nsg"
}

# DB Tier NSG
resource "oci_core_network_security_group" "db_nsg" {
  compartment_id = var.compartment_id
  vcn_id         = var.vcn_id
  display_name   = "db-tier-nsg"
}

# NSG-to-NSG Rule: DB allows ingress only from App NSG
resource "oci_core_network_security_group_security_rule" "allow_app_to_db" {
  network_security_group_id = oci_core_network_security_group.db_nsg.id
  direction                 = "INGRESS"
  protocol                  = "6" # TCP
  source_type               = "NETWORK_SECURITY_GROUP"
  source                    = oci_core_network_security_group.app_nsg.id # NSG Reference!

  tcp_options {
    destination_port_range {
      min = 5432
      max = 5432
    }
  }
  is_stateless = false # Stateful tracking
}
```

## 8. Data Flow
```text
TCP Handshake Validation in Stateful Cloud Firewall:

1. Client sends SYN ──> [Cloud Hypervisor / SmartNIC]
2. Firewall inspects Security Group / NSG ingress rules:
   * Rule matches port 443 ALLOW.
   * conntrack entry created: State = SYN_RECV.
   * Packet forwarded to instance OS socket.
3. Instance sends SYN-ACK ──> [Cloud Hypervisor / SmartNIC]
   * Hypervisor consults conntrack table.
   * Match found! conntrack entry updated: State = ESTABLISHED.
   * Packet transmitted out WITHOUT inspecting outbound SG rules.
4. Client sends ACK ──> Connection fully established.
```

## 9. Security
- **Defense-in-Depth Layering**:
  - Use **NACLs** (AWS) for coarse perimeter defense: block known malicious IP ranges (`DENY 198.51.100.0/24`) and restrict access to authorized management subnets.
  - Use **Security Groups / NSGs** for granular microsegmentation: restrict communication between microservice tiers using security group references.
- **The Principle of Least Privilege**: Never grant `0.0.0.0/0` ingress on management ports (SSH 22, RDP 3389, Database 5432/1521). Use AWS Systems Manager (SSM) Session Manager or OCI Bastion Service instead.

## 10. Reliability
- **Connection Tracking Table Exhaustion**:
  - On AWS EC2 Nitro instances, connection tracking capacity scales with instance size (typically 250,000 to 1,000,000 concurrent tracked connections) `[Doc: Nitro Conntrack Metrics, checked 2026-09-03]`.
  - When connection table reaches 100%, CloudWatch metric `conntrack_allowance_exceeded` spikes, and the hypervisor drops new connections.
  - *Reliability Mitigation*: For ultra-high-concurrency workloads (e.g., reverse proxies, DNS servers, Kafka brokers), use **Stateless Rules** in OCI, or use Network Load Balancers which bypass instance-level connection tracking.

## 11. Scaling
- **Rule Limit Constraints & Security Group Churn**:
  - AWS limits SGs to 60 rules. If an architecture creates 1 rule per client IP, it quickly breaches cloud quotas.
  - *Scaling Pattern*: Use **Security Group Referencing** or **Prefix Lists** (managed sets of CIDR blocks) in AWS. In OCI, use **Network Security Groups** rather than bloating subnet Security Lists.

## 12. Observability
- **Monitoring Dropped Packets**:
  - AWS: CloudWatch metric `conntrack_allowance_exceeded` on EC2 Nitro instances.
  - VPC Flow Logs: Filter by `action == "REJECT"`.
  - OCI: VCN Flow Logs with `action = "RE_REJECT"` indicates a packet dropped by an NSG or Security List rule.

## 13. Cost
- Security Groups, NACLs, NSGs, and Security Lists are **completely free** cloud primitives. They incur zero hourly charges and zero per-rule costs.
- Cost savings arise from eliminating commercial third-party virtual firewall appliances (e.g., Palo Alto, Fortinet) where native cloud microsegmentation via SGs/NSGs satisfies enterprise security requirements.

## 14. Failure Modes
- **The Stateless NACL / Security List Return Blackhole**: A developer creates a stateless rule allowing inbound HTTP (port 80) but fails to configure an outbound rule for the ephemeral port range (`1024–65535`). Traffic arrives, but all response packets are dropped.
- **The Security List Subnet Pollution Trap in OCI**: An engineer adds a Security List rule opening port 8080 to support a new web server. Because Security Lists apply at the **Subnet** boundary, all database instances and internal backend workers residing in that same subnet suddenly have port 8080 opened to the network. *Remediation: Migrate to NSGs immediately.*

## 15. Troubleshooting
When an application fails to connect across subnets:
1. **Determine Firewall Boundary**: Is the failure at the subnet layer (NACL / Security List) or the interface layer (SG / NSG)?
2. **Inspect VPC/VCN Flow Logs**:
   - `REJECT` on ingress: Target Security Group / NSG does not permit source IP or source SG.
   - `ACCEPT` on ingress, but client times out: Check subnet NACL outbound rules for ephemeral port allowance (`1024–65535`).
3. **Verify Security Group Reference Validity**: If using SG referencing, verify that the caller instance is actually associated with the referenced Security Group ID and both SGs belong to the same VPC (or peered VPC with SG referencing enabled).

## 16. Common Mistakes
- **Assuming AWS NACLs are Evaluated Like Security Groups**: Adding a rule to a NACL and expecting it to apply regardless of numerical order. Rule 50 `DENY 10.0.0.0/16` will permanently override Rule 100 `ALLOW 10.0.1.0/24`.
- **Using Security Lists Instead of NSGs for New OCI Projects**: Oracle officially recommends using **Network Security Groups (NSGs)** for all new application architectures because NSGs separate security policy from network topology `[Doc: OCI NSG Best Practices, checked 2026-09-03]`.

## 17. Trade-offs
| Dimension | Security Groups / NSGs | NACLs / Subnet Security Lists |
| :--- | :--- | :--- |
| **Granularity** | Highly Granular (Per-VNIC / Per-Container) | Coarse (Entire Subnet) |
| **State Tracking Overhead**| Consumes conntrack memory; subject to limits | Zero conntrack overhead; maximum line-rate performance |
| **Configuration Complexity** | High (Managed per workload tier) | Low (Centralized per subnet) |
| **Explicit Deny Support** | Unsupported in AWS SG; Supported in OCI NSG | Supported in AWS NACL; Evaluated via rule order |

## 18. Interview Questions
1. *Why should you never equate an AWS Security Group with an OCI Security List? What is the correct OCI equivalent, and why does Oracle recommend it?*
2. *Under what specific production workload conditions can an AWS Security Group drop packets even when an ingress rule explicitly allows the traffic?*
3. *A developer opens port 443 in an AWS NACL inbound rule, but clients cannot complete an HTTPS handshake. Explain the root cause and the exact configuration needed to fix it.*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "An AWS Security Group can drop packets even when an explicit allow rule matches under **Connection Tracking Table Exhaustion**:
>
> 1. **The Conntrack Mechanism**: AWS Security Groups are strictly stateful. To allow return traffic automatically without checking outbound rules, the underlying Nitro hypervisor maintains an internal connection tracking (`conntrack`) table in memory.
> 2. **How Exhaustion Occurs**: Every compute instance type has a hard architectural quota for tracked connections (e.g., 250,000 to 1,000,000 concurrent sessions). If an instance is subjected to a massive volume of short-lived connections—such as an unmitigated TCP SYN flood DDoS, high-frequency DNS query traffic, or an application opening thousands of new TCP connections per second without connection reuse—the `conntrack` table becomes completely saturated.
> 3. **The Symptom**: Once the table fills, the hypervisor's stateful engine cannot allocate a new tracking entry. It drops incoming SYN packets immediately, triggering the CloudWatch metric `conntrack_allowance_exceeded`.
> 4. **Architectural Prevention**:
>    - Deploy an AWS Network Load Balancer (NLB) in front of the instances. NLB handles connection distribution across AWS Hyperplane fabric, shielding the backend instances.
>    - In OCI, the equivalent solution is to mark high-volume rules as **Stateless** (`is_stateless = true`) in the Network Security Group, which completely bypasses the SmartNIC connection tracking engine."

## 20. Hands-on Exercise
**Objective**: Demonstrate stateless ephemeral return port blocking in AWS NACLs or OCI Stateless Security Rules.

### Verification Steps
1. Create a private subnet with an instance running an NGINX web server on port 80.
2. In the subnet NACL (or OCI Stateless Security List), create an inbound rule: `ALLOW TCP Port 80 from 0.0.0.0/0`.
3. Ensure the outbound rule allows only port 80.
4. Execute `curl -v --connect-timeout 3 http://<instance-ip>`:
   *Result*: Connection times out. The SYN packet reached the server, but the SYN-ACK response destined for the client's ephemeral port (e.g., port 51234) was dropped by the stateless outbound rule.
5. Add an outbound NACL rule: `ALLOW TCP Ports 1024-65535 to 0.0.0.0/0`.
6. Re-run `curl`: connection succeeds immediately, proving that stateless firewalls mandate explicit ephemeral return routing.
