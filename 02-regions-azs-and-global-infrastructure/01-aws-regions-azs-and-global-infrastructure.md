# 01. AWS Regions, Availability Zones & Global Infrastructure

## 1. Problem
Distributed cloud systems are bound by the laws of physics: light in fiber travels at roughly $200,000 \text{ km/s}$ ($\approx 5 \mu\text{s per kilometer}$) in vacuum and slower in glass `[Inference]`. When engineers design multi-tier architectures without understanding physical topology, they inadvertently introduce crippling round-trip latency into synchronous database transactions or create single points of failure by co-locating critical replicas within the same physical data center. Furthermore, in multi-account enterprise environments, assuming that `us-east-1a` points to the same physical building across accounts leads to disastrous cluster misconfigurations.

## 2. Cloud Concept
AWS organizes its global footprint into a hierarchical topology of physical isolation and low-latency optical connectivity:

1. **AWS Region**: A separate geographic area (e.g., `us-east-1` in N. Virginia, `eu-central-1` in Frankfurt) containing multiple physically isolated and distinct data centers. Each region is completely autonomous and isolated from other regions to prevent cascading multi-region failures.
2. **Availability Zone (AZ)**: One or more discrete physical data centers with independent redundant power (diesel generators, separate municipal utility feeds), networking, and cooling. AZs within a region are connected via high-bandwidth, low-latency private metropolitan dark fiber loops, engineered to provide sub-2ms round-trip latency `[Doc: AWS Global Infrastructure, checked 2026-09-03]`.
3. **AZ Name vs. AZ ID**: To prevent all AWS customers from overloading `us-east-1a`, AWS dynamically maps logical AZ names (`us-east-1a`, `us-east-1b`) to different physical **AZ IDs** (`use1-az1`, `use1-az2`, `use1-az4`) randomly per AWS account.
4. **Edge Locations & Regional Edge Caches**: A global network of Points of Presence (PoPs) connected to AWS regions via the AWS private global network backbone, used by Amazon CloudFront and AWS Global Accelerator to terminate TLS close to end users.
5. **Local Zones & Wavelength**: Extensions of AWS regions placing compute and storage close to major urban centers (Local Zones) or 5G telecommunication data centers (Wavelength) for sub-10ms edge processing.

## 3. Mental Model
Think of AWS infrastructure as an airport transport network:
- **Regions** are major international airports (London Heathrow, Tokyo Haneda, New York JFK). Flying between them takes hours (cross-region network latency: 70–250ms). If London closes due to fog, Tokyo operates unaffected.
- **Availability Zones** are separate physical passenger terminals within the same airport. They share the same regional airspace but have independent electrical generators, HVAC systems, and baggage handling systems. Moving between terminals takes 2 minutes via automated monorail (cross-AZ optical fiber: 1–2ms).
- **Edge Locations** are regional ticket kiosks located in city downtowns. They can validate your ticket (cache static content, terminate TLS), but you cannot board a plane there.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                         AWS REGION: us-east-1                          │
│                                                                        │
│  ┌────────────────────────┐              ┌──────────────────────────┐  │
│  │ AVAILABILITY ZONE A    │              │ AVAILABILITY ZONE B      │  │
│  │ Logical: us-east-1a    │              │ Logical: us-east-1b      │  │
│  │ Physical: use1-az2     │              │ Physical: use1-az4       │  │
│  │ ┌────────────────────┐ │  Private     │ ┌──────────────────────┐ │  │
│  │ │ DC 1     DC 2      │ │  Metro Dark  │ │ DC 3       DC 4      │ │  │
│  │ │ (Diesel Gen / UPS) │ │◄────────────►│ │ (Diesel Gen / UPS)   │ │  │
│  │ └────────────────────┘ │  Fiber Loop  │ └──────────────────────┘ │  │
│  └────────────────────────┘  (< 2ms RTT) └──────────────────────────┘  │
│                                                                        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ AWS Private Global Backbone
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │ EDGE LOCATIONS (CloudFront / Global Accelerator PoPs)   │
       │ New York, London, Paris, Tokyo, Singapore, Sydney       │
       └─────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS, every VPC spans all Availability Zones in the region, but individual subnets are strictly **Zonal**:
