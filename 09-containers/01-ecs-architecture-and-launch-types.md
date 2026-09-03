# 01. ECS Architecture & Launch Types

## 1. Problem
Running containers on unmanaged virtual machines introduces severe operational burdens: manual container scheduling, port collisions when running multiple instances of the same service on a single host, fragmented compute capacity, and host OS patching. While Kubernetes has become the enterprise standard, its steep operational complexity, multi-component control plane, and administrative overhead are frequently excessive for teams seeking to run straightforward microservices. Amazon Elastic Container Service (ECS) and AWS Fargate provide high-performance, deeply integrated container orchestration without Kubernetes management toil.

## 2. Cloud Concept
### The Core Primitives of Amazon ECS
- **Cluster**: A regional grouping of compute capacity (EC2 instances or Fargate infrastructure) where container workloads run.
- **Task Definition**: The immutable blueprint (JSON manifest) specifying container images, CPU/memory reservations, port mappings, environment variables, storage mounts, and IAM roles.
- **Task**: The running instantiation of a Task Definition (analogous to a single Kubernetes Pod).
- **Service**: The supervisor responsible for maintaining a specified number of simultaneous, healthy Tasks (e.g., maintain 5 copies of Task Definition `api:v2`). Manages rolling updates, target group registration with Application Load Balancers, and auto-scaling.

### Launch Types: EC2 vs. AWS Fargate
- **EC2 Launch Type**:
  - You manage the underlying EC2 instances registered in the ECS cluster.
  - *Pros*: Full control over instance types, GPU shapes, custom AMIs, SSH access, and cost optimization using Spot instances or Reserved Instances.
  - *Cons*: Operational overhead of patching host operating systems, managing Docker daemon versions, and configuring EC2 Auto Scaling Groups.
- **AWS Fargate (Serverless Compute)**:
  - You define the required vCPU and memory; AWS provisions and manages the underlying compute infrastructure.
  - *Pros*: Zero server management, zero OS patching, seamless per-second billing, and hardware-level isolation for every Task `[Doc: AWS Fargate Overview, checked 2026-09-03]`.
  - *Cons*: Slightly higher baseline unit compute cost; no local daemon sets or direct host access.

### Task Networking Modes
1. **`awsvpc` (The Enterprise Standard)**:
   - Every ECS Task receives its own dedicated Elastic Network Interface (ENI) and its own private IPv4 address allocated directly from the VPC subnet.
   - Tasks enjoy full first-class citizen status in the VPC: Security Groups can be applied **directly to individual Tasks**, rather than the shared host.
   - *Mandatory for AWS Fargate*.
2. **`bridge`**:
   - Uses Docker's internal virtual bridge. Multiple containers share the host's single ENI.
   - Requires dynamic host port mapping (e.g., host maps container port 80 to ephemeral port 49152) and ALB integration to discover dynamic ports.
3. **`host`**:
   - Containers bind directly to the host's network stack, bypassing Docker bridge NAT for maximum performance. Cannot run multiple instances of the same container on the same host if ports collide.
4. **`none`**:
   - Zero external network connectivity. Used for isolated batch jobs.

## 3. Mental Model
Think of container orchestration models as residential housing:
- **EC2 Launch Type** is buying an apartment building. You own the brick and mortar, maintain the plumbing, repair the roof (patch host OS), and decide how many tenants (containers) fit inside each apartment.
- **AWS Fargate** is booking hotel rooms. You show up, stay in a private room (dedicated microVM), pay for the exact hours you sleep, and never worry about who fixes the boiler or replaces the building's roof.
- **`awsvpc` Networking** gives every apartment its own individual mailbox and private street address, rather than having all residents share a single giant mailbox in the building lobby (`bridge` mode).

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                        AMAZON ECS CLUSTER FABRIC                       │
│                                                                        │
│   INCOMING TRAFFIC: Application Load Balancer (ALB)                   │
│         │                                                              │
│         ├──────────────────────────────┬───────────────────────────────┤
│         ▼                              ▼                               ▼
│   AWS FARGATE LAUNCH TYPE       EC2 LAUNCH TYPE               EC2 LAUNCH TYPE
│   ┌──────────────────────┐      ┌───────────────────────────┐ ┌─────────────┐
│   │ TASK: checkout (v2)  │      │ EC2 INSTANCE (m6i.xlarge) │ │ EC2 INSTANCE│
│   │ * Dedicated ENI      │      │ ┌───────────┐ ┌─────────┐ │ │             │
│   │ * Private IP:        │      │ │Task: API  │ │Task: API│ │ │             │
│   │   10.0.1.50          │      │ │(awsvpc ENI│ │(awsvpc) │ │ │             │
│   │ * Task Security Group│      │ │10.0.1.80) │ │10.0.1.81│ │ │             │
│   │ * MicroVM Isolation  │      │ └───────────┘ └─────────┘ │ │             │
│   └──────────────────────┘      │ ECS Container Agent       │ │             │
│   (Zero Host Management)        └───────────────────────────┘ └─────────────┘
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS ECS:
- **Capacity Providers & Cluster Auto Scaling (CAS)**:
  - Dynamically scales the underlying EC2 Auto Scaling Group based on pending container resource demands.
  - Automatically matches Task vCPU/memory requirements to optimal EC2 instance sizes.
