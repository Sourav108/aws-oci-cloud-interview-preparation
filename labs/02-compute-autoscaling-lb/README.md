# Lab 02: Compute Autoscaling & Load Balanced Fleets

---

## 1. Goal & Core Concept

The objective of this hands-on lab is to deploy, configure, and validate an elastic, self-healing web tier utilizing **Application Load Balancers** and **Horizontal Auto Scaling Groups / Instance Pools** across AWS and OCI.

### Core Architectural Concepts Tested
- **Layer-7 Reverse Proxy Routing**: Path-based routing (`/api` vs `/static`) and SSL/TLS termination.
- **Dynamic Horizontal Scaling**: AWS Target Tracking Scaling Policies (Target CPU = 65%) vs. OCI Autoscaling Configurations (Metric-based threshold rules).
- **Health Check Mechanics**: Distinguishing between Load Balancer health checks and Compute Hypervisor/Instance status checks.
- **Self-Healing Automation**: Automatic replacement of unhealthy compute instances without manual operator intervention.

---

## 2. Cost Warning & Resource Estimate

> [!NOTE]
> **ESTIMATED HOURLY RUN COST: `< $0.15 / hour`**
>
> | Cloud Provider | Resource Component | Quantity | Hourly Rate | Run Cost (2 Hours) |
> | :--- | :--- | :---: | :---: | :---: |
> | **AWS** | Application Load Balancer (ALB) `[Doc: ALB, checked 2026]` | 1 | $0.0225 / hr | $0.045 |
> | **AWS** | EC2 `t4g.nano` (2 Baseline Instances) `[Doc: EC2, checked 2026]` | 2 | $0.0042 / hr each | $0.017 |
> | **OCI** | Flexible Load Balancer (10-100 Mbps) `[Doc: OCI LB, checked 2026]` | 1 | $0.0113 / hr | $0.023 |
> | **OCI** | `VM.Standard.E5.Flex` (1 OCPU / 4 GB) or Free-Tier A1 | 2 | $0.025 / hr each | $0.050 |
>
> *Total Estimated Lab Cost for a 2-hour session: **$0.135 USD**.*
> *In No-Live-Account Fallback Mode, cost is **$0.00**.*

---

## 3. Architecture Diagram

```
========================================================================================================================
                               AUTOSCALING LOAD-BALANCED COMPUTE TOPOLOGY
========================================================================================================================

                                        [ Client Traffic (HTTP GET /) ]
                                                       │
                                                       ▼
                            [ Layer 7 Load Balancer: ALB / OCI Flexible LB ]
                            - Health Check Target: /healthz (Port 80)
                            - Forwarding Rule: Round-Robin distribution
                                                       │
                           ┌───────────────────────────┴───────────────────────────┐
                           │ (Private Subnet IP Forwarding)                        │
                           ▼                                                       ▼
  ═══════════════════════════════════════════════════════  ═══════════════════════════════════════════════════════
  FAILURE DOMAIN 1 (AWS us-east-1a / OCI Fault-Domain-1)   FAILURE DOMAIN 2 (AWS us-east-1b / OCI Fault-Domain-2)
  ┌─────────────────────────────────────────────────────┐  ┌─────────────────────────────────────────────────────┐
  │ [ Compute Instance 1 (t4g.nano / E5.Flex) ]         │  │ [ Compute Instance 2 (t4g.nano / E5.Flex) ]         │
  │  - Nginx Web Server + Health Endpoint               │  │  - Nginx Web Server + Health Endpoint               │
  │  - CloudWatch / OCI Monitoring Metric Agent         │  │  - CloudWatch / OCI Monitoring Metric Agent         │
  └─────────────────────────────────────────────────────┘  └─────────────────────────────────────────────────────┘
  ═══════════════════════════════════════════════════════  ═══════════════════════════════════════════════════════
                           ▲                                                       ▲
                           │                                                       │
                           └─────────────────[ AUTOSCALING CONTROLLER ]────────────┘
                                             - Scale-Out: Average CPU > 65% (Adds +2 Instances)
                                             - Scale-In:  Average CPU < 30% (Removes -1 Instance)
```

---

## 4. Prerequisites

1. Existing VPC / VCN with at least two public and two private subnets (from Lab 01).
2. SSH Keypair generated (`id_rsa.pub`) for debugging compute nodes.
3. IAM Instance Profile (AWS) / Dynamic Group (OCI) permitting compute instances to push metrics.

---

## 5. Infrastructure Code (Terraform HCL)

### 5.1 AWS Autoscaling & ALB (`aws_autoscaling.tf`)

```hcl
# AWS Reference Implementation: Launch Template, Auto Scaling Group, and ALB
resource "aws_launch_template" "web_template" {
  name_prefix   = "lab02-web-"
  image_id      = "ami-0c7217cdde317cfec" # Amazon Linux 2023 ARM64
  instance_type = "t4g.nano"

  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              dnf install -y nginx stress
              systemctl enable --now nginx
              echo "<h1>Instance $(hostname -f) Healthy</h1>" > /usr/share/nginx/html/index.html
              echo "OK" > /usr/share/nginx/html/healthz
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags          = { Name = "lab02-web-asg-node" }
  }
}

resource "aws_autoscaling_group" "web_asg" {
  name                = "lab02-web-asg"
  desired_capacity    = 2
  max_size            = 6
  min_size            = 2
  target_group_arns   = [aws_lb_target_group.web_tg.arn]
  vpc_zone_identifier = [var.private_subnet_az1_id, var.private_subnet_az2_id]
  health_check_type   = "ELB" # Replaces unhealthy instances when ALB probe fails

  health_check_grace_period = 180

  launch_template {
    id      = aws_launch_template.web_template.id
    version = "$Latest"
  }
}

resource "aws_autoscaling_policy" "cpu_target_tracking" {
  name                   = "target-tracking-cpu-65"
  autoscaling_group_name = aws_autoscaling_group.web_asg.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 65.0
  }
}
```

