# 02. Serverless Concurrency, Cold Starts & Provisioning

## 1. Problem
In serverless architectures, performance unpredictability and catastrophic cascading outages are almost always caused by an incomplete understanding of **Concurrency** and **Cold Starts**. When a sudden traffic spike hits an un-warmed serverless application, incoming requests experience multi-second cold start latency delays as the cloud provider initializes runtimes and attaches network interfaces. More dangerously, because cloud accounts enforce shared regional concurrency quotas (e.g., 1,000 concurrent executions), an unthrottled, runaway background image-processing function can consume 100% of the account's concurrency pool, causing business-critical customer checkout APIs to fail with HTTP 429 `Too Many Requests` throttling errors.

## 2. Cloud Concept
### Understanding Serverless Concurrency
Concurrency is defined as the number of requests that your function is actively serving simultaneously at any given point in time:
$$\text{Concurrency} = \text{Average Invocations per Second (RPS)} \times \text{Average Execution Duration in Seconds}$$

If an API receives **1,000 requests per second** and each execution takes **500 milliseconds (0.5s)**:
$$\text{Required Concurrency} = 1,000 \times 0.5 = \mathbf{500\text{ concurrent execution environments}}$$

If a downstream database query slows down, increasing execution duration to **2 seconds**:
$$\text{Required Concurrency} = 1,000 \times 2.0 = \mathbf{2,000\text{ concurrent execution environments}}$$
A 4x increase in duration requires 4x more concurrency for the exact same transaction volume, triggering immediate account throttling!

### The Cold Start Lifecycle & VPC Networking
- **What is a Cold Start?**: When an invocation arrives and no idle warm execution environment exists, the platform must provision a new microVM, download the code artifact, start the language runtime, and execute global initialization code.
- **The Modern VPC Cold Start Revolution**:
  - *Legacy Architecture (Pre-2019)*: Placing a Lambda function inside a private VPC required dynamically creating an Elastic Network Interface (ENI) on every cold start, adding **10 to 30 seconds of latency** per cold start!
  - *Modern Architecture (AWS Hyperplane & OCI SmartNICs)*: AWS redesigned VPC networking using AWS Hyperplane. ENIs are pre-provisioned at the subnet level when the function is deployed. Multiple concurrent Lambda microVMs share these pre-warmed tunnel interfaces, slashing VPC cold start latency to **under 50 milliseconds** `[Doc: AWS Lambda VPC Networking Improvements, checked 2026-09-03]`.

### The Three Concurrency Allocation Models
1. **Unreserved Concurrency (The Shared Pool)**: All functions in an account share a common pool (default: 1,000 concurrent executions in AWS). Unmanaged functions compete for this shared pool.
2. **Reserved Concurrency**:
   - Acts as a **Guaranteed Floor**: Guarantees that a dedicated slice of concurrency (e.g., 200) is always available for this specific function.
   - Acts as a **Hard Ceiling**: Caps the function so it can never exceed 200 concurrent executions, shielding downstream relational databases from being overwhelmed.
3. **Provisioned Concurrency**:
   - Pre-warms a designated number of execution environments (e.g., 50 environments).
   - Executes the complete `Init` phase (runtime boot + global DB connection initialization) ahead of time.
   - Incoming invocations land on pre-warmed memory, **completely eliminating cold starts** ($p99 < 10\text{ms}$).

## 3. Mental Model
Think of serverless concurrency as cash registers at a supermarket:
- **Concurrency** is the number of active cash register lanes currently checking out customers.
- **Cold Start** is a manager running to the back office, unlocking a register, turning on the computer, logging in, and loading the cash drawer before scanning the first item.
- **Warm Start** is a cashier already sitting at the register ready to scan items instantly.
- **Provisioned Concurrency** is paying 5 cashiers to sit at open registers all morning waiting for shoppers, guaranteeing zero checkout lines even during the opening rush.
- **Reserved Concurrency** is reserving exactly 2 express lanes for VIP loyalty members that regular shoppers cannot touch.