- **ECS Deployment Circuit Breaker**:
  - Monitors rolling deployments. If new tasks fail health checks and crash-loop, the circuit breaker automatically halts the deployment and rolls back the ECS Service to the last healthy Task Definition without human intervention `[Doc: ECS Deployment Circuit Breaker, checked 2026-09-03]`.
- **Fargate Spot**: Delivers up to 70% discounts on Fargate compute for fault-tolerant batch workloads.

## 6. OCI Implementation
In Oracle Cloud Infrastructure:
- **OCI Container Architecture Comparison**:
  - While AWS offers ECS as a proprietary non-Kubernetes orchestrator alongside EKS, **OCI takes a cleaner, bifurcated architectural approach**:
    1. **Full Orchestration**: Handled natively by **OCI Container Engine for Kubernetes (OKE)**.
    2. **Serverless Lightweight Execution**: Handled by **OCI Container Instances** `[Doc: OCI Container Instances, checked 2026-09-03]`.
- **OCI Container Instances vs. AWS Fargate**:
  - *Instant Startup*: Launches in seconds without requiring an orchestrator cluster, Task Definition manifests, or Service constructs.
  - *Native VCN Integration*: Every container instance is attached directly to your VCN subnet via a native VNIC (identical to `awsvpc` mode), inheriting private IPs, Security Lists, and NSGs.
  - *Resource Flexibility*: Sized using OCI Flexible Shapes, allowing precise specification of OCPUs (0.1 to 64) and memory (1 GB to 1,024 GB) per container instance.
  - *No Cluster Maintenance*: Perfect for batch jobs, webhook handlers, and CI/CD pipelines where managing an ECS cluster or Kubernetes control plane is unnecessary overhead.

## 7. Configuration
Comparing serverless container provisioning in Terraform across AWS and OCI:

### AWS ECS Fargate Service (Terraform)
```hcl
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "production-ecs-cluster"
}

# Task Definition with awsvpc networking
resource "aws_ecs_task_definition" "api" {
  family                   = "api-task"
  network_mode             = "awsvpc" # Mandatory for Fargate!
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"    # 0.5 vCPU
  memory                   = "1024"   # 1 GB RAM
  execution_role_arn       = var.ecs_execution_role_arn
  task_role_arn            = var.ecs_task_role_arn

  container_definitions = jsonencode([
    {
      name      = "api-container"
      image     = "${var.ecr_repo_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 8080
          hostPort      = 8080
          protocol      = "tcp"
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/api-task"
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# ECS Fargate Service
resource "aws_ecs_service" "api_service" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [var.task_security_group_id]
    assign_public_ip = false
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}
```

### OCI Container Instance (Terraform)
```hcl
# OCI Serverless Container Instance
resource "oci_container_instances_container_instance" "api_worker" {
  compartment_id      = var.compartment_id
  availability_domain = var.availability_domain
  display_name        = "api-container-instance"
  shape               = "CI.Standard" # Container Instance Shape

  shape_config {
    ocpus         = 1
    memory_in_gbs = 4
  }

  vnics {
    subnet_id        = var.private_subnet_id
    assign_public_ip = false
    nsg_ids          = [var.container_nsg_id]
  }

  containers {
    display_name = "worker-container"
    image_url    = "${var.ocir_repo_url}:latest"

    resource_config {
      vcpus_limit   = 1
      memory_in_gbs = 4
    }
  }
}
```

