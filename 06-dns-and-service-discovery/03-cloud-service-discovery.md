# 03. Cloud Microservice Discovery & Internal Resolution

## 1. Problem
In dynamic microservice architectures running on Kubernetes (EKS / OKE) or container instances (ECS / OCI Container Instances), container pods and tasks scale out, scale in, and restart continuously, receiving new ephemeral private IP addresses on every lifecycle event. If calling services rely on hardcoded IP addresses or static DNS records, communication breaks within minutes. Engineering teams need dynamic, automated service discovery that registers healthy instances, de-registers failing containers, and shields clients from ephemeral infrastructure churn.

## 2. Cloud Concept
Microservice service discovery operates via two primary paradigms:

```text
Server-Side / DNS-Based Discovery:
[Client] ──> Queries DNS: "orders.internal" ──> [Cloud DNS / CoreDNS] ──► Returns Healthy IP List
                                                                               │
[Client] ──> Connects directly to IP ──────────────────────────────────────────┘

Client-Side / Service Mesh Discovery:
[Client] ──> [Envoy Sidecar Proxy] ──(Dynamic Control Plane: xDS / Consul)──► [Target Pod]
```

- **DNS-Based Discovery**: Services query an internal DNS name (e.g., `orders.production.local`). The cloud service discovery registry continuously updates Route 53 Private Hosted Zones or internal CoreDNS records with current healthy container IPs.
- **Client-Side Discovery / Service Mesh**: Services talk to a local sidecar proxy (Envoy in Istio/Linkerd). The control plane streams active pod IP updates directly to the proxy over persistent gRPC connections, bypassing DNS entirely and enabling sub-millisecond dynamic routing.

## 3. AWS Implementation: AWS Cloud Map
In AWS:
- **AWS Cloud Map**: A fully managed application service discovery registry.
- Supports both **API-based discovery** (calling `DiscoverInstances`) and **DNS-based discovery** (automatically managing Route 53 Private Hosted Zone records) `[Doc: AWS Cloud Map Developer Guide, checked 2026-09-03]`.
- Integrates natively with Amazon ECS and EKS. When an ECS task launches, ECS automatically registers the task's private IP with Cloud Map.
- **Custom Attributes & Filtering**: Allows clients to query instances based on custom metadata tags (e.g., request an instance where `version = "2.1"` or `stage = "canary"`).

## 4. OCI & Kubernetes Implementation: CoreDNS & Consul
In OCI:
- **Kubernetes CoreDNS in OKE**: OCI Container Engine for Kubernetes (OKE) utilizes **CoreDNS** running as a clustered deployment inside the `kube-system` namespace.
  - Kubernetes Services automatically receive internal DNS names: `<service-name>.<namespace>.svc.cluster.local`.
  - CoreDNS watches the Kubernetes API server; when a pod fails readiness probes or terminates, its IP is immediately stripped from the CoreDNS endpoint list.
- **OCI Private DNS Integration**: OCI VCN Private Resolvers automatically register hostnames for compute instances and database nodes launched in private subnets.
- **HashiCorp Consul on OCI**: Enterprise architectures frequently deploy Consul clusters across OCI compute shapes to achieve multi-region, multi-cloud service discovery with active health checking.

## 5. Architectural Trade-offs
| Service Discovery Model | Implementation Example | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **DNS-Based Discovery** | AWS Cloud Map, CoreDNS | Universal protocol; zero client-side library dependencies | Subject to client-side DNS caching delays and stale IPs |
| **API-Based Discovery** | Cloud Map API, Consul | Real-time updates; supports custom attribute filtering | Mandates proprietary SDKs; extra HTTP call before connection |
| **Service Mesh (Sidecar)** | Istio, Envoy, Linkerd | Sub-second propagation; rich telemetry & mTLS encryption | Significant CPU/memory overhead; high operational complexity |

## 6. Failure Modes & The JVM Caching Pitfall
- **The Stale IP / Blackhole Pod Loop**: A Kubernetes pod terminates, and its private IP (`10.0.1.42`) is released. If an upstream Java microservice cached the DNS response indefinitely (`networkaddress.cache.ttl = -1`), it continues sending TCP packets to `10.0.1.42`. The packets time out, causing a localized outage.
  - *Mitigation*: Override JVM DNS settings to enforce a 5-second TTL:
    ```ini
    networkaddress.cache.ttl=5
    ```
- **CoreDNS Scaling Exhaustion**: In high-throughput clusters, millions of unbuffered DNS lookups saturate CoreDNS pods, causing DNS timeouts. Deploy **NodeLocal DNSCache** to resolve lookups on the node loopback.

## 7. Senior Interview Question & Defense
**Question**: *When architecting internal microservices communication on Kubernetes (EKS/OKE), when should you use native Kubernetes CoreDNS, when should you use an internal Load Balancer, and when should you adopt a full Service Mesh?*

**Staff-Level Defense**:
> "The choice depends strictly on traffic volume, protocol requirements, and operational maturity:
>
> 1. **Kubernetes CoreDNS (ClusterIP Service)**:
>    - *Best For*: Standard HTTP/REST services with low-to-moderate request rates.
>    - *Mechanism*: Simple, zero-cost abstraction where CoreDNS returns the virtual ClusterIP, and iptables/IPVS on the node routes traffic to pods.
>    - *Limitation*: Fails on **gRPC / HTTP/2** because HTTP/2 multiplexes multiple RPCs over a single persistent TCP connection. Standard CoreDNS/ClusterIP causes all gRPC requests to pool onto a single backend pod, completely breaking load balancing!
>
> 2. **Internal L7 Load Balancer (ALB / OCI LB)**:
>    - *Best For*: High-throughput gRPC routing, centralized TLS termination, and path-based routing between decoupled organizational boundaries.
>    - *Trade-off*: Incurs cloud load balancer hourly and data processing fees, adding a 1–2ms network hop.
>
> 3. **Service Mesh (Istio / Envoy)**:
>    - *Best For*: Enterprise microservice meshes requiring **end-to-end mTLS**, fine-grained canary traffic splitting (e.g., 99% v1, 1% v2), and client-side gRPC load balancing without centralized hardware bottlenecks.
>    - *Trade-off*: Introduces non-trivial CPU/memory sidecar tax (typically 5–15% cluster overhead) and demands high operational SRE maturity."
