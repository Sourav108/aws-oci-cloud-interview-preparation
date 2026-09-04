# 01. Autoscaling Mechanisms & Scaling Policies

## 1. Problem
Static infrastructure provisioning forces engineering organizations into a lose-lose trade-off: over-provisioning compute capacity to survive rare peak traffic spikes wastes millions of dollars annually on idle servers during off-peak hours, while under-provisioning compute guarantees catastrophic platform collapse when traffic surges during flash sales, marketing campaigns, or viral news cycles. Furthermore, poorly calibrated autoscaling configurations introduce a fatal operational failure called **Autoscaling Thrashing (Flapping)**: instances scale out rapidly in response to a momentary CPU spike, and then immediately terminate 3 minutes later when CPU drops, continuously cycling instances, dropping active user connections, and burning cloud budgets.

## 2. Cloud Concept
### Reactive vs. Predictive Scaling Models
- **Reactive Autoscaling**:
  - Responds to real-time telemetry metrics (CPU utilization, memory consumption, active network connections, or queue backlog).
  - *The Latency Problem*: Scaling out a virtual machine is not instantaneous. Between CloudWatch/OCI metric aggregation (1–3 minutes), alarm evaluation (1–3 minutes), instance boot time, and application warm-up/health checks (2–5 minutes), reactive autoscaling requires **5 to 10 minutes** to deliver active compute capacity.
- **Predictive Autoscaling (Machine Learning Forensics)**:
  - Analyzes historical traffic patterns (diurnal cycles, weekly peaks) using machine learning models to forecast future capacity requirements 48 hours in advance.
  - Provisions compute instances **15 to 30 minutes before** the anticipated traffic surge begins, ensuring new instances are fully warmed and healthy before traffic arrives.

### The Autoscaling Policy Spectrum
1. **Target Tracking Scaling (The Industry Standard)**:
   - Operates like a household thermostat: you set a target metric value (e.g., maintain average EC2 CPU utilization at **60%**, or maintain `ALBRequestCountPerTarget` at **1,000 requests/instance**).
   - The cloud scaling controller continuously calculates the mathematical ratio:
     $$\text{Capacity Ratio} = \frac{\text{Current Metric Value}}{\text{Target Metric Value}}$$
   - Automatically increases or decreases instance count proportionally to hold the metric steady.
2. **Step Scaling**:
   - Executes non-linear scaling adjustments based on the severity of the metric breach:
     - *Step 1*: If CPU is between 60% and 75% $\longrightarrow$ Add 2 instances.
     - *Step 2*: If CPU is between 75% and 90% $\longrightarrow$ Add 5 instances.
     - *Step 3*: If CPU is $> 90\%$ $\longrightarrow$ Add 15 instances immediately!
   - Ideal for aggressive, fast-moving traffic surges where Target Tracking cannot add capacity fast enough.
3. **Scheduled Scaling**:
   - Executes fixed capacity adjustments based on predetermined cron schedules (e.g., scale up to 50 instances at 8:45 AM every Monday before the financial markets open).

### Mitigating Flapping via Cooldowns & Warm-up Times
- **Default Cooldown Period**: A mandatory timer (typically 300 seconds) after a scaling activity completes. During cooldown, the autoscaling engine **ignores all additional scale-in/scale-out alarms**, allowing newly launched instances time to absorb traffic before evaluating metrics again.
- **Instance Warm-up Time**: Instructs the autoscaling engine to exclude newly launched instances from aggregate metric calculations until their application runtime has finished warming up (e.g., 180 seconds).

## 3. Mental Model
Think of cloud autoscaling as an airport security checkpoint:
- **Reactive Scaling** is the airport manager noticing a line of 500 passengers wrapping around the building, picking up the telephone, calling off-duty TSA agents, and waiting 30 minutes for them to drive to the airport and open new lanes.
- **Target Tracking Scaling** is an automated turnstile counter: as soon as the line exceeds 15 people per open lane, an electronic bell rings to open another lane.
- **Predictive Scaling** is the airport analyzing flight departure schedules: they know that 5 Boeing 777s depart at 9:00 AM every Friday, so they open 12 security lanes at 7:30 AM before passengers even arrive at the airport.
- **Cooldown Period** is forcing the manager to wait 10 minutes after opening a lane before deciding whether to open another one, preventing the manager from calling 100 agents in a panic.