## 8. Data Flow
```text
Task Ingress and Network Traversal in awsvpc Mode:
1. External client sends request to Public ALB.
2. ALB terminates TLS, consults target group:
   * Target group contains private IP addresses of ECS tasks (e.g., 10.0.1.50:8080).
3. ALB forwards request across private VPC subnet directly to Task ENI (10.0.1.50).
4. Task Hypervisor / Fargate Nitro card inspects Task Security Group:
   * Verifies source is ALB Security Group.
   * Delivers packet directly to container network namespace.
5. Zero host port remapping (bridge) overhead; zero IP table translation lag!
```

## 9. Security
- **The Two IAM Roles of ECS**:
  - **Task Execution Role (`execution_role_arn`)**: Used by the **AWS ECS infrastructure agent**. Authorizes pulling private container images from Amazon ECR, fetching decrypted secrets from AWS Secrets Manager/SSM, and shipping logs to CloudWatch.
  - **Task Role (`task_role_arn`)**: Used by the **application code running inside the container**. Authorizes application code to execute SQL queries, write objects to Amazon S3, or publish events to SNS/SQS.
  - *Critical Security Rule*: Never combine these two roles. Application code must never possess permissions to pull container images or modify ECS infrastructure.

## 10. Reliability
- **Rolling Deployments & Minimum/Maximum Percent**:
  - `minimum_healthy_percent = 100`: Ensures existing capacity never drops below 100% during a deployment (AWS launches new tasks before terminating old ones).
  - `maximum_percent = 200`: Allows ECS to double task count temporarily during rollouts to prevent capacity degradation.

## 11. Scaling
- **ENI Trunking on EC2 Container Hosts**:
  - When running tasks in `awsvpc` mode on the EC2 Launch Type, every task requires a dedicated ENI.
  - Standard EC2 instances have hard ENI limits (e.g., an `m5.large` supports only 3 ENIs, capping the host at 2 tasks!).
  - *Solution*: Enable **ENI Trunking** (`awsvpcTrunking`), which allows modern Nitro instances to attach up to 4x more ENIs per host, increasing task density significantly.

## 12. Observability
- **ECS CloudWatch Metrics**:
  - `CPUUtilization` and `MemoryUtilization` aggregated at the Cluster and Service level.
  - **Container Insights**: Collects diagnostic metrics, CPU/memory saturation, and container disk metrics using embedded CloudWatch agent DaemonSets.

## 13. Cost
- **Fargate Pricing**:
  - Billed strictly per vCPU-hour (\$0.04048/vCPU-hr) and per GB-hour (\$0.004445/GB-hr) rounded to the nearest second `[Doc: AWS Fargate Pricing, checked 2026-09-03]`.
  - Zero base cluster fees; you pay only for running tasks.
- **OCI Container Instances Pricing**:
  - Billed based on the standard OCI Compute flexible shape rates for the provisioned OCPUs and memory. No premium markup for the serverless container abstraction.

## 14. Failure Modes
- **The Task Definition Secret Fetch Failure**: A new Task Definition references a secret ARN in AWS Secrets Manager. An engineer forgets to update the **Task Execution Role** with `secretsmanager:GetSecretValue`. When ECS attempts to launch the new task, the task crashes instantly in the `PROVISIONING` state. The deployment freezes indefinitely until the deployment circuit breaker triggers a rollback.
- **Subnet IP Exhaustion via awsvpc Mode**: An auto-scaling ECS Fargate service expands from 10 to 200 tasks in a small `/24` subnet ($251$ usable IPs). Because each Fargate task consumes a dedicated ENI IP, the subnet exhausts its address space, causing new tasks to fail with `RESOURCE:ENI` errors and blocking production autoscaling.

## 15. Troubleshooting
When an ECS Task fails to start or continuously crash-loops:
1. **Check Task Stopped Reason**:
   ```bash
   aws ecs describe-tasks --cluster <cluster> --tasks <task-id> \
     --query "tasks[0].containers[0].[exitCode, reason]"
   ```
2. **Interpret Exit Codes**:
   - `ExitCode: 137`: Container killed by Linux kernel OOM (Out-Of-Memory) killer. Increase Task memory reservation.
   - `ExitCode: 1`: Uncaught application exception (check application logs in CloudWatch `/ecs/<task>`).
   - `CannotPullContainerError`: ECR authentication failure, missing VPC endpoint for ECR, or missing Task Execution Role permissions.

