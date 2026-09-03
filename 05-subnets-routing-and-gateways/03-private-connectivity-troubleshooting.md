# 03. Private Connectivity Forensic Troubleshooting

## 1. Problem
During production outages involving private cloud networking, engineers frequently lose hours making random changes to security groups, restarting services, or blaming database performance. Network failure modes in virtualized clouds are strictly deterministic: packets are either accepted, routed, translated, or dropped based on routing tables, connection tracking tables, stateless access lists, or gateway states. Senior and Staff engineers follow an evidence-based forensic methodology to isolate and resolve private connectivity failures in minutes.

## 2. Cloud Concept: The 5 Network Checkpoints
Whenever a packet fails to reach its destination in a VPC or VCN, the failure must exist at one of five sequential checkpoints:
```text
[1. OS Stack / Socket] ──> [2. Interface Firewall] ──> [3. Subnet Firewall] ──> [4. Route Table] ──> [5. Gateway / Target]
 (ARP, Routing, MTU)     (AWS SG / OCI NSG)          (AWS NACL / OCI SL)      (LPM Matching)        (NAT, TGW, DRG)
```

## 3. The Classic Production Scenario: "App Reaches DB, But Times Out Reaching External API"
An EC2 or OCI Compute instance running a backend application in a private subnet can execute queries against PostgreSQL on `10.0.2.50:5432`, but all outbound HTTPS calls to `https://api.stripe.com` hang and terminate with `Connection timed out`.

### Forensic Triage Protocol:
1. **Checkpoint 1: Validate Local DNS Resolution**
   - Execute `dig api.stripe.com` or `nslookup api.stripe.com`.
   - *If DNS fails*: Check `/etc/resolv.conf`. Verify that the VPC DNS resolver (`10.0.0.2` in AWS or `10.0.0.1` in OCI) is reachable. If DNS resolution succeeds and returns a public IP (`54.187.159.182`), eliminate DNS as the root cause.
2. **Checkpoint 2: Test Transport Layer Connectivity**
   - Execute `curl -v --connect-timeout 5 https://api.stripe.com`.
   - *Symptom*: Output hangs at `* Connecting to api.stripe.com (54.187.159.182:443)...`. This proves Layer 4 TCP SYN packets are not receiving SYN-ACK responses.
3. **Checkpoint 3: Inspect Subnet Route Table**
   - Query the route table attached to the private application subnet.
   - *Check 3A*: Is `0.0.0.0/0` present? If missing, packets are dropped immediately by the virtual router.
   - *Check 3B*: Does `0.0.0.0/0` point to a **NAT Gateway**?
     - *Common Mistake*: Pointing `0.0.0.0/0` directly to an Internet Gateway (`igw-xxxx`). An Internet Gateway cannot route private RFC 1918 IPs without a public IP assigned to the instance ENI!
4. **Checkpoint 4: Verify the NAT Gateway's Host Subnet**
   - Inspect the subnet where the NAT Gateway is deployed:
     - Is the NAT Gateway in a **Public Subnet**?
     - Does the NAT Gateway's subnet route table contain a default route `0.0.0.0/0` targeting an **Internet Gateway**?
     - *Common Mistake*: A developer creates a NAT Gateway inside the private application subnet itself. The NAT Gateway attempts to forward traffic out, but its own route table loops back into the private subnet, creating a blackhole.
5. **Checkpoint 5: Inspect VPC / VCN Flow Logs**
   - Query Flow Logs for the calling instance ENI / VNIC:
     - Filter by `srcAddr = 10.0.1.5` and `dstAddr = 54.187.159.182`.
     - *If `action == REJECT`*: An egress Security Group rule or subnet NACL outbound rule is explicitly blocking outbound port 443.
     - *If `action == ACCEPT` on egress, but zero inbound packets return*: The packet reached the NAT Gateway, but the subnet NACL inbound rule is blocking the **ephemeral return port range** (`1024–65535`).

## 4. AWS Implementation Traps
- **Zonal NAT Gateway Outages**: In AWS, if you deploy a single NAT Gateway in AZ-1, and AZ-1 suffers an infrastructure failure, subnets in AZ-2 and AZ-3 that route through AZ-1 lose all outbound internet access. *Fix: Deploy 1 NAT Gateway per AZ*.
- **Transit Gateway Blackhole Routes**: When an attached VPC is deleted, routes in spoke VPC route tables targeting that deleted VPC attachment become `blackhole`. Clean up dangling routes via automation.

## 5. OCI Implementation Traps
- **Missing Service Gateway Route**: In OCI, an instance in a private subnet can reach the internet via a NAT Gateway, but calls to OCI Object Storage fail or incur unnecessary latency because the route table lacks a route rule for `SERVICE_CIDR_BLOCK` targeting the OCI Service Gateway.
- **Security List Stateless Flag Asymmetry**: Marking an ingress rule as stateless in an OCI Security List without adding the corresponding stateless egress rule for ephemeral return ports blocks all response traffic.

## 6. Architectural Trade-offs
| Diagnostic Tool | Visibility | Performance Overhead | Best Use Case |
| :--- | :--- | :--- | :--- |
| **VPC / VCN Flow Logs** | Layer 3/4 metadata (`ACCEPT`/`REJECT`) | Zero (Captured at hypervisor) | Root-cause firewall drops and traffic volume analysis |
| **Packet Capture (`tcpdump`)** | Full Layer 2–7 payload & headers | CPU & disk I/O overhead on host | Inspecting TLS handshake failures and corrupt headers |
| **Reachability Analyzer** | Static path analysis of cloud routing | Zero runtime impact | Validating security group and route table paths before launch |

## 7. Senior Interview Question & Defense
**Question**: *An EC2 instance in a private subnet can connect to other instances in the same VPC, but cannot download packages from external yum repositories or reach AWS S3. Walk me through how you isolate the exact point of failure.*

**Staff-Level Defense**:
> "I isolate the failure using a systematic outside-in diagnostic process:
>
> 1. **Separate Internet Egress from AWS API Egress**: S3 can be reached via two completely different paths: a public NAT Gateway or a private S3 VPC Gateway Endpoint.
> 2. **Check S3 Gateway Endpoint First**: I inspect the VPC route table. If the S3 Prefix List (`pl-xxxx`) is missing, S3 traffic is forced through the default route (`0.0.0.0/0`). Adding an S3 Gateway Endpoint immediately restores S3 connectivity and bypasses internet routing entirely.
> 3. **Triage yum Repository Egress via NAT Gateway**:
>    - I verify DNS: `dig mirrors.fedoraproject.org`. If resolution succeeds, DNS is healthy.
>    - I inspect the private route table: verify `0.0.0.0/0` targets an active NAT Gateway (`nat-xxxx`).
>    - I inspect the NAT Gateway status: verify the NAT Gateway state is `available` and resides in a public subnet whose route table points `0.0.0.0/0` to an Internet Gateway (`igw-xxxx`).
>    - I check Security Groups: verify egress allows TCP port 80/443 to `0.0.0.0/0`.
>    - I check NACLs: verify inbound rules permit ephemeral ports (`1024–65535`) to allow response packets back into the subnet.
> 4. By verifying these 5 checkpoints, the exact root cause—whether a missing route, an incorrectly placed NAT Gateway, or a stateless NACL block—is identified with zero guesswork."
