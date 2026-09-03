# 02. Elasticity, Durability & Availability Economics

## 1. Problem
Engineering teams frequently conflate **scalability** with **elasticity**, and confuse **availability** with **durability**. A system can be infinitely scalable while being completely inelastic, leading to catastrophic cloud overspending during off-peak hours. Similarly, an architecture can boast 11 nines of data durability while experiencing frequent service outages that breach business SLAs. In senior and staff engineering interviews, failing to distinguish these concepts with mathematical rigor demonstrates a lack of distributed systems depth.

## 2. Cloud Concept
### Scalability vs. Elasticity
- **Scalability**: The structural capacity of a system to handle increased load by adding resources without redesigning core software architecture. Scalability is a property of the system's design (e.g., stateless services, database sharding).
- **Elasticity**: The operational ability of a system to automatically adapt resource provisioning in real time to match fluctuating workload demands, scaling **both out and in**. Elasticity is a property of the cloud platform automation.

### Availability vs. Durability
- **Availability**: The probability that a system is operational and accessible to process requests at any given point in time:
  $$A = \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}} = \frac{\text{Uptime}}{\text{Uptime} + \text{Downtime}}$$
  Where $\text{MTBF}$ is Mean Time Between Failures, and $\text{MTTR}$ is Mean Time To Recovery.
- **Durability**: The probability that stored data remains intact, uncorrupted, and retrievable over time, independent of whether the service API is currently available to serve it:
  $$D = 1 - \frac{\text{Data Lost}}{\text{Total Data Stored}}$$

A storage service can experience a 30-minute network outage (reducing availability to $99.93\%$ for that month) while losing zero bytes of data (maintaining $99.999999999\%$ durability).

## 3. Mental Model
### The Serial vs. Parallel Reliability Law
In distributed architectures, availability compounding behaves according to probability theory:

1. **Components in Series (Dependent Chains)**: Every component must function for the request to succeed:
   $$A_{\text{serial}} = A_1 \times A_2 \times \dots \times A_n$$
   *Consequence*: As you add microservices to a synchronous call chain, overall availability **strictly decreases**. Five services each operating at $99.9\%$ ($3\text{ nines}$) yield:
   $$0.999 \times 0.999 \times 0.999 \times 0.999 \times 0.999 \approx 99.50\% \quad (\text{Over } 3.6 \text{ hours of downtime/month!})$$

2. **Components in Parallel (Redundant Multi-AZ Paths)**: System succeeds if at least one redundant component functions:
   $$A_{\text{parallel}} = 1 - \prod_{i=1}^{n} (1 - A_i)$$
   *Consequence*: Two independent components with $99.0\%$ availability deployed in parallel yield:
   $$1 - (1 - 0.99)(1 - 0.99) = 1 - (0.01 \times 0.01) = 99.99\% \quad (4\text{ nines})$$

## 4. Architecture Diagram
```text
Synchronous Serial Chain (Availability Degrades Multiplicatively):
[Client] ──> [API Gateway (99.95%)] ──> [Auth Service (99.9%)] ──> [Order Service (99.9%)] ──> [Database (99.95%)]
Aggregate Availability: 0.9995 * 0.999 * 0.999 * 0.9995 = 99.70% (~2.16 hrs downtime/month)

Parallel Redundant Architecture (Availability Amplifies Exponentially):
                          ┌──> [AZ-1: App Replica (99.9%)] ──┐
[Load Balancer (99.99%)] ─┼──> [AZ-2: App Replica (99.9%)] ──┼──> [Multi-AZ DB (99.95%)]
                          └──> [AZ-3: App Replica (99.9%)] ──┘
Redundant App Tier Availability: 1 - (0.001)^3 = 99.9999999%
```

## 5. AWS Implementation
AWS formalizes these metrics through service level agreements (SLAs) and elasticity mechanisms:

