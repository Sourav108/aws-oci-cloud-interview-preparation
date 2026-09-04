# Software Supply Chain Security, SBOM & Image Signing (AWS vs. OCI)

---

## 1. Executive Summary & Core Definitions

In modern enterprise cloud engineering, securing the production runtime perimeter is meaningless if the code running inside your containers was tampered with before deployment. High-profile software supply chain attacks (SolarWinds, Codecov, Log4j, XZ Utils) demonstrated that sophisticated adversaries do not attack hardened production firewalls; instead, they compromise third-party open-source dependencies, inject malicious build scripts into CI runners, or poison container registries.

**Software Supply Chain Security** establishes end-to-end cryptographic provenance from source code to production deployment. Every software artifact must be scanned, cataloged in a **Software Bill of Materials (SBOM)**, cryptographically signed with trusted KMS keys, and verified by Kubernetes admission controllers before execution is permitted.

```
       UNPROTECTED SUPPLY CHAIN (Vulnerable to Tampering)
Source Code ---> CI Runner (Injected with Malicious Code) ---> Registry ---> Unverified Deployment

       CRYPTOGRAPHICALLY SECURED SUPPLY CHAIN (Zero Trust)
Git (GPG Signed) ---> CI (Reproducible Build) ---> Generate SBOM (Syft) ---> Sign Image (Cosign / Signer)
                                                                                  |
                                                                                  v
Kubernetes Cluster (EKS / OKE) <--- [ Admission Controller ] <--- Cryptographic Signature Verified!
```

### Core Terminology
* **Software Bill of Materials (SBOM)**: A comprehensive, machine-readable inventory of all software components, direct dependencies, transitive libraries, licenses, and compiler versions that make up an application package (standardized in **SPDX** or **CycloneDX** formats).
* **SLSA (Supply-chain Levels for Software Artifacts)**: A security framework developed by Google and the OpenSSF defining 4 progressive levels of supply chain integrity, focusing on build hermeticity, provenance generation, and tamper resistance [Doc: SLSA Framework, checked 2026].
* **Cosign / Sigstore**: An open-source standard for container signing, verification, and storage in OCI-compliant container registries using KMS or keyless ephemeral certificates.
* **AWS Signer**: A fully managed code-signing service that creates digital signatures for container images in Amazon ECR and AWS Lambda packages [Doc: AWS Signer, checked 2026].
* **OCI Container Image Signing**: Native OCI KMS integration that digitally signs container images pushed to OCI Container Registry (OCIR) and enforces signature verification on OKE via admission controllers [Doc: OCI Image Signing, checked 2026].

---

## 2. Architectural Deep Dive: The SLSA Framework & Cryptographic Provenance

```
[ SLSA LEVEL 1: Basic Build ]
- Build process is automated via CI script.
- Basic provenance document generated.

[ SLSA LEVEL 2: Tamper Resistance ]
- Hosted build service (CodeBuild / OCI DevOps / GitHub Actions).
- Provenance is cryptographically signed by the build service identity.

[ SLSA LEVEL 3: Non-Falsifiable Provenance ]
- Isolated, ephemeral build environments.
- Build environment cannot be accessed or altered by developers during build execution.

[ SLSA LEVEL 4: Hermetic & Reproducible ]
- Hermetic builds: Zero external network access during compilation; all dependencies pinned and vendored.
- Two-party review on all code changes and build configuration files.
```

---

## 3. Side-by-Side Comparison: Container Signing & Attestation

| Capability / Dimension | AWS Supply Chain Security | OCI Supply Chain Security |
| :--- | :--- | :--- |
| **Managed Signing Service**| **AWS Signer** (Native ECR & Lambda signing) | **OCI Image Signing** (Native KMS integration) |
| **Open-Source Standard** | Cosign / Sigstore backed by AWS KMS | Cosign / Sigstore backed by OCI Vault KMS |
| **Vulnerability Scanning** | Amazon Inspector (Continuous CVE scanning) | **OCI Vulnerability Scanning Service (VSS)** |
| **Kubernetes Admission Control**| Kyverno / Gatekeeper / Kritis on EKS | Native **OKE Image Verification Policy** / Kyverno |
| **SBOM Generation Tooling** | AWS Inspector SBOM export / Syft / Trivy | OCI DevOps integrated scanning / Syft |
| **Registry Immutability** | ECR Immutable Image Tags | OCIR Immutable Tag Rules |
| **Artifact Formats** | OCI Artifacts, Docker v2, Cosign signatures | OCI Artifacts, Helm Charts, Container Images |

---

## 4. Implementation & Configuration (Cosign / Dockerfile / Admission Controller)