## 4. Architecture Diagram
```text
┌────────────────────────────────────────────────────────────────────────┐
│                   SERVERLESS CONCURRENCY QUOTA MANAGEMENT              │
│                                                                        │
│   TOTAL ACCOUNT CONCURRENCY LIMIT: 1,000 CONCURRENT EXECUTIONS         │
│                                                                        │
│   RESERVED SLICE 1: Checkout API (Critical Path)                       │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Reserved Concurrency = 400 | Provisioned Concurrency = 100     │   │
│   │ * 100 Pre-warmed Environments: ZERO COLD STARTS!               │   │
│   │ * Can scale up to 400 without being impacted by noisy neighbors│   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│   RESERVED SLICE 2: Image Resizing Worker (Background Queue)           │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Reserved Concurrency = 100 (HARD CEILING!)                     │   │
│   │ * Prevents runaway batch processing from eating account pool!  │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│   UNRESERVED SHARED POOL: Remaining 500 Concurrency                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Shared across all unconfigured microservices in the account    │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 5. AWS Implementation
In AWS Lambda:
- **Provisioned Concurrency Auto Scaling**:
  - AWS Application Auto Scaling can scale Provisioned Concurrency dynamically based on schedules (e.g., scale up to 100 warm instances at 8:50 AM before market open) or metric tracking (`ProvisionedConcurrencyUtilization` maintained at 70%).
- **Account Concurrency Quota**:
  - Default soft limit: **1,000 concurrent executions per region** `[Doc: AWS Lambda Quotas, checked 2026-09-03]`. Can be increased to tens of thousands via AWS Support quota request.
  - AWS reserves at least **100 unreserved concurrency** that cannot be allocated to reserved functions, ensuring basic administrative functions can execute.

## 6. OCI Implementation
In Oracle Cloud Infrastructure Functions:
- **OCI Provisioned Concurrency**:
  - OCI Functions supports **Provisioned Concurrency** directly on individual functions `[Doc: OCI Functions Provisioned Concurrency, checked 2026-09-03]`.
  - Maintains pre-initialized Docker container pods on managed OKE nodes.
  - When provisioned concurrency is enabled, the Fn Project container runtime pulls the image from OCIR and runs container initialization in advance, delivering sub-millisecond execution times for latency-sensitive APIs.
- **Compartment-Level Concurrency Management**:
  - In OCI, tenancy resources are partitioned into **Compartments**.
  - OCI enforces service limits per compartment and availability domain. An architect can set strict function execution limits on a development compartment, preventing development testing from starving production functions in the production compartment.

## 7. Configuration
Comparing concurrency and provisioning configuration in Terraform across AWS and OCI:

### AWS Lambda with Reserved & Provisioned Concurrency (Terraform)
```hcl
# AWS Lambda Function with Reserved Concurrency (Hard Ceiling)
resource "aws_lambda_function" "checkout" {
  function_name                  = "checkout-api"
  role                           = aws_iam_role.lambda_role.arn
  handler                        = "index.handler"
  runtime                        = "nodejs20.x"
  filename                       = "checkout.zip"
  reserved_concurrent_executions = 200 # Reserved ceiling & floor!
}

# Publish a version (Mandatory for Provisioned Concurrency)
resource "aws_lambda_alias" "prod" {
  name             = "production"
  function_name    = aws_lambda_function.checkout.function_name
  function_version = aws_lambda_function.checkout.version
}

# Pre-warm 50 execution environments (Eliminates Cold Starts!)
resource "aws_lambda_provisioned_concurrency_config" "warm_pool" {
  function_name                     = aws_lambda_function.checkout.function_name
  qualifier                         = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 50
}
```

### OCI Function with Provisioned Concurrency (Terraform)
```hcl
# OCI Function with Provisioned Concurrency
resource "oci_functions_function" "checkout_oci" {
  application_id = var.application_id
  display_name   = "checkout-api"
  image          = "${var.ocir_repo_url}/checkout:v1.0"
  memory_in_mbs  = 1024

  # Keep 20 pre-warmed container instances active
  provisioned_concurrency_config {
    strategy = "CONSTANT"
    count    = 20
  }
}
```

## 8. Data Flow
```text
Provisioned vs. Unreserved Routing Flow:
1. Incoming API Gateway Call: POST /checkout
2. AWS Lambda / OCI Fn Router evaluates function target.
3. Check 1: Is Provisioned Concurrency available?
   * YES: Request routed immediately to pre-warmed container in memory.
   * Execution completes in 15ms. Cold start = 0ms.
4. Check 2: All 50 provisioned slots active?
   * Spill-over traffic routes to Standard On-Demand execution.
   * Cold start triggered for instance #51 (takes 250ms for Init).
   * Subsequent invocations on instance #51 remain warm.