### Availability SLAs
- **Amazon EC2**: Single-instance SLA is $99.5\%$; Multi-AZ EC2 SLA is $99.99\%$ `[Doc: Amazon EC2 Service Level Agreement, checked 2026-09-03]`.
- **Amazon S3 Standard**: Designed for $99.999999999\%$ (11 nines) of durability over a given year by redundantly storing objects across multiple physical facilities separated by kilometers `[Doc: Amazon S3 FAQs, checked 2026-09-03]`. SLA guarantees $99.9\%$ monthly availability.
- **Amazon Aurora**: Multi-AZ clusters provide an availability SLA of $99.99\%$, leveraging a distributed storage volume replicated 6 ways across 3 AZs `[Doc: Amazon Aurora SLA, checked 2026-09-03]`.

### Elasticity Automation
- **Target Tracking Scaling**: Automatically adjusts Auto Scaling Group (ASG) capacity to maintain a metric (e.g., maintain aggregate CPU utilization at 60%, or average ALB request count per target at 1000 RPS).
- **Predictive Scaling**: Uses machine learning to forecast daily and weekly traffic spikes, provisioning EC2 instances ahead of demand to overcome VM boot delays.

## 6. OCI Implementation
Oracle Cloud Infrastructure enforces identical distributed systems math while introducing unique elasticity primitives and enterprise durability guarantees:

### Availability SLAs
- **OCI Compute**:
  - Multi-AD / Multi-Fault Domain SLA guarantees $99.99\%$ availability `[Doc: OCI Service Level Agreement, checked 2026-09-03]`.
  - Single-instance VM SLA guarantees $99.9\%$ availability.
- **OCI Object Storage**: Designed for $99.999999999\%$ (11 nines) of annual durability. Standard Object Storage provides a $99.9\%$ availability commitment.
- **OCI Autonomous Database**: Multi-region Active Data Guard deployments guarantee a $99.995\%$ availability SLA (less than 2.16 minutes downtime per month) `[Doc: OCI PaaS Pillar Document, checked 2026-09-03]`.

### Elasticity Automation: Flexible Shapes & Auto-Scaling
- **Fine-Grained Compute Elasticity**: Unlike AWS EC2 where scaling up requires jumping between fixed rigid instance sizes (e.g., moving from `c6i.xlarge` with 4 vCPUs/8GB to `c6i.2xlarge` with 8 vCPUs/16GB), OCI **Flexible Shapes** (`VM.Standard.E5.Flex`) allow dynamic vertical elasticity:
  - Add single OCPUs or gigabytes of RAM via API or CLI without recreating the storage root volume.
- **OCI Instance Pool Autoscaling**: Evaluates metrics every minute, triggering scale-out or scale-in rules with configurable cooldown periods to prevent metric oscillation (thrashing).

## 7. Configuration
Comparing target-tracking elasticity configuration across clouds:

### AWS Auto Scaling Target Tracking (Terraform)
```hcl
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "target-tracking-cpu-60"
  autoscaling_group_name = aws_autoscaling_group.app_asg.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value     = 60.0
    disable_scale_in = false # Enables downward elasticity
  }
}
```

### OCI Auto-Scaling Configuration (Terraform)
```hcl
resource "oci_autoscaling_auto_scaling_configuration" "app_pool_autoscaling" {
  compartment_id = var.compartment_id
  display_name   = "app-pool-autoscaling"
  is_enabled     = true

  auto_scaling_resources {
    id   = oci_core_instance_pool.app_pool.id
    type = "instancePool"
  }

  policies {
    policy_type = "threshold"
    display_name = "cpu-threshold-policy"

    rules {
      action {
        type  = "CHANGE_COUNT_BY"
        value = 1
      }
      scale_in_rule {
        metric {
          metric_type = "CPU_UTILIZATION"
          threshold {
            operator = "LT"
            value    = 30
          }
        }
      }
      scale_out_rule {
        metric {
          metric_type = "CPU_UTILIZATION"
          threshold {
            operator = "GT"
            value    = 70
          }
        }
      }
    }
  }
}
```