## 4. Architecture Diagram
```text
AUTOSCALING TARGET TRACKING & HYSTERESIS LOOP:

[Application Load Balancer / OCI Flexible LB]
       │
       ├──► Aggregates Metric: ALBRequestCountPerTarget
       │    Target: 1,000 requests per instance
       │
       ▼
┌────────────────────────────────────────────────────────────────────────┐
│ CLOUD AUTOSCALING CONTROLLER (AWS ASG / OCI Instance Pool)            │
│                                                                        │
│   METRIC MATH:                                                         │
│   Current Load = 5,000 requests/sec across 2 instances (2,500/inst)    │
│   Target Metric = 1,000 requests/instance                              │
│   New Desired Capacity = 5,000 / 1,000 = 5 Instances!                  │
│                                                                        │
│   SCALE-OUT ACTION: Launches 3 New Compute Instances                   │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ INSTANCE WARM-UP WINDOW (180 Seconds):                         │   │
│   │ * Instance boots OS, pulls container, warms JVM connection pool│   │
│   │ * EXCLUDED from aggregate metrics to prevent metric skew!      │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│   COOLDOWN TIMER (300 Seconds):                                        │
│   * Freezes scaling controller; blocks premature scale-in!             │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Auto Scaling:
- **EC2 Auto Scaling Groups (ASG)**:
  - Orchestrates instance lifecycle using **EC2 Launch Templates**:
    - Configures instance shape, AMI ID, security groups, EBS volumes, and `user_data` scripts.
    - **Mixed Instances Policy**: Combines multiple instance types (e.g., `c6i.xlarge`, `c5.xlarge`, `c6a.xlarge`) and mixes **On-Demand and Spot Instances** in a single ASG (e.g., 20% On-Demand baseline + 80% Spot) to slash compute costs by up to 70%!
- **Predictive Scaling Engine**:
  - Employs Amazon machine learning models that analyze at least 24 hours (and up to 14 days) of historical CloudWatch traffic data.
  - Automatically schedules predictive capacity scaling ahead of recurring daily/weekly spikes `[Doc: AWS Predictive Scaling Architecture, checked 2026-09-04]`.
- **Termination Policies**:
  - Determines which specific instance is terminated during scale-in:
    - Default: Balances instances evenly across Availability Zones, then terminates the instance with the oldest Launch Template or Launch Configuration.
    - Custom: Protects instances with active user sessions using **Scale-In Protection**.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Autoscaling:
- **OCI Instance Pools & Autoscaling Configurations**:
  - In OCI, an **Instance Pool** manages a group of identical compute instances based on an **Instance Configuration** (the OCI equivalent of an AWS Launch Template) `[Doc: OCI Autoscaling Overview, checked 2026-09-04]`.
  - An **Autoscaling Configuration** attaches to the Instance Pool and defines scaling policies.
- **Metric-Based Autoscaling**:
  - OCI supports native threshold scaling on CPU utilization or Memory utilization:
    - *Scale-out rule*: If average CPU $> 75\%$, scale out by 2 instances (or 20%).
    - *Scale-in rule*: If average CPU $< 25\%$, scale in by 1 instance (or 10%).
  - Configures **Cooldown Periods** in seconds (e.g., 300 seconds) independently for scale-out and scale-in.
- **Schedule-Based Autoscaling**:
  - Allows defining recurring cron expressions (e.g., `0 8 * * 1-5` for 8:00 AM Monday–Friday) to adjust pool size up, and evening rules to scale pool size down to save money.
- **Fault Domain Native Placement**:
  - OCI Instance Pools automatically distribute newly launched compute instances evenly across **OCI Fault Domains** within an Availability Domain, ensuring that autoscaled fleets remain resilient against physical rack power and network switch failures.

## 7. Configuration
Comparing autoscaling policies in Terraform across AWS and OCI:

### AWS EC2 Auto Scaling Group with Target Tracking (Terraform)
```hcl
# AWS Launch Template
resource "aws_launch_template" "app_template" {
  name_prefix   = "app-launch-template-"
  image_id      = var.ami_id
  instance_type = "c6i.large"

  vpc_security_group_ids = [var.app_security_group_id]
  user_data              = filebase64("${path.module}/bootstrap.sh")

  lifecycle {
    create_before_destroy = true
  }
}