### 5.2 OCI Instance Pool & Autoscaling (`oci_autoscaling.tf`)

```hcl
# OCI Reference Implementation: Instance Configuration, Pool, and Autoscaling
resource "oci_core_instance_configuration" "web_config" {
  compartment_id = var.compartment_ocid
  display_name   = "lab02-web-instance-config"

  instance_details {
    instance_type = "compute"

    launch_details {
      compartment_id = var.compartment_ocid
      shape          = "VM.Standard.E5.Flex"

      shape_config {
        ocpus         = 1
        memory_in_gbs = 4
      }

      source_details {
        source_type = "image"
        image_id    = var.oracle_linux_image_id
      }

      metadata = {
        user_data = base64encode(<<-EOF
                    #!/bin/bash
                    dnf install -y nginx stress
                    systemctl enable --now nginx
                    echo "OK" > /usr/share/nginx/html/healthz
                    EOF
        )
      }
    }
  }
}

resource "oci_core_instance_pool" "web_pool" {
  compartment_id            = var.compartment_ocid
  instance_configuration_id = oci_core_instance_configuration.web_config.id
  display_name              = "lab02-web-instance-pool"
  size                      = 2

  placement_configurations {
    availability_domain = var.ad_name
    primary_subnet_id   = var.private_subnet_ocid
    fault_domains       = ["FAULT-DOMAIN-1", "FAULT-DOMAIN-2"]
  }

  load_balancers {
    load_balancer_id = oci_load_balancer_load_balancer.web_lb.id
    backend_set_name = oci_load_balancer_backend_set.web_backend.name
    port             = 80
    vnic_selection   = "PrimaryVnic"
  }
}
```

---

## 6. Step-by-Step Deployment Guide

```bash
# 1. Statically validate configurations
terraform init -backend=false
terraform validate

# 2. Review resources planned
terraform plan
```

---

## 7. Expected Validation Results

```text
[Statically validated — not applied to a live account]

Load Balancer Health Validation:
$ curl -s http://${ALB_DNS_NAME}/healthz
OK

Round Robin Verification:
$ for i in {1..4}; do curl -s http://${ALB_DNS_NAME}/; done
<h1>Instance ip-10-0-10-42.ec2.internal Healthy</h1>
<h1>Instance ip-10-0-20-89.ec2.internal Healthy</h1>
<h1>Instance ip-10-0-10-42.ec2.internal Healthy</h1>
<h1>Instance ip-10-0-20-89.ec2.internal Healthy</h1>
```

---

## 8. Failure Injection Drill: High CPU Load Spiking

### The Scenario
Simulate a flash-sale traffic surge by pegging the CPU of an instance to 100% for 5 minutes using the `stress` utility, triggering an automatic scale-out event.

### The Injection
```bash
# SSH into Instance 1 and spawn 4 CPU-burning worker threads
ssh ec2-user@10.0.10.42 "stress --cpu 4 --timeout 300s"
```

### Manifested Symptoms
- CloudWatch metric `ASGAverageCPUUtilization` / OCI `CpuUtilization` spikes from 5% to 85%.
- Target tracking alarm transitions from `OK` to `ALARM`.
- Auto Scaling Group provisions 2 additional instances to absorb load.

---

## 9. Debugging Walkthrough

```text
====================================================================================================
                        AUTOSCALING SCALE-OUT VERIFICATION
====================================================================================================

Step 1: Inspect Autoscaling Activity History
  $ aws autoscaling describe-scaling-activities --auto-scaling-group-name lab02-web-asg
  Output:
  [
    {
      "ActivityId": "act-12345678",
      "StatusCode": "Successful",
      "Description": "Executing state change: TargetTracking: Scaling out fleet from 2 to 4 instances."
    }
  ]

Step 2: Verify Target Group Registration
  $ aws elbv2 describe-target-health --target-group-arn ${TG_ARN}
  Output:
  - Target: 10.0.10.42 (Status: Healthy)
  - Target: 10.0.20.89 (Status: Healthy)
  - Target: 10.0.10.95 (Status: Initial -> Healthy after 30s)
  - Target: 10.0.20.101 (Status: Initial -> Healthy after 30s)
====================================================================================================
```

---

## 10. Teardown & Verification

```bash
terraform destroy -auto-approve
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[?AutoScalingGroupName=='lab02-web-asg']"
```

---

## 11. Senior Interview Defense & Takeaways

> **Interviewer**: *"Why configure the Auto Scaling Group health check type to 'ELB' instead of the default 'EC2'?"*

**Candidate Defense**:
*"The default 'EC2' health check only evaluates hypervisor-level hardware status and kernel liveliness. If Nginx crashes, if the application deadlocks, or if the process runs out of memory, the EC2 instance status checks remain 100% green. The load balancer marks the target unhealthy and stops routing traffic, but the Auto Scaling Group never terminates the broken node!*

*By setting `health_check_type = "ELB"`, the ASG monitors the Layer-7 Application Load Balancer health check status. If the application stops responding to `/healthz`, the ASG automatically terminates the defective instance and launches a fresh, healthy replacement."*