## 8. Data Flow
```text
Load Surge Detected:
Client Traffic Spike ──> [Load Balancer] ──> Metrics Emitted (CloudWatch / OCI Monitor)
                                                     │
                                                     ▼
                                       [Threshold Evaluated > 70%]
                                                     │
                                                     ▼
                                    [Scale-Out Triggered (+2 Instances)]
                                                     │
                                                     ▼
                                     [Instances Boot & Health Check Pass]
                                                     │
                                                     ▼
                                     [Load Balancer Registers New Nodes]
```

## 9. Security
- **Elasticity as a DDoS Shield**: Elastic autoscaling absorbs volumetric HTTP floods, keeping services online while edge WAF rules propagate.
- **Instance Draining Security**: During scale-in events, compute nodes must gracefully drain active TLS sessions and revoke ephemeral IAM tokens before termination to prevent in-flight request truncation or credential leakage.

## 10. Reliability
- **The Downtime Equation**:
  $$\text{Downtime per Month} = (1 - A) \times 730 \text{ hours} \times 60 \text{ minutes}$$
  - $99.0\%$ (2 nines) = 7.3 hours/month downtime.
  - $99.9\%$ (3 nines) = 43.8 minutes/month downtime.
  - $99.99\%$ (4 nines) = 4.38 minutes/month downtime.
  - $99.999\%$ (5 nines) = 26 seconds/month downtime.
- **Designing for Durability**: Durability is achieved via erasure coding and multi-datacenter quorum replication. S3 and OCI Object Storage break objects into data and parity chunks distributed across geographically isolated facilities.

## 11. Scaling
- **Scale-Out Latency**: Spinning up a VM takes 60–180 seconds; spinning up a container pod takes 5–15 seconds; executing a serverless function takes 50–300ms. Architectures must buffer spikes (via message queues) when scaling latency exceeds client request timeouts.
- **Scale-In Thrashing Prevention**: Cooldown periods (e.g., 300 seconds in AWS ASG) prevent rapid oscillation where a fleet scales out and immediately scales in due to momentary metric fluctuations.

## 12. Observability
- **Leading vs. Lagging Indicators**:
  - *Lagging Metric*: CPU utilization. By the time CPU reaches 85%, request latency has already spiked, and queues have formed.
  - *Leading Metric*: SQS queue depth (`ApproximateNumberOfMessagesVisible`) or ALB target request count. Scaling on queue depth adds compute capacity before latency degrades.
- **SLO Error Budget Tracking**: Monitor error budget consumption rate ($1 - \text{SLO}$). If 20% of the monthly error budget burns in 1 hour, immediately freeze deployments.

## 13. Cost
- **The Financial Waste of Inelasticity**:
  A static 50-node cluster provisioned for peak load costs $50 \times \$0.20/\text{hr} \times 730\text{ hrs} = \$7,300/\text{month}$.
  An elastic cluster averaging 15 nodes off-peak and 50 nodes at peak costs $22 \times \$0.20/\text{hr} \times 730\text{ hrs} = \$3,212/\text{month}$ ($56\%$ cost reduction).
- **The Multi-Nines Cost Curve**:
  Moving from 3 nines ($99.9\%$) to 4 nines ($99.99\%$) roughly doubles infrastructure cost (requiring multi-AZ active-active, redundant load balancers, and standby databases). Moving to 5 nines ($99.999\%$) increases cost by 5x–10x due to multi-region infrastructure and automated cross-region consensus.

## 14. Failure Modes
- **The Asymmetric Scale-In Trap**: Scaling in too quickly during intermittent traffic lulls terminates healthy nodes, causing immediate overload on surviving nodes and triggering an infinite flapping loop.
- **The Illusion of 11 Nines**: An organization assumes their data is indestructible because it is on S3, but lacks S3 Versioning or MFA Delete. A malicious actor with compromised admin credentials executes `s3:DeleteObject` across the bucket, permanently wiping data. *Durability protects against hardware failure, not unauthorized deletion.*