# AWS Auto Scaling Group across 3 AZs
resource "aws_autoscaling_group" "app_asg" {
  name_prefix         = "prod-app-asg-"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [var.alb_target_group_arn]

  min_size         = 2
  max_size         = 20
  desired_capacity = 4

  launch_template {
    id      = aws_launch_template.app_template.id
    version = "$Latest"
  }

  default_cooldown          = 300
  health_check_type         = "ELB" # Uses ALB health checks!
  health_check_grace_period = 300
}

# Target Tracking Policy: Maintain ALB Requests at 1,000 per instance
resource "aws_autoscaling_policy" "target_tracking" {
  name                   = "alb-target-tracking-policy"
  autoscaling_group_name = aws_autoscaling_group.app_asg.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${var.alb_arn_suffix}/${var.target_group_arn_suffix}"
    }
    target_value = 1000.0
  }
}
```

### OCI Instance Pool & Autoscaling Configuration (Terraform)
```hcl
# 1. OCI Instance Configuration (Template)
resource "oci_core_instance_configuration" "app_config" {
  compartment_id = var.compartment_id
  display_name   = "app-instance-config"

  instance_details {
    instance_type = "compute"
    launch_details {
      compartment_id = var.compartment_id
      shape          = "VM.Standard.E5.Flex"

      shape_config {
        ocpus         = 2
        memory_in_gbs = 16
      }

      source_details {
        source_type = "image"
        image_id    = var.oci_image_id
      }
    }
  }
}

# 2. OCI Instance Pool
resource "oci_core_instance_pool" "app_pool" {
  compartment_id           = var.compartment_id
  instance_configuration_id = oci_core_instance_configuration.app_config.id
  size                     = 2
  display_name             = "prod-app-instance-pool"

  placement_configurations {
    availability_domain = var.availability_domain
    primary_subnet_id   = var.private_subnet_id
  }
}

# 3. OCI Metric-Based Autoscaling Policy
resource "oci_autoscaling_auto_scaling_configuration" "pool_autoscaling" {
  compartment_id       = var.compartment_id
  cool_down_in_seconds = 300
  is_enabled           = true

  auto_scaling_resources {
    id   = oci_core_instance_pool.app_pool.id
    type = "instancePool"
  }

  policy {
    policy_type = "threshold"
    capacity {
      initial = 2
      min     = 2
      max     = 20
    }

    rules {
      action {
        type  = "CHANGE_COUNT_BY"
        value = 2
      }
      display_name = "Scale Out Rule"
      metric {
        metric_type = "CPU_UTILIZATION"
        threshold {
          operator = "GT"
          value    = 75
        }
      }
    }

    rules {
      action {
        type  = "CHANGE_COUNT_BY"
        value = -1
      }
      display_name = "Scale In Rule"
      metric {
        metric_type = "CPU_UTILIZATION"
        threshold {
          operator = "LT"
          value    = 25
        }
      }
    }
  }
}
```

## 8. Data Flow
```text
Autoscaling Execution Sequence:
1. Traffic surges from 1,000 RPS to 8,000 RPS in 30 seconds.
2. ALB metrics aggregate: RequestCountPerTarget spikes to 2,000.
3. Target Tracking Controller detects breach:
   - Target is 1,000. Current is 2,000. Ratio = 2.0x.
   - Current instances = 4. New required instances = 8.
4. ASG / Instance Pool calls EC2 / OCI Compute API: Launch 4 instances.
5. Instances launch across Fault Domains / AZs:
   - Boot OS, execute cloud-init bootstrap (120 seconds).
   - Enter 'InService' or 'Running'.