- **Zonal Subnet Binding**: A subnet exists within exactly one AZ. When provisioning an Amazon EC2 instance or RDS replica, you bind it to a specific subnet, which physically locks the instance to that AZ's underlying data centers.
- **Cross-AZ Data Replication**: Managed services like Amazon Aurora and Amazon S3 manage cross-AZ replication transparently:
  - Aurora replicates every write 6 ways across 3 AZs (2 copies per AZ) using an optimized write-ahead log (WAL) engine over direct optical links `[Doc: Aurora Storage Architecture, checked 2026-09-03]`.
  - S3 stripes erasure-coded chunks across at least 3 AZs to guarantee 11 nines of durability.
- **Inter-Account AZ ID Alignment**: When engineering low-latency Cassandra clusters or Kafka clusters across multiple AWS accounts in an AWS Organization, engineers must query the `aws ec2 describe-availability-zones` API to map logical names to physical `ZoneId` strings (e.g., matching `use1-az1` across both accounts).

## 6. OCI Implementation
Oracle Cloud Infrastructure designs regional and physical boundaries with significant architectural contrasts:
- **OCI Regions & Availability Domains (ADs)**: OCI regions are categorized as either **Multi-AD regions** (such as Ashburn and Phoenix, containing 3 physically distinct Availability Domains) or **Single-AD regions** (such as Frankfurt, London, or Tokyo) `[Doc: OCI Regions and Availability Domains, checked 2026-09-03]`.
- **Fault Domains (FDs)**: To solve the single-AZ disaster vulnerability that plagues single-zone cloud regions, OCI builds every Availability Domain with **3 Fault Domains**:
  - Each Fault Domain represents a distinct physical hardware rack, independent power distribution units (PDUs), and redundant top-of-rack switches within the data center.
  - In OCI single-AD regions, compute instances distributed across `FAULT-DOMAIN-1`, `FAULT-DOMAIN-2`, and `FAULT-DOMAIN-3` achieve hardware anti-affinity and protect against maintenance reboots.
- **Regional Subnets by Default**: While AWS subnets are strictly zonal, OCI subnets are **Regional by default**. An OCI subnet CIDR block spans all Availability Domains in the region, dramatically simplifying routing tables and load balancer target attachment.

## 7. Configuration
Querying physical AZ ID mappings to guarantee cross-account co-location:

### AWS CLI: Discovering Physical Zone IDs
```bash
# Query how logical AZ names map to physical Zone IDs in this account
aws ec2 describe-availability-zones \
    --region us-east-1 \
    --query "AvailabilityZones[*].[ZoneName, ZoneId, State]" \
    --output table
```

**Expected Output**:
```text
--------------------------------------------
|         DescribeAvailabilityZones        |
+-------------+------------+---------------+
|  us-east-1a |  use1-az1  |  available    |
|  us-east-1b |  use1-az2  |  available    |
|  us-east-1c |  use1-az4  |  available    |
|  us-east-1d |  use1-az6  |  available    |
+-------------+------------+---------------+
```

### Terraform: Pinning Multi-AZ Deployment by Zone ID
```hcl
# Fetch region availability zones with ZoneId filtering
data "aws_availability_zones" "available" {
  state = "available"
}

# Provision subnets mapped explicitly to physical Zone IDs to ensure cross-account alignment
resource "aws_subnet" "app_az1" {
  vpc_id            = var.vpc_id
  cidr_block        = "10.0.1.0/24"
  availability_zone_id = "use1-az1" # Physical ID, not logical name
  tags = { Name = "app-private-use1-az1" }
}

resource "aws_subnet" "app_az2" {
  vpc_id            = var.vpc_id
  cidr_block        = "10.0.2.0/24"
  availability_zone_id = "use1-az2" # Physical ID, not logical name
  tags = { Name = "app-private-use1-az2" }
}
```

## 8. Data Flow
```text
Client (London) ──> [CloudFront Edge PoP: LHR50] (Terminates TLS, Caches Static Assets)
                          │
                          ▼ (Traverses AWS Private Global Fiber Backbone ~75ms)
                   [Application Load Balancer: us-east-1 Regional Listener]
                          │
            ┌─────────────┴─────────────┐
            ▼ (Sub-2ms Metro Fiber)     ▼
     [Subnet in use1-az1]        [Subnet in use1-az2]
     (App Server Instance)       (App Server Instance)
            │                           │
            └─────────────┬─────────────┘
                          ▼ (Synchronous Quorum Replication)
     [Aurora Primary: use1-az1] ──► [Aurora Storage Node: use1-az2 / use1-az4]
```

