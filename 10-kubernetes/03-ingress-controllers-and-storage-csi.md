# 03. Cloud Ingress Controllers & Storage CSI Drivers

## 1. Problem
Running production workloads on cloud-managed Kubernetes requires bridging native Kubernetes declarative manifests (`Ingress`, `Service`, `PersistentVolumeClaim`) with proprietary cloud infrastructure (Application Load Balancers, Target Groups, Elastic Block Store, and OCI Block Volumes). Naive setups deploy generic in-cluster NGINX ingress proxies that route traffic through NodePort services, adding redundant network hops, obscuring client source IPs, and creating bandwidth bottlenecks. Similarly, misconfigured Container Storage Interface (CSI) drivers cause volumes to hang in `VolumeAttachment` deadlocks during pod rescheduling across Availability Zones.

## 2. Cloud Concept: The Ingress Controller & CSI Driver Interface
```text
KUBERNETES DECLARATIVE CLOUD INTEGRATION:

[Kubernetes Ingress Manifest] ──► [Cloud Ingress Controller] ──► Provisions Cloud ALB / OCI LB
                                  (Reconciles k8s API to Cloud API)

[Kubernetes PVC Manifest]     ──► [Cloud Storage CSI Driver] ──► Dynamically Creates EBS / OCI BV
                                  (Attaches / Formats volume via cloud APIs)
```

- **Cloud Ingress Controller**: A Kubernetes controller watching `Ingress` and `Service` resources. Instead of proxying traffic in software, it calls cloud APIs to automatically provision native cloud load balancers (AWS ALB/NLB or OCI Load Balancer) and binds pod IP addresses directly into cloud backend target groups.
- **Container Storage Interface (CSI)**: An open standard specification enabling Kubernetes to manage the lifecycle of persistent storage (provision, attach, format, mount, snapshot, and resize) across cloud block and file storage systems.

## 3. AWS Implementation
In Amazon EKS:
- **AWS Load Balancer Controller (ALBC)**:
  - Supports two primary ingress modes via annotations:
    - `alb.ingress.kubernetes.io/target-type: instance`: Routes traffic to worker node NodePorts.
    - `alb.ingress.kubernetes.io/target-type: ip`: **The Production Best Practice**. Routes traffic directly from the ALB to individual **Pod IP addresses** (leveraging the AWS VPC CNI), bypassing `kube-proxy` iptables overhead entirely!
  - **TargetGroupBinding CRD**: Allows binding Kubernetes pods directly to existing, manually provisioned AWS target groups.
- **AWS EBS & EFS CSI Drivers**:
  - *EBS CSI Driver*: Dynamically provisions `gp3` or `io2` block volumes for stateful pods (ReadWriteOnce / RWO).
  - *EFS CSI Driver*: Dynamically provisions shared NFS filesystems supporting multi-pod concurrent access (ReadWriteMany / RWX).

## 4. OCI Implementation
In OCI Container Engine for Kubernetes (OKE):
- **OCI Load Balancer Controller**:
  - Native controller that automatically provisions OCI Flexible Load Balancers or Network Load Balancers when a service of `type: LoadBalancer` is deployed.
  - Supports annotations to configure exact flexible bandwidth bounds (e.g., `oci.oraclecloud.com/load-balancer-shape-flex-min: "50"`).
  - Integrates with OCI VCN-Native Pod Networking to route traffic directly to pod VNIC IPs with zero intermediate translation.
- **OCI Block Volume & File Storage (FSS) CSI Drivers**:
  - *OCI Block Volume CSI Driver*: Provisions ultra-high-performance block storage tuned via Volume Performance Units (VPUs) directly from PVC annotations.
  - *OCI FSS CSI Driver*: Automatically creates and attaches OCI File Storage Service NFS exports for RWX storage workloads.

## 5. Production Failure Modes: Multi-Attach & Cross-AZ Lockouts
- **The Multi-Attach Volume Deadlock**: An EBS or standard OCI Block Volume is formatted with ext4 and attached to Node A in `us-east-1a`. When the stateful pod crashes and the Kubernetes scheduler reschedules it onto Node B in `us-east-1b`, the volume attachment hangs. **EBS volumes are strictly zonal** and cannot attach across AZ boundaries! The pod hangs permanently in `ContainerCreating` with events: `Multi-Attach error for volume: volume is already exclusively attached to one node`.
  - *Mitigation*: Enforce `volumeBindingMode: WaitForFirstConsumer` in the StorageClass to ensure the scheduler places the pod in the specific AZ where storage exists.

## 6. Troubleshooting & Diagnostics
1. **Inspect Cloud Ingress Controller Logs**:
   ```bash
   kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100
   ```
   Look for `Failed to reconcile Ingress: WebIdentityErr` (IRSA permission failure) or `SubnetNotFound` (missing subnet discovery tags).
2. **Inspect Persistent Volume Claim Status**:
   ```bash
   kubectl describe pvc <pvc-name>
   ```
   Look for `Waiting for a volume to be created by the CSI driver` or `VolumeAttachment error`.

## 7. Senior Interview Question & Defense
**Question**: *When exposing Kubernetes microservices via an AWS Application Load Balancer, what is the architectural difference between `target-type: instance` and `target-type: ip`? Which do you mandate for production, and why?*

**Staff-Level Defense**:
> "I mandate **`target-type: ip`** for all production microservices behind the AWS Load Balancer Controller:
>
> 1. **`target-type: instance` (Legacy NodePort Routing)**:
>    - The ALB sends traffic to a random EC2 worker node on a high NodePort (e.g., 31240).
>    - When the packet arrives at the node, `kube-proxy` (iptables or IPVS) evaluates cluster NAT rules. If the target pod does not reside on that specific node, the node forwards the packet across the internal VPC network to a *second* worker node where the pod actually lives (SNAT hop).
>    - *Consequences*: Adds an extra, unnecessary network hop (increasing p99 latency), obscures the client's original source IP due to node SNAT, and causes uneven load distribution.
>
> 2. **`target-type: ip` (Direct Pod Routing)**:
>    - Because we run the **AWS VPC CNI**, every pod possesses its own first-class private IPv4 address in the VPC subnet.
>    - The AWS Load Balancer Controller registers these pod IP addresses **directly into the ALB Target Group**.
>    - When an incoming request hits the ALB, the ALB forwards the packet directly across the private hypervisor overlay straight to the specific pod's virtual interface.
>    - *Consequences*: Completely bypasses `kube-proxy` and NodePort translation, eliminates the extra network hop (cutting latency by 1–2ms), preserves true client IPs, and delivers accurate end-to-end health checking directly against the application process."