```

## 9. Security
- **Mitigating Denial-of-Wallet Attacks**:
  - Because serverless scales automatically, a malicious actor flooding an unreserved function with 10,000,000 requests can generate thousands of dollars in compute bills within hours.
  - Setting **Reserved Concurrency** caps the maximum scale of the function, bounding financial liability.
  - Front serverless APIs with **AWS WAF** or **OCI WAF** to enforce rate-limiting rules (e.g., max 500 requests per 5-minute window per client IP).

## 10. Reliability
- **The "Noisy Neighbor" Account Throttling Brownout**:
  - In an unconfigured AWS account, Function A (payment API) and Function B (batch image processor) share the 1,000 concurrency pool.
  - A user uploads 10,000 images to S3. S3 triggers 1,000 concurrent instances of Function B.
  - Account concurrency reaches 1,000 / 1,000 (100% capacity).
  - A paying customer attempts to checkout via Function A. AWS immediately drops the request with **HTTP 429 `TooManyRequestsException`**!
  - *Reliability Mandate*: Always assign a **Reserved Concurrency ceiling** to asynchronous background batch functions to prevent them from exhausting the account.

## 11. Scaling
- **Cold Start Duration by Language Runtime**:
  - Compiled languages (Java, .NET, Scala) experience long cold starts ($1,000\text{ to }4,000\text{ms}$) due to JVM classloading.
  - Interpreted/Lightweight runtimes (Node.js, Python, Go, Rust) experience ultra-fast cold starts ($50\text{ to }250\text{ms}$).
  - *Optimization*: Use **AWS Lambda SnapStart** for Java (takes a MicroVM memory snapshot of initialized JVM state, reducing Java cold starts by 90% to $< 200\text{ms}$) `[Doc: AWS Lambda SnapStart, checked 2026-09-03]`.

## 12. Observability
- **Key Concurrency Metrics in CloudWatch**:
  - `ConcurrentExecutions`: Measures active concurrency.
  - `Throttles`: Number of invocation requests rejected due to rate limits or concurrency ceilings.
  - `ProvisionedConcurrencyUtilization`: Fraction of provisioned capacity in active use.
- **OCI Metrics**:
  - `ExecutionThrottles`: Counts requests rejected due to concurrency saturation.

## 13. Cost
- **Provisioned Concurrency Billing Math**:
  - Provisioned Concurrency charges for keeping environments warm, even if zero requests are served:
    $$\text{Cost} = (\text{Provisioned GB} \times \text{Hours} \times \$0.0000041667/\text{sec}) + (\text{Active Execution Time})$$
  - Keeping 100 instances of a 1 GB function warm 24/7 costs:
    $$100 \times 1\text{ GB} \times 730\text{ hrs} \times \$0.015/\text{GB-hr} \approx \mathbf{\$1,095/\text{month}}.$$
  - *Staff-Level Decision*: If a function processes consistent high-volume traffic 24/7, running on **AWS Fargate** or **OCI Container Instances** is often 40% cheaper than maintaining high Provisioned Concurrency on Lambda!

## 14. Failure Modes
- **The Accidental Zero Concurrency Kill Switch**: An engineer sets `reserved_concurrent_executions = 0` on a production Lambda function to "pause" it. This completely blocks 100% of invocations, instantly returning HTTP 429 to all callers.
- **Downstream Database Meltdown via Unthrottled Serverless Spikes**: A marketing campaign triggers 3,000 simultaneous Lambda invocations. Lambda scales to 3,000 instances in 10 seconds. Each instance opens a connection to an Amazon RDS PostgreSQL `db.t3.medium` (max connections: 150). The database runs out of connections instantly, deadlocking the entire backend.

## 15. Troubleshooting
When serverless functions throw HTTP 429 throttling errors:
1. **Determine Throttling Type**:
   - *Function-Level Throttling*: The function reached its own `Reserved Concurrency` limit. Remediation: Increase reserved concurrency or optimize execution duration.
   - *Account-Level Throttling*: The aggregate account concurrency reached 1,000. Remediation: Identify the offending runaway function using CloudWatch `ConcurrentExecutions` by function.
2. **Profile Execution Duration**:
   - If concurrency spikes unexpectedly while request volume remains steady, execution duration has increased (e.g., an external API dependency is running slow).

## 16. Common Mistakes
- **Using Java for Latency-Sensitive Serverless APIs Without SnapStart**: Building an interactive mobile backend on standard Java Lambda without SnapStart or Provisioned Concurrency. Users suffer 4-second delays on cold starts.
- **Failing to Set Alerts on Concurrency Utilization**: Allowing account-wide concurrency to hover at 950 / 1,000 without alerting, leaving zero buffer for traffic surges.

## 17. Trade-offs
| Concurrency Model | Cold Start Latency | Financial Cost | Blast Radius Protection |
| :--- | :--- | :--- | :--- |
| **Unreserved Concurrency** | Standard cold starts ($50–500\text{ms}$) | Zero baseline cost (Billed per request) | Zero (Vulnerable to account starvation) |
| **Reserved Concurrency** | Standard cold starts | Zero baseline cost | High (Guarantees floor & caps ceiling) |
| **Provisioned Concurrency**| **Zero cold starts** ($< 10\text{ms}$) | High (Billed for idle warm memory) | High (Guarantees pre-warmed capacity) |

## 18. Interview Questions
1. *An e-commerce company's checkout API experiences intermittent HTTP 429 'Too Many Requests' errors during flash sales, despite aggregate CPU on backend databases remaining below 20%. Walk me through your forensic investigation and architectural fix.*
2. *Explain the mathematical relationship between execution duration and concurrency. Why does a slow database query trigger serverless throttling even if request volume remains constant?*
3. *What is AWS Lambda SnapStart, how does it mitigate cold starts in Java runtimes, and what uniqueness constraints must application code observe when using it?*

## 19. Interview Answer
**Exemplary Answer to Question 1**:
> "Intermittent HTTP 429 errors during flash sales with low database CPU utilization indicates that the system is hitting **Serverless Concurrency Throttling**, not downstream database saturation:
>
> 1. **Forensic Triage Protocol**:
>    - I immediately check CloudWatch metrics for the regional account: specifically `ConcurrentExecutions` and `Throttles`.
>    - I check whether throttling is occurring at the **Function Level** (the checkout function reached its configured `Reserved Concurrency` ceiling) or at the **Account Level** (aggregate account executions hit the regional 1,000 concurrency limit).
>    - In most production incidents, the culprit is a **Noisy Neighbor**: an unthrottled asynchronous event function (e.g., an order confirmation email worker or an image thumbnail generator triggered by S3) experienced a surge, scaling to 900 concurrent executions and consuming the entire account's unreserved pool.
>
> 2. **The 3-Step Architectural Fix**:
>    - **Step 1: Isolate the Checkout Function via Reserved Concurrency**:
>      I assign an explicit **Reserved Concurrency** pool (e.g., 500) to the checkout API. This mathematically guarantees that 500 execution slots are permanently reserved for checkout and can never be stolen by other functions.
>    - **Step 2: Eliminate Cold Starts with Provisioned Concurrency**:
>      Flash sales deliver instantaneous traffic spikes that outpace standard scaling. I configure **Provisioned Concurrency = 200** on the checkout function. This keeps 200 execution environments fully warm with pre-initialized database connection pools, delivering consistent $< 15\text{ms}$ latency without cold starts.
>    - **Step 3: Cap Asynchronous Background Workers**:
>      I set a strict Reserved Concurrency ceiling of **50** on the background email and thumbnail functions. This ensures background queues throttle themselves gracefully without impacting the customer checkout flow."

## 20. Hands-on Exercise
**Objective**: Measure the exact latency difference between a Cold Start and a Warm Start on AWS Lambda or OCI Functions.

### Verification Steps
1. Deploy a Python serverless function that connects to an external HTTPS API during global initialization.
2. In the AWS CLI, execute a cold start invocation:
   ```bash
   aws lambda invoke --function-name test-func --log-type Tail response.json \
     | jq -r .LogResult | base64 --decode
   ```
3. Inspect the `REPORT` line:
   ```text
   Init Duration: 350.21 ms    Duration: 45.12 ms    Billed Duration: 396 ms
   ```
4. Immediately invoke the function a second time:
   ```text
   Duration: 8.45 ms    Billed Duration: 9 ms    (Init Duration is absent!)
   ```
5. Observe that the warm invocation executes in **8.45ms** compared to **396ms** for the cold start, confirming that warm container reuse slashes execution latency by 98%.