## 9. Security
- **Data Residency & Sovereignty**: Data stored within an AWS or OCI region never leaves that region unless the customer explicitly configures cross-region replication. This boundary is critical for regulatory compliance (e.g., GDPR in `eu-central-1`, HIPAA in US regions).
- **Physical Data Center Security**: Cloud data centers enforce strict compartmentalized physical security: biometric scanners, electromagnetic interference shielding, two-factor authenticated mantraps, and automated disk degaussing/shredding facilities.

## 10. Reliability
- **The Blast Radius Axiom**: An architecture must contain failure within the smallest possible blast radius:
  $$\text{Rack Failure (Fault Domain)} < \text{Building Failure (AZ/AD)} < \text{Regional Disaster (Region)}$$
- **Regional Isolation**: AWS regions share no shared memory, no shared databases, and no shared control planes. Even core services like IAM and Route 53, while presenting global interfaces, execute against geographically distributed, replicated state engines.

## 11. Scaling
- **Zonal Capacity Imbalance**: In major events, a specific physical AZ in an AWS region may experience temporary compute capacity shortages (e.g., `InsufficientInstanceCapacity` error when launching `m6i.4xlarge` instances).
- **Mitigation**: Configure Auto Scaling Groups to use the `balanced-best-effort` or `allocation-strategy` set to `capacity-optimized`, allowing the ASG to automatically provision across surviving AZs with available hardware capacity.

## 12. Observability
- **AWS Health Dashboard / OCI Status**: Monitor for regional service impairments and localized AZ power disruptions.
- **Cross-AZ Network Telemetry**: Collect VPC Flow Logs / VCN Flow Logs. Analyze traffic volume across AZs using CloudWatch Logs Insights to detect asymmetrical routing or cross-AZ traffic loops.

## 13. Cost
- **Cross-AZ Network Charges**:
  - Intra-AZ traffic (same AZ, private IP): **Free** `[Doc: AWS EC2 Pricing, checked 2026-09-03]`.
  - Cross-AZ traffic (between two AZs in the same region): **\$0.01/GB in each direction** (\$0.02/GB round-trip) in AWS `[Approximation — verify before relying on this]`.
  - In OCI, data transfer between Availability Domains in the same region is **completely free** `[Doc: OCI Cloud Economics, checked 2026-09-03]`.
- **Cross-Region Egress Charges**:
  - Transferring data across cloud regions incurs internet egress rates: typically \$0.02/GB in AWS and significantly lower in OCI `[Doc: OCI Networking Pricing, checked 2026-09-03]`.

## 14. Failure Modes
- **The Cross-Account AZ Mismatch Trap**: Team A deploys Kafka brokers in Account 1 under `us-east-1a`. Team B deploys consumer microservices in Account 2 under `us-east-1a`. Because logical AZs are randomized, Account 1's `us-east-1a` is physical `use1-az1`, while Account 2's `us-east-1a` is physical `use1-az4`. The teams unknowingly incur hundreds of thousands of dollars in cross-AZ network fees and add 1.5ms latency to every message.
- **The Transit Fiber Sever (Split-Brain Risk)**: A backhoe cuts the fiber line connecting AZ-1 and AZ-2. If a database cluster does not use a 3-way quorum voting model (e.g., 2 AZs with 50/50 vote split), both sides may believe the other is dead and attempt to promote themselves as master, causing split-brain data divergence.

## 15. Troubleshooting
When encountering unexpected cross-AZ latency or connectivity loss:
1. **Verify AZ ID Mapping**: Check whether client and server instances reside in the same physical `ZoneId`.
2. **Ping / Traceroute Diagnostic**: Run `mtr -c 100 --report <target-ip>` to measure packet loss and jitter across the internal VPC network.
3. **Inspect CloudWatch / OCI Metrics**: Check `NetworkPacketsOut` and `NetworkPacketsIn`. Verify whether single-AZ network interfaces are dropping packets due to exceeding PPS limits.