### Generating SBOM and Signing Container Image (GitHub Actions / Bash)

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE="123456789012.dkr.ecr.us-east-1.amazonaws.com/order-service:v2.1.0"
KMS_KEY_ARN="arn:aws:kms:us-east-1:123456789012:key/abc-123-def"

echo "=== 1. Building Container Image ==="
docker build -t "${IMAGE}" .

echo "=== 2. Generating Software Bill of Materials (SBOM) via Syft ==="
syft "${IMAGE}" -o cyclonedx-json=sbom.json

echo "=== 3. Scanning for Known Vulnerabilities via Trivy ==="
trivy image --exit-code 1 --severity CRITICAL "${IMAGE}"

echo "=== 4. Pushing Image to Container Registry ==="
docker push "${IMAGE}"

echo "=== 5. Attaching SBOM to Registry as OCI Artifact ==="
cosign attach sbom --sbom sbom.json "${IMAGE}"

echo "=== 6. Cryptographically Signing Image with AWS KMS ==="
cosign sign --key "awskms:///${KMS_KEY_ARN}" --yes "${IMAGE}"

echo "SUCCESS: Image signed and provenance verified."
```

---

### OKE Image Verification Policy Enforcement (Kyverno ClusterPolicy)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
  annotations:
    policies.kyverno.io/title: Verify OCI / AWS Container Signatures
    policies.kyverno.io/description: Rejects any pod deployment whose image signature is missing or invalid.
spec:
  validationFailureAction: Enforce # Hard blocking admission controller
  rules:
    - name: verify-signature-rule
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
            - "*.ocir.io/*"
          key: |
            -----BEGIN PUBLIC KEY-----
            MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...public_key...
            -----END PUBLIC KEY-----
```

---

## 5. Failure Modes, Edge Cases & Compromised Dependency Scenarios

```
[ Attack Vector: Dependency Confusion / Typosquatting ]
Developer imports: 'lodash-utils' (Malicious NPM package) instead of 'lodash'
                                 |
                                 v
[ DEFENSIVE SHIELD: SBOM + TRIVY SCAN IN PIPELINE ]
Pipeline extracts dependency graph -> Matches against CVE database -> Identifies malicious package
Build terminates with exit code 1 -> Production cluster protected!
```

### Critical Edge Cases
1. **Dynamic Image Tag Mutability (`:latest`)**:
   * Deploying with mutable tags like `:latest` allows an attacker who compromises registry credentials to overwrite the tag with malicious code without changing Git manifests.
   * *Mandate*: Enforce **Immutable Image Tags** in Amazon ECR and OCI OCIR. Deploy exclusively using cryptographic image digests:
     `image: order-service@sha256:7f8a9b0c...`
2. **KMS Signing Key Deletion / Rotation Outage**:
   * If the KMS key used to sign production images is deleted or rotated without updating admission controllers, all new pod replicas (and autoscaling events) will fail admission validation.
   * *Rule*: Use dedicated, long-lived asymmetric signing keys protected by deletion locks.

---

## 6. Real-World Case Study / Enterprise Supply Chain Breach

* **Context**: Global developer analytics SaaS platform.
* **The Incident**: Attackers compromised the company's Bash uploader script in their CI pipeline (similar to the 2021 Codecov breach).
* **The Attack**: The malicious script silently intercepted developer credentials and modified the compiled production binary right before packaging into a Docker container.
* **The Remediation**:
  * Adopted **SLSA Level 3** build isolation using ephemeral container runners.
  * Implemented automated **SBOM generation** via Syft.
  * Enforced **Cosign Cryptographic Signing** directly inside the isolated build runner.
  * Configured Kubernetes Admission Controllers on EKS/OKE to reject any pod whose SHA-256 hash does not match a verified cryptographic signature signed by the official corporate KMS key.

---

## 7. Interview Defense & Technical Trade-Offs

### Scenario: Implementing Supply Chain Security Without Breaking CI Velocity

* **Interviewer**: "How do you enforce SBOM generation, CVE scanning, and image signing without adding 15 minutes to every developer build?"
* **Staff Candidate Response**:
  1. *Parallelize Pipeline Stages*: Execute SBOM generation and vulnerability scanning asynchronously in parallel with unit test execution rather than sequentially.
  2. *Differential Layer Scanning*: Use scanners like Trivy that cache vulnerability databases locally on CI runners, completing container scans in **under 15 seconds**.
  3. *Shift Verification to the Edge*: Perform cryptographic signing using **AWS Signer** or **OCI Image Signing** automatically upon registry push via event triggers (EventBridge / OCI Events), completely offloading signing compute from developer workstations.
  4. *Enforce at Admission*: Deploy Kyverno or native OKE Image Verification policies. Verification takes **under 10 milliseconds** at the Kubernetes admission webhook layer.