## 15. Troubleshooting
When auto-scaling fails to respond during an outage:
1. **Check Scaling Limits**: Has the ASG or Instance Pool reached its `max_size` boundary?
2. **Check Cloud Quotas**: Has the account reached regional vCPU / OCPU limits?
3. **Inspect Launch Failures**: In AWS, check `Activity History` in the ASG console. In OCI, inspect `Work Requests` for errors like `OutOfCapacity` or `SubnetIPAllocationExhausted`.
4. **Check Target Group Health**: Are new instances failing health checks and getting terminated immediately after launch?

## 16. Common Mistakes
- **Scaling on Memory in AWS Without CloudWatch Agent**: AWS EC2 hypervisors cannot read guest OS memory utilization. An ASG configured to scale on memory without installing and configuring the CloudWatch agent will never scale.
- **Ignoring Database Connection Scaling**: Scaling the application tier from 10 to 100 instances causes database connections to jump from 500 to 5,000, instantly exhausting PostgreSQL/MySQL `max_connections` and causing a site-wide outage.

## 17. Trade-offs
| Strategy | Advantage | Trade-off / Cost |
| :--- | :--- | :--- |
| **Aggressive Elasticity (Fast Scale-In)** | Minimizes cloud bill immediately | Higher risk of thrashing and cold-start latency spikes |
| **Conservative Elasticity (High Headroom)** | High resilience against sudden traffic surges | Higher baseline cloud infrastructure expense |
| **Multi-Region High Availability** | Protects against total cloud region blackout | Extreme operational complexity, high data transfer fees, eventual consistency |

## 18. Interview Questions
1. *If a service has an availability SLA of 99.99%, how much downtime is permitted per calendar month? If this service synchronously calls a third-party payment provider with a 99.9% SLA, what is the maximum achievable availability of your system?*
2. *Why does Amazon S3 advertise 99.999999999% durability but only 99.9% availability? How is 11 nines of durability physically achieved?*
3. *How do OCI Flexible Shapes provide an architectural and financial advantage over AWS EC2 instance types when building an elastic backend?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "An availability SLA of $99.99\%$ (four nines) allows:
> $$(1 - 0.9999) \times 730 \text{ hours} \times 60 \text{ minutes} = 0.0001 \times 43,800 \text{ minutes} \approx 4.38 \text{ minutes of downtime per month}.$$
>
> If our service makes a synchronous serial call to a third-party payment provider with an SLA of $99.9\%$ ($43.8$ minutes downtime/month), probability theory dictates that components in series multiply:
> $$A_{\text{total}} = A_{\text{our\_service}} \times A_{\text{third\_party}} = 0.9999 \times 0.999 \approx 99.89\%$$
>
> Under a purely synchronous architecture, our system cannot exceed the availability of its weakest dependency. To meet our $99.99\%$ SLA, we must decouple the payment dependency architecturally:
> 1. Accept the customer payment intent asynchronously, writing it to a durable, multi-AZ message queue (AWS SQS or OCI Queue with $99.999\%$ availability).
> 2. Acknowledge HTTP 202 Accepted to the client immediately.
> 3. Process payment settlement asynchronously via workers equipped with exponential backoff and dead-letter queues. This decouples our user-facing availability from the third party's downtime."

## 20. Hands-on Exercise
**Objective**: Calculate the availability and blast radius of a real infrastructure topology.

### Exercise Steps
1. Map a 3-tier application: Route 53 ($100\%$ SLA) $\to$ ALB ($99.99\%$) $\to$ ECS/EC2 instances in 3 AZs ($99.99\%$) $\to$ Multi-AZ Aurora ($99.99\%$).
2. Calculate the theoretical maximum end-to-end availability:
   $$A = 1.0 \times 0.9999 \times 0.9999 \times 0.9999 = 0.9997 = 99.97\% \quad (\approx 13.14 \text{ minutes downtime/month}).$$
3. Inject a single synchronous call to an un-replicated microservice ($99.0\%$ availability) and observe the collapse of system availability down to $98.97\%$ (over 7.5 hours downtime/month).
