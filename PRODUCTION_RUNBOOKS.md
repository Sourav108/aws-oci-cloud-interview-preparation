# Production Emergency Operational Runbooks Catalog

> **Operational Maturity Axiom**: *When an outage occurs at 3:00 AM, engineers do not rise to the level of their intellect; they fall to the level of their runbooks. Every runbook below follows a standardized, hypothesis-driven diagnostic and mitigation protocol.*

---

## 📑 Runbook Index

1. [RB-01: Compute Instance Unreachable (EC2 / OCI Compute)](#rb-01-compute-instance-unreachable)
2. [RB-02: High CPU / Memory Saturation & OOM Throttling](#rb-02-high-cpu--memory-saturation)
3. [RB-03: Inter-Service Network Timeout & Packet Drop](#rb-03-inter-service-network-timeout)
4. [RB-04: DNS Resolution Failure & NXDOMAIN Surge](#rb-04-dns-resolution-failure)
5. [RB-05: Unhealthy Load Balancer Targets & 502/504 Surge](#rb-05-unhealthy-load-balancer-targets)
6. [RB-06: Database Connection Pool Exhaustion](#rb-06-database-connection-pool-exhaustion)
7. [RB-07: Asynchronous Message Queue Backlog Surge](#rb-07-asynchronous-message-queue-backlog-surge)
8. [RB-08: Serverless Function Throttling & Concurrency Limits](#rb-08-serverless-function-throttling)
9. [RB-09: Kubernetes Pod CrashLoopBackOff & Service Unreachable](#rb-09-kubernetes-pod-crashloopbackoff)
10. [RB-10: Expired TLS/SSL Certificate Outage](#rb-10-expired-tlsssl-certificate-outage)
11. [RB-11: IAM Authorization Failure & Access Denied Cascade](#rb-11-iam-authorization-failure)
12. [RB-12: Object Storage 403 Forbidden / 503 SlowDown](#rb-12-object-storage-access-failure)
13. [RB-13: Production Deployment Emergency Rollback](#rb-13-production-deployment-emergency-rollback)
14. [RB-14: Cloud Region Partial or Complete Impairment](#rb-14-cloud-region-impairment)
15. [RB-15: Database Replication Lag Spike](#rb-15-database-replication-lag-spike)
16. [RB-16: Unexpected Cloud Billing & Cost Spike](#rb-16-unexpected-cloud-cost-spike)

---

## RB-01: Compute Instance Unreachable

### Symptoms
Instance fails health checks; SSH/SSM times out; load balancer marks instance dead; ping packets dropped.

### Immediate Checks
- Check instance state in AWS Console (`ec2 describe-instance-status`) or OCI Console (`oci compute instance get`).
- Verify System Status Checks (hypervisor health) vs. Instance Status Checks (OS/kernel health).

### Metrics & Telemetry
- CloudWatch: `StatusCheckFailed_System`, `StatusCheckFailed_Instance`, `CPUUtilization`.
- OCI Monitoring: `CpuUtilization`, `InstanceStatus`, `MemoryUtilization`.

### Network & Security Checks
- Inspect Security Group / NSG inbound rules: Is port 22/443 open from expected source CIDR?
- Check Route Table: Is `0.0.0.0/0` routed to IGW (public) or NAT Gateway (private)?
- Check Subnet NACL / Security List for explicit DENY rules.

### Root Cause Candidates
1. OS kernel panic or OOM killer freeze.
2. Local firewall (`iptables` / `ufw` / Windows Firewall) blocking traffic.
3. Hypervisor hardware degradation.
4. Route table detached or misconfigured default gateway.

### Mitigation
- If `StatusCheckFailed_System`: Stop and Start instance to migrate VM to a new physical host.
- Connect via AWS Systems Manager (SSM) Session Manager or OCI Console Connection / Cloud Shell.

### Permanent Fix & Prevention
- Bake health-check agents into base AMI / custom image.
- Configure auto-recovery alarms in CloudWatch / OCI Monitoring.

---

## RB-02: High CPU / Memory Saturation

### Symptoms
Application latency increases; HTTP 504 timeouts; system becomes unresponsive; Linux OOM killer terminates Java/Node processes.

### Immediate Checks
- SSH/SSM into instance: run `top -c`, `htop`, `vmstat 1`, `free -m`.
- Identify top consuming PIDs: `ps aux --sort=-%cpu | head -10` or `ps aux --sort=-%mem | head -10`.

### Metrics & Logs
- Check application garbage collection logs (`gc.log`): Is full GC stalling the JVM?
- Review `/var/log/messages` or `dmesg -T` for `Out of memory: Kill process` entries.

### Root Cause Candidates
1. Traffic spike exceeding provisioned capacity.
2. Software memory leak in application runtime.
3. Unindexed database query causing high CPU loop in client data transformation.

### Mitigation
- Temporarily increase Auto Scaling Group desired capacity or add nodes to OCI Instance Pool.
- Restart leaking application service to recover memory immediately.

### Permanent Fix & Prevention
- Tune JVM memory flags (`-Xmx`, `-Xms`) to leave 25% host RAM for OS and buffers.
- Implement target-tracking autoscaling at 60% average CPU utilization.

---

## RB-03: Inter-Service Network Timeout

### Symptoms
Service A reports `java.net.SocketTimeoutException` or `i/o timeout` when calling Service B in another subnet/VPC.

### Immediate Checks
- Run `curl -v --connect-timeout 2 http://<internal-ip>:<port>/health`.
- Test port reachability: `nc -zv -w 2 <internal-ip> <port>`.

### Network & Security Checks
- Trace the route: `traceroute -n -T -p <port> <internal-ip>`.
- Verify VPC Peering / Transit Gateway / OCI DRG route tables: Does a return route exist?
- Inspect VPC Flow Logs / VCN Flow Logs: Filter by `REJECT` or `RE_REJECT`.

### Root Cause Candidates
1. Missing or asymmetric route in Route Table.
2. Security Group / NSG missing ingress allow from caller's security group ID.
3. Ephemeral port exhaustion on the caller host (`net.ipv4.ip_local_port_range`).

### Mitigation
- Add temporary allow rule in Security Group / NSG for caller's CIDR block.
- Adjust connection pool settings on Service A to reuse HTTP persistent connections (Keep-Alive).

---

## RB-04: DNS Resolution Failure

### Symptoms
Applications log `UnknownHostException` or `dial tcp: lookup failed: no such host`; cross-service RPCs fail abruptly.

### Immediate Checks
- Test resolution from instance: `dig +trace +nodnssec api.internal.domain` and `nslookup api.internal.domain`.
- Query the cloud-provided DNS server (`169.254.169.253` or VPC base $+2$ address): `dig @169.254.169.253 api.internal.domain`.

### Metrics & Telemetry
- Check Route 53 Resolver query logs.
- Monitor EC2 DNS quota: 1024 packets per second (PPS) per network interface `[Doc: AWS EC2 Quotas]`.

### Root Cause Candidates
1. EC2 ENI hitting the 1024 PPS DNS limit due to unbuffered, zero-TTL lookups.
2. Misconfigured `/etc/resolv.conf` or corrupted local `systemd-resolved`.
3. Route 53 Private Hosted Zone not associated with the requesting VPC.

### Mitigation
- Flush local DNS cache: `systemd-resolve --flush-caches`.
- Associate Private Hosted Zone with the target VPC via AWS CLI / Terraform.

### Permanent Fix & Prevention
- Deploy NodeLocal DNSCache in Kubernetes (EKS/OKE) to absorb high-frequency lookups.
- Implement client-side DNS caching in JVM (`networkaddress.cache.ttl=60`).

---

## RB-05: Unhealthy Load Balancer Targets

### Symptoms
Load balancer returns HTTP 502 Bad Gateway or 503 Service Unavailable; Target Group health dashboard turns red.

### Immediate Checks
- Query target health: `aws elbv2 describe-target-health` or `oci lb backend-health get`.
- Inspect health check error reason: `Target.FailedHealthChecks`, `Target.Timeout`, `Target.ResponseCodeMismatch`.

### Metrics & Logs
- ALB metrics: `HTTPCode_Target_5XX_Count`, `UnHealthyHostCount`, `TargetResponseTime`.
- Application access logs: Verify if `/health` requests are receiving responses or timing out.

### Root Cause Candidates
1. Application crashed or listening on a different port than configured in the target group.
2. Health check endpoint performs deep database validation that is failing or timing out.
3. Backend connection backlog is full (`somaxconn` / backlog parameter exhausted).

### Mitigation
- Temporarily relax health check path to a static endpoint that verifies process readiness.
- Scale up the backend target group instances to dilute connection load.

### Permanent Fix & Prevention
- Ensure health check endpoints are lightweight and separated from heavy external dependencies.
- Calibrate health check interval (e.g., 10s interval, 5s timeout, 2 healthy / 3 unhealthy thresholds).

---

## RB-06: Database Connection Pool Exhaustion

### Symptoms
Application logs `SQLException: Connection pool exhausted` or `Too many connections`; database CPU surges; p99 latency spikes.

### Immediate Checks
- Query active database connections:
  `SELECT count(*), state FROM pg_stat_activity GROUP BY state;`
- Identify long-running idle or locked queries:
  `SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE state != 'idle' ORDER BY 2 DESC;`

### Metrics & Logs
- CloudWatch: `DatabaseConnections`, `CPUUtilization`, `FreeableMemory`.
- OCI Database: `CurrentConnections`, `CpuUtilization`.

### Root Cause Candidates
1. Connection pool leakage: Applications open connections without releasing them in `finally` blocks.
2. Unindexed slow query locking tables and queuing up hundreds of subsequent connections.
3. Autoscaling event launched 100 new app instances, each requesting 50 connections ($100 \times 50 = 5000$ connections).

### Mitigation
- Terminate hanging queries: `SELECT pg_terminate_backend(<pid>);`.
- Temporarily increase `max_connections` parameter if memory permits.

### Permanent Fix & Prevention
- Deploy **AWS RDS Proxy** or **PgBouncer** between application tier and database.
- Enforce conservative connection pool sizing: $\text{Pool Size} = (\text{Core Count} \times 2) + \text{Effective Spindle Count}$.

---

## RB-07: Asynchronous Message Queue Backlog Surge

### Symptoms
Consumer lag spikes; processing delays exceed SLA; SQS `ApproximateAgeOfOldestMessage` increases exponentially.

### Immediate Checks
- Query queue attributes: `aws sqs get-queue-attributes --attribute-names All` or `oci queue queue get`.
- Check Dead-Letter Queue (DLQ) depth: Are messages failing processing repeatedly?

### Metrics & Telemetry
- SQS: `ApproximateNumberOfMessagesVisible`, `ApproximateAgeOfOldestMessage`.
- Consumer metrics: CPU, memory, error rates, and consumer thread counts.

### Root Cause Candidates
1. Poison pill message causing consumer crashes in infinite retry loop.
2. Downstream write database throttling consumer batch insertions.
3. Consumer fleet failed to autoscale due to max capacity limit or scaling policy cooldown.

### Mitigation
- Purge or divert poison messages to DLQ with reduced maximum receive count (`maxReceiveCount: 3`).
- Manually bump consumer worker ASG / instance pool to maximum scale.

### Permanent Fix & Prevention
- Configure target-tracking autoscaling directly against `ApproximateNumberOfMessagesVisible`.
- Implement robust DLQ redrive workflows and consumer-side circuit breakers.

---

## RB-08: Serverless Function Throttling

### Symptoms
API Gateway returns HTTP 429 Too Many Requests; Lambda logs `ThrottlingException` or `Rate Exceeded`.

### Immediate Checks
- Inspect account concurrency limit vs. function reserved concurrency.
- Run `aws lambda get-account-settings` to check regional concurrent executions quota.

### Metrics & Logs
- CloudWatch: `Throttles`, `ConcurrentExecutions`, `Duration`, `Errors`.
- OCI Monitoring: `ThrottledExecutions` for OCI Functions.

### Root Cause Candidates
1. Spiky burst traffic exceeding regional unreserved concurrency pool.
2. Downstream database bottleneck causing functions to run slower and hold concurrency slots longer.
3. A single misbehaving background function consuming the entire shared account concurrency quota.

### Mitigation
- Increase Provisioned Concurrency for latency-critical API functions.
- Set Reserved Concurrency on the spiking function to isolate its blast radius.

### Permanent Fix & Prevention
- Request account-level concurrency quota increase from AWS/Oracle support.
- Decouple traffic bursts using an intermediary queue (SQS / OCI Queue) buffer.

---

## RB-09: Kubernetes Pod CrashLoopBackOff

### Symptoms
Pods repeatedly restart; deployments degraded; Kubelet events report `Back-off restarting failed container`.

### Immediate Checks
- Inspect pod status: `kubectl get pods -n <namespace> -o wide`.
- Fetch previous crash logs: `kubectl logs <pod-name> -n <namespace> --previous`.
- Describe pod events: `kubectl describe pod <pod-name> -n <namespace>`.

### Root Cause Candidates
1. Application exit with non-zero status due to missing environment variable or secret.
2. Liveness probe failing due to aggressive timeout or slow application bootstrap.
3. Container exceeding memory limit, triggering Linux OOM killer (`OOMKilled: true`).

### Mitigation
- If OOMKilled, temporarily increase container memory limit in deployment manifest.
- If liveness probe failure, increase `initialDelaySeconds` and `timeoutSeconds`.

### Permanent Fix & Prevention
- Implement graceful startup routines and warm-up handlers.
- Tune memory requests and limits based on observed $p99$ profiling data.

---

## RB-10: Expired TLS/SSL Certificate Outage

### Symptoms
Browsers display security warnings (`SEC_ERROR_EXPIRED_CERTIFICATE`); API clients terminate connections with SSL handshake failure.

### Immediate Checks
- Test certificate validity: `echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null | openssl x509 -noout -dates`.
- Inspect certificate status in AWS ACM or OCI Certificate Service.

### Mitigation
- Request and validate new certificate in AWS ACM or OCI Certificates.
- Bind the new certificate ARN/OCID to the Application Load Balancer or API Gateway listener immediately.

### Permanent Fix & Prevention
- Use cloud-managed certificates (ACM / OCI Certificates) with **automated DNS-based renewal**.
- Configure CloudWatch / OCI Monitoring alarms triggering 30, 14, and 7 days prior to certificate expiration.

---

## RB-11: IAM Authorization Failure

### Symptoms
Applications log `AccessDenied` or `NotAuthorizedException`; microservices fail to read S3 buckets or retrieve KMS keys.

### Immediate Checks
- Review CloudTrail / OCI Audit logs filtering by `errorCode = AccessDenied`.
- Verify the exact principal ARN/OCID, requested API action, and target resource ARN.

### Root Cause Candidates
1. Workload identity token expired or instance profile detached.
2. S3 Bucket Policy / KMS Key Policy explicitly denies access or lacks principal in allowed list.
3. Organization SCP or OCI Security Zone blocking API calls in specific regions.

### Mitigation
- Update IAM policy to grant explicit minimum required action on the specific resource ARN.

### Permanent Fix & Prevention
- Validate IAM policies with automated CI linting (`cfn-nag`, `tfsec`) and IAM Access Analyzer.
- Use IAM policy variables and resource tags to minimize manual policy modifications.

---

## RB-12: Object Storage Access Failure (403 / 503)

### Symptoms
File uploads or downloads fail; S3 returns `503 SlowDown` or `403 AccessDenied`.

### Immediate Checks
- If 403: Verify bucket ACL, bucket policy, IAM policy, and KMS key access policy.
- If 503: Check request rate per prefix. S3 supports 3,500 PUT and 5,500 GET requests/sec per prefix `[Doc: S3 Performance, checked 2026-09-03]`.

### Root Cause Candidates
1. Prefix throttling: Storing all objects under a single root prefix (e.g., `bucket/data/2026-09-03/`) exceeding RPS quotas.
2. Missing `kms:Decrypt` or `kms:GenerateDataKey` permissions for the caller.

### Mitigation
- Introduce randomized hash prefixes or partitioned prefix hierarchies (e.g., `bucket/<hash>/data/`).
- Enable exponential backoff with jitter on the storage client SDK.

---

## RB-13: Production Deployment Emergency Rollback

### Symptoms
Post-release error rate spikes; release metrics show regression in latency or database errors.

### Immediate Execution Steps
1. **Freeze CI/CD Pipelines**: Halt all pending deployments to avoid rolling over an active incident.
2. **Traffic Redirection (Blue/Green)**:
   - AWS ALB: Switch Target Group weights back to Blue (previous stable version).
   - OCI Load Balancer: Update backend set weights to 100% Blue.
3. **Canary Abort**: If running canary, trigger automated deployment pipeline cancel & rollback hook.
4. **Database Backward Compatibility**: Verify that recent DB migrations did not drop columns or break contract with previous app version.

---

## RB-14: Cloud Region Partial or Complete Impairment

### Symptoms
Multiple AZs report elevated latency or API errors; cloud provider status dashboard confirms regional disruption.

### Emergency DR Failover Execution
1. **Declare Disaster**: Incident Commander officially approves multi-region failover.
2. **Promote Database Replica in Secondary Region**:
   - AWS: Promote Aurora Global Database secondary cluster to standalone read-write primary.
   - OCI: Trigger Data Guard switchover/failover on OCI Base DB or Autonomous Database.
3. **Shift Global DNS Traffic**:
   - Flip Route 53 Failover Routing or OCI DNS Steering Policy to direct 100% of traffic to the secondary region.
4. **Scale Up Secondary Compute**: Adjust secondary ASG / Instance Pool capacity to 100% production load.

---

## RB-15: Database Replication Lag Spike

### Symptoms
Read replicas return stale data; replica lag metrics trend upwards; replication slots consume database disk space.

### Immediate Checks
- PostgreSQL: `SELECT pid, client_addr, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes FROM pg_stat_replication;`
- MySQL: `SHOW SLAVE STATUS\G;` inspect `Seconds_Behind_Master`.

### Root Cause Candidates
1. Long-running DDL or analytical query running on read replica holding locks.
2. Network bandwidth saturation between primary and replica AZs.
3. High write throughput on master overwhelming single-threaded replica apply process.

### Mitigation
- Kill blocking analytical queries on the replica.
- Route time-sensitive read traffic temporarily back to the primary database.

---

## RB-16: Unexpected Cloud Billing & Cost Spike

### Symptoms
Billing alert fires; daily cloud burn rate jumps by 200–500%; cost anomaly detection flags unbudgeted spend.

### Immediate Checks
- Open AWS Cost Anomaly Detection / OCI Cost Analysis.
- Group costs by **Usage Type**, **Service**, and **Tag**.

### Common Culprits
1. Unintended NAT Gateway data processing loop.
2. Abandoned unattached EBS / Block Volumes from terminated test instances.
3. Runaway Lambda / Function recursive execution loop.
4. Uncompressed debug logging ingested into CloudWatch Logs.

### Immediate Mitigation
- Terminate abandoned rogue compute resources or recursive function triggers.
- Enforce AWS Budget actions / OCI Budget alerts to shut down non-essential resources if thresholds breach.