## 16. Common Mistakes
- **Deploying 2-AZ Architectures for Quorum Systems**: Deploying Raft, etcd, or Consul across only 2 Availability Zones. If the AZ holding 2 of the 3 nodes fails, the cluster loses quorum and freezes all writes. Quorum clusters must span **at least 3 AZs** (or 3 Fault Domains in OCI).
- **Treating `us-east-1` as the Default Gold Standard**: While `us-east-1` (N. Virginia) historically receives new AWS features first, it also has the largest legacy surface area and has historically suffered more global blast-radius incidents than newer regions like `us-east-2` (Ohio) or `eu-central-1` (Frankfurt).

## 17. Trade-offs
| Architecture Scope | Latency Characteristic | Availability SLA | Cost Profile |
| :--- | :--- | :--- | :--- |
| **Single-AZ / Single-AD** | Lowest (< 0.5ms) | Low (Vulnerable to facility power/flood outage) | Lowest (Zero cross-AZ transfer fees) |
| **Multi-AZ / Multi-AD** | Balanced (1–2ms RTT) | High (99.99% multi-AZ SLA) | Moderate (Cross-AZ network transfer fees apply in AWS) |
| **Multi-Region Active-Active**| High (70–200ms RTT) | Extreme (Survives catastrophic regional disaster) | Very High (Duplicate compute/storage + continuous egress replication fees) |

## 18. Interview Questions
1. *Why does AWS randomize the mapping between logical AZ names (`us-east-1a`) and physical AZ IDs (`use1-az1`)? How do you architect a multi-account low-latency trading engine around this?*
2. *If you deploy a 3-node etcd or ZooKeeper cluster across 2 Availability Zones (AZ-1 has 2 nodes, AZ-2 has 1 node), what happens when AZ-1 suffers a power outage? How would you solve this on AWS vs. OCI?*
3. *How do OCI Fault Domains differ from AWS Availability Zones in terms of physical blast radius and failure isolation?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "A 3-node etcd or ZooKeeper cluster relies on majority consensus (Raft or ZAB). The minimum nodes required for quorum is:
> $$Q = \left\lfloor \frac{N}{2} \right\rfloor + 1 = \left\lfloor \frac{3}{2} \right\rfloor + 1 = 2 \text{ nodes}$$
>
> If we deploy 2 nodes in AZ-1 and 1 node in AZ-2, and AZ-1 suffers a power outage, only 1 node survives in AZ-2. Since $1 < 2$, the surviving node cannot form a quorum. The cluster immediately freezes all writes and transitions to read-only or becomes completely unavailable. Deploying a quorum cluster across only 2 AZs provides **zero additional availability** over deploying it in a single AZ during a catastrophic zone failure.
>
> **AWS Solution**:
> In AWS, quorum clusters must span **at least 3 distinct Availability Zones** (1 node in AZ-1, 1 node in AZ-2, 1 node in AZ-3). If any single AZ fails, 2 nodes survive ($2 \ge 2$), preserving write quorum.
>
> **OCI Solution**:
> In OCI, if we are deployed in a 3-AD region, we place 1 node in each AD. However, in an OCI single-AD region, we achieve hardware fault tolerance by placing 1 node in `FAULT-DOMAIN-1`, 1 node in `FAULT-DOMAIN-2`, and 1 node in `FAULT-DOMAIN-3`. If an entire server rack, power distribution unit, or top-of-rack switch fails in Fault Domain 1, the 2 nodes in Fault Domains 2 and 3 maintain quorum and keep the system operational."

## 20. Hands-on Exercise
**Objective**: Query and map physical AZ IDs in AWS and identify Fault Domain allocation in OCI.

### Verification Steps
1. In AWS CLI, run `aws ec2 describe-availability-zones --region us-east-1` and record the physical `ZoneId` for each logical name.
2. In OCI CLI, run `oci iam fault-domain list --availability-domain "<AD-Name>"` and verify that 3 distinct Fault Domains are returned.
3. Verify in Terraform that subnet definitions reference `availability_zone_id` (AWS) or specify `fault_domain` in compute instance declarations (OCI).