## 16. Common Mistakes
- **Putting Secrets in Plaintext Environment Variables**: Storing database passwords in the `environment` block of a Task Definition. Anyone with `ecs:DescribeTaskDefinition` can read plaintext credentials. Always reference secrets via the `secrets` block targeting AWS Secrets Manager or OCI Vault.
- **Failing to Configure Health Check Grace Periods**: Registering an ECS Service with an ALB with a 0-second grace period. The ALB probes the container before the Spring Boot / Node.js process finishes initializing, marks it unhealthy, and triggers a continuous restart loop. Set `health_check_grace_period_seconds >= 60`.

## 17. Trade-offs
| Container Architecture | Pros | Cons | Best Use Case |
| :--- | :--- | :--- | :--- |
| **AWS ECS (Fargate)** | Zero server management, per-second billing, native IAM | Proprietary manifest format; no GPU Fargate support | Standard microservices, web APIs, queue consumers |
| **AWS ECS (EC2)** | Full host OS access, Spot instances, GPU shapes | Host OS patching & capacity scaling overhead | High-density compute, custom kernel modules |
| **OCI Container Instances** | Instant launch, true VCN integration, zero cluster fees | Single-task/batch focused; no complex ingress routing | Batch jobs, CI/CD runners, scheduled tasks |
| **Kubernetes (EKS / OKE)** | Open standard, rich ecosystem, multi-cloud portability | High management complexity, steep learning curve | Enterprise microservice meshes, complex stateful apps |

## 18. Interview Questions
1. *In an enterprise architecture review, a junior engineer proposes deploying Kubernetes (EKS/OKE) for 5 simple microservices. How do you evaluate and challenge this decision using AWS ECS / Fargate or OCI Container Instances?*
2. *Explain the precise architectural difference between an ECS Task Execution Role and an ECS Task Role. What catastrophic security vulnerability occurs if these are combined?*
3. *Why does the `awsvpc` networking mode consume more IP addresses than `bridge` mode, and how do you prevent subnet IP exhaustion in large Fargate deployments?*

## 19. Interview Answer
**Exemplary Answer to Question 2**:
> "The architectural separation between the **ECS Task Execution Role** and the **ECS Task Role** enforces the principle of least privilege between the **cloud infrastructure control plane** and the **untrusted application runtime**:
>
> 1. **ECS Task Execution Role (`execution_role_arn`)**:
>    - This role is assumed by the **AWS ECS container agent** running on the host (or within the Fargate infrastructure) *before* the container is launched.
>    - Its responsibilities are purely operational: authorizing the agent to authenticate with Amazon ECR to pull private Docker images, fetching decrypted environment secrets from AWS Secrets Manager, and streaming container logs to Amazon CloudWatch Logs.
>    - The application code inside the running container has **zero access** to this role.
>
> 2. **ECS Task Role (`task_role_arn`)**:
>    - This role is delivered directly into the container's environment via the AWS credentials metadata endpoint (`169.254.170.2`).
>    - It is assumed by the **application business logic** executing inside the container to interact with AWS APIs—such as querying Amazon DynamoDB, reading files from Amazon S3, or publishing messages to Amazon SQS.
>
> 3. **The Catastrophic Vulnerability of Combining Them**:
>    - If an engineer creates a single monolithic IAM role and assigns it to both parameters, the application container inherits the permissions of the infrastructure agent.
>    - If an attacker discovers a Remote Code Execution (RCE) or Server-Side Request Forgery (SSRF) vulnerability in the web application, the attacker can query the metadata endpoint, steal temporary IAM credentials, and gain permissions to **pull all proprietary container images across the entire corporate ECR registry** and **read every secret stored in AWS Secrets Manager**.
>    - Strict separation of these two roles is an absolute, non-negotiable security baseline in enterprise cloud engineering."

## 20. Hands-on Exercise
**Objective**: Deploy a secure ECS Fargate task in Terraform with distinct Execution and Task IAM roles.

### Verification Steps
1. Provision an ECS Cluster and Task Definition with `network_mode = "awsvpc"`.
2. Configure `execution_role_arn` with `AmazonECSTaskExecutionRolePolicy`.
3. Configure `task_role_arn` with read-only access to a specific S3 bucket (`s3:GetObject`).
4. Execute `aws ecs execute-command` into the running container.
5. Attempt an S3 read: `aws s3 cp s3://<bucket>/test.txt .` $\longrightarrow$ Success.
6. Attempt an ECR read: `aws ecr describe-repositories` $\longrightarrow$ Denied (`AccessDeniedException`), verifying that application runtime cannot access infrastructure capabilities.