6. ALB registers new targets:
   - Connection draining grace period active.
   - Health checks verify HTTP 200 OK.
7. Traffic rebalances across 8 instances; RequestCountPerTarget returns to 1,000!
```

## 9. Security
- **Dynamic Security Group Binding**:
  - Launch templates / configurations must strictly assign instance security groups dynamically on boot.
  - Instances must never be assigned public IP addresses; ingress is routed exclusively through Load Balancers.
- **Ephemeral IAM Instance Profiles**:
  - Autoscaled instances authenticate to databases and cloud APIs using IAM Roles and OCI Instance Principals with zero persistent credentials stored in AMIs.

## 10. Reliability
- **Connection Draining (Deregistration Delay)**:
  - When the autoscaling engine scales in and terminates an instance, active customer in-flight HTTP requests must not be severed!
  - **Configure Connection Draining (300 seconds)**:
    - The Load Balancer stops routing *new* requests to the deregistering instance.
    - It allows *existing in-flight requests* up to 300 seconds to complete gracefully before sending the `SIGTERM` / termination signal.

## 11. Scaling
- **The Metric Aggregation Lag Problem**:
  - Standard CloudWatch metrics aggregate every 1 minute.
  - If traffic spikes instantly, the autoscaling controller won't detect the surge until minute 2 or 3.
  - *Optimization*: For spiky flash sales, combine Target Tracking with **Step Scaling on 1-minute high-resolution metrics**, or pre-scale via **Scheduled Scaling**.

## 12. Observability
- **Monitoring Scaling Operations**:
  - Track `GroupInServiceInstances`, `GroupPendingInstances`, and `GroupTerminatingInstances` in CloudWatch.
  - In OCI, monitor `InstancePoolSize` and `PoolCapacityUtilization` in OCI Monitoring.

## 13. Cost
- **FinOps Sizing Optimization**:
  - Autoscaling saves 50% to 70% of compute spend compared to running peak capacity 24/7.
  - Sizing Rule: Ensure `min_size` sustains the minimum baseline off-peak traffic, while `max_size` caps the financial blast radius to prevent runaway bills from DDoS attacks.

## 14. Failure Modes
- **The Autoscaling Flapping Meltdown**: Setting a scale-out threshold of CPU $> 60\%$ and a scale-in threshold of CPU $< 55\%$ with a 60-second cooldown. When 2 instances are added, CPU drops to 54%; the system terminates an instance; CPU spikes to 61%; the system launches an instance. The cluster enters an infinite flapping loop, dropping connections continuously. *Remediation: Maintain a wide hysteresis gap (e.g., scale-out at 75%, scale-in at 25%) and a 300-second cooldown.*
- **The Broken AMI Startup Death Loop**: A deployment updates the Launch Template with an AMI containing a syntax error in the systemd startup script. The instance boots, fails application health checks, and the ASG terminates it. The ASG launches another instance, which fails and terminates. The ASG launches thousands of instances over a weekend, burning the entire monthly cloud budget on non-functional compute!

## 15. Troubleshooting
When autoscaling fails to launch or terminate instances:
1. **Inspect ASG Scaling Activity History**:
   ```bash
   aws autoscaling describe-scaling-activities --auto-scaling-group-name prod-app-asg \
     --query "Activities[0:5].[Description,StatusCode,StatusMessage]"
   ```
   Look for `Failed` activities with messages like `InsufficientInstanceCapacity` (cloud provider out of stock in that AZ) or `VcpuLimitExceeded`.
2. **Verify Subnet IP Address Availability**:
   - If the private subnet runs out of free private IPv4 addresses, new instances fail to launch!

## 16. Common Mistakes
- **Using CPU Utilization for I/O-Bound Services**: Configuring CPU-based autoscaling on a microservice that queries a slow external database. The CPU sits at 15%, but all 200 HTTP worker threads are blocked waiting on network sockets. The service collapses without ever triggering a CPU autoscaling event! Use **`ALBRequestCountPerTarget`** or **Active Connection Count**.
- **Forgetting Health Check Grace Period**: Setting a 30-second grace period for an application that takes 90 seconds to boot JVM classes. The Load Balancer checks health at second 35, sees the port closed, declares the instance unhealthy, and terminates it before it ever finishes booting.

## 17. Trade-offs
| Policy Type | Reaction Speed | Implementation Complexity | Cost Efficiency |
| :--- | :--- | :--- | :--- |
| **Scheduled Scaling** | **Instant (Zero lag)** | Low (Cron expressions) | Moderate (Pre-provisions) |
| **Target Tracking** | Moderate (2–5 mins) | **Lowest (Thermostat model)**| **Highest (Matches load)** |
| **Step Scaling** | Fast (Aggressive steps)| Moderate (Threshold tuning) | Moderate |
| **Predictive Scaling** | **Proactive (Warmed ahead)**| High (Requires historical data)| High |

## 18. Interview Questions
1. *What is Autoscaling Flapping (Thrashing)? How do you design autoscaling thresholds, cooldown periods, and hysteresis bands to mathematically eliminate flapping?*
2. *Why is scaling on CPU utilization an architectural anti-pattern for network-bound or I/O-bound microservices behind an Application Load Balancer? What metric must be used instead?*
3. *How do OCI Instance Pools and Autoscaling Configurations compare to AWS EC2 Auto Scaling Groups in terms of Launch Templates and Fault Domain placement?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "Scaling on raw CPU utilization for I/O-bound or network-bound microservices is a critical architectural anti-pattern that leads directly to silent production outages:
>
> 1. **The Architectural Bottleneck of I/O-Bound Workloads**:
>    - Consider an API microservice that validates payment transactions by querying a downstream database or third-party banking API.
>    - When traffic surges, the application's web worker thread pool (e.g., Tomcat, Puma, or Node.js event loop) becomes saturated because threads block waiting on network socket I/O.
>    - Because blocked threads consume **zero CPU cycles**, the virtual machine's CPU utilization remains low (typically 15% to 25%).
>    - The CloudWatch CPU metric never breaches the 70% threshold. The Auto Scaling Group refuses to scale out, even as incoming client connections back up in the load balancer queue, timeout, and fail with HTTP 504 Gateway Timeouts!
>
> 2. **The Correct Production Metric: `ALBRequestCountPerTarget`**:
>    - Instead of measuring internal resource exhaustion (CPU), we measure **Demand at the Ingress Gateway using `ALBRequestCountPerTarget`**:
>    - Through load testing, we determine that a single instance of our service can comfortably process **500 concurrent requests per minute** while maintaining a sub-50ms p99 response time.
>    - We configure an AWS **Target Tracking Autoscaling Policy** targeting exactly `500` requests per target.
>
> 3. **The Scaling Mechanics**:
>    - If total incoming traffic is 5,000 requests per minute across 4 instances, each instance is handling 1,250 requests (breaching the 500 target).
>    - The scaling controller calculates the mathematical ratio:
>      $$\text{Required Instances} = \frac{5,000}{500} = \mathbf{10\text{ instances}}.$$
>    - The ASG immediately launches 6 additional instances, distributing the traffic so each instance handles exactly 500 requests, completely insulating the service from I/O thread starvation regardless of CPU utilization."

## 20. Hands-on Exercise
**Objective**: Create an AWS Auto Scaling Target Tracking policy and verify target capacity calculations via CLI.

### Verification Steps
1. Create an ASG attached to an ALB target group.
2. Apply a Target Tracking scaling policy targeting `ALBRequestCountPerTarget = 1000`.
3. Inspect the created CloudWatch alarms generated automatically by the Target Tracking controller:
   ```bash
   aws autoscaling describe-policies --auto-scaling-group-name prod-app-asg \
     --query "ScalingPolicies[*].Alarms[*].AlarmName"
   ```
   Observe that AWS automatically generates **two paired CloudWatch alarms**: one for High Threshold (Scale Out) and one for Low Threshold (Scale In), with calculated hysteresis bands.
4. Confirm that the health check grace period is set to at least 300 seconds to allow clean instance boot.
