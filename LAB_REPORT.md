# Lab Report - Kubernetes Security Hardening & Runtime Threat Detection

**Author:** Muhammad Rumman Aqeel
**Date Completed:** May 11, 2026
**Duration:** Approximately 6 hours
**Environment:** AWS EC2 t2.medium, Ubuntu 22.04 LTS, K3s Kubernetes, us-east-1
**Falco Version:** 0.43.1
**kube-bench Version:** 0.8.0

---

## Executive Summary

This lab demonstrates end-to-end security hardening of a Kubernetes cluster deployed on AWS EC2. Security controls were implemented across six domains: access control (RBAC), network segmentation (NetworkPolicy), runtime threat detection (Falco), container image vulnerability scanning (Trivy), secrets management, and CIS Benchmark compliance auditing (kube-bench).

Key outcomes: Falco successfully detected 4 real security events during simulated attack scenarios. Container image scanning showed an 82% reduction in HIGH+CRITICAL CVEs by updating the base image. CIS Benchmark audit identified 61 FAIL items with full remediation guidance documented.

---

## Environment Details

| Component | Details |
|---|---|
| Cloud Provider | AWS (us-east-1) |
| Instance Type | t2.medium (2 vCPU, 4GB RAM) |
| Operating System | Ubuntu 22.04 LTS |
| Kubernetes Distribution | K3s |
| Kernel | Linux 6.8.0-1015-aws |
| Falco Version | 0.43.1 (eBPF probe) |
| kube-bench Version | 0.8.0 |
| Trivy Version | 0.70.x |

---

## Phase 1 - Cluster Deployment

### Actions Taken
- Launched t2.medium EC2 instance with Ubuntu 22.04 LTS in us-east-1
- Configured security group with least-privilege inbound rules (SSH port 22, K8s API port 6443, NodePort range 30000-32767 - all restricted to my IP only)
- Installed K3s lightweight Kubernetes distribution
- Created 3 namespaces: production, monitoring, security-testing
- Deployed nginx:latest to production namespace and exposed via NodePort

### Result
- Cluster deployed successfully, single node status: Ready
- nginx application accessible via EC2 public IP and NodePort
- All 3 namespaces created successfully

### Screenshots
- 01-aws-console-home.png - AWS console home
- 02-ec2-dashboard-empty.png - EC2 dashboard before launch
- 03-keypair-created.png - Key pair created
- 04-instance-launched.png - Instance launch success
- 05-ec2-running-instance.png - Instance running with 2/2 status checks
- 06-terminal-showing-success.png - SSH connection confirmed
- 07-k8s-working.png - kubectl get nodes showing Ready
- 08-k8s-namespaces.png - All namespaces created
- 09-nginx-exposed.png - nginx application accessible via browser

---

## Phase 2 - RBAC Security Hardening

### Security Objective
Implement least-privilege access control for a simulated developer user, restricting access to read-only operations in the production namespace only.

### Actions Taken
- Created developer-readonly Role with get/list/watch permissions on pods, services, deployments
- Created developer-readonly-binding RoleBinding assigning role to developer-user
- Validated enforcement using kubectl auth can-i

### Findings

| Permission Test | User | Expected | Result | Status |
|---|---|---|---|---|
| delete pods | developer-user | Denied | no | PASS |
| create deployments | developer-user | Denied | no | PASS |
| list pods | developer-user | Allowed | yes | PASS |
| get services | developer-user | Allowed | yes | PASS |

### Security Impact
- Developer access scoped to read-only operations only
- No ability to modify, delete, or create resources in production namespace
- Separation of duties enforced at cluster level
- Aligned with principle of least privilege (ISO 27001 A.9.2)

### Screenshots
- 10-rbac-config-applied.png - Roles and RoleBindings applied
- 11-can-i-commands.png - Both auth can-i tests showing correct results

---

## Phase 3 - Network Policy Implementation

### Security Objective
Implement zero-trust network segmentation between pods, blocking all traffic by default and selectively allowing only required paths.

### Actions Taken
- Applied default-deny-all NetworkPolicy blocking all ingress and egress in production namespace
- Applied allow-nginx-ingress NetworkPolicy selectively allowing TCP port 80 to nginx pods only
- Verified both policies active with kubectl get networkpolicies

### Findings

| Policy | Effect | Status |
|---|---|---|
| default-deny-all | Blocks all pod-to-pod traffic by default | Applied |
| allow-nginx-ingress | Allows TCP:80 to nginx-app pods only | Applied |

### Security Impact
- Zero-trust baseline established
- Compromised pod cannot communicate laterally to other pods
- Only explicitly permitted traffic flows allowed
- Aligns with PCI DSS Requirement 1 network segmentation

### Screenshot
- 12-k8s-network-policy.png - Both policies active

---

## Phase 4 - Secrets Management

### Security Objective
Demonstrate secure credential storage and identify the risk of base64 encoding vs encryption.

### Actions Taken
- Created db-credentials Secret storing simulated database credentials
- Inspected Secret to demonstrate values are base64 encoded (not encrypted at rest by default)

### Findings

| Finding | Severity | Detail |
|---|---|---|
| Secrets base64 encoded not encrypted | MEDIUM | Default K8s secrets are not encrypted at rest |
| Values accessible to cluster admins | MEDIUM | Anyone with kubectl + permissions can decode |

### Remediation Recommendations
- Enable Kubernetes encryption at rest via EncryptionConfiguration
- Use external secrets management (AWS Secrets Manager, HashiCorp Vault)
- Restrict Secret access via RBAC to only required service accounts

### Screenshot
- 13-k8s-secrets.png - Secret created, values shown as hidden fields

---

## Phase 5 - Falco Runtime Security

### Security Objective
Deploy runtime threat detection to identify malicious activity inside running containers in real time.

### Environment
- Falco 0.43.1 with modern eBPF probe
- Container runtimes monitored: containerd (K3s), Docker, CRI-O, Podman

### Threat Scenarios Executed and Alerts Captured

**Alert 1 - 07:32:01 - Shell Spawned in Container (NOTICE)**
- Action: Executed interactive bash shell inside running nginx container
- Falco Rule Triggered: A shell was spawned in a container with an attached terminal
- Container: nginx | Pod: nginx-app-5d665df68-qzlwr | Namespace: production
- User: root | Parent process: runc

**Alert 2 - 07:33:03 - Sensitive File Read (WARNING)**
- Action: Attempted to read /etc/shadow from inside container
- Falco Rule Triggered: Sensitive file opened for reading by non-trusted program
- File: /etc/shadow | Command: cat /etc/shadow | Container: nginx

**Alert 3 - 07:36:59 - Sensitive File Read (WARNING)**
- Action: Second attempt to read /etc/shadow
- Same rule triggered, confirming consistent detection across repeated events

**Alert 4 - 07:37:10 - Shell Spawned in Container (NOTICE)**
- Action: Second shell spawn from different parent process (containerd-shim)
- Falco detected same threat pattern from different execution path

### Security Impact
- Real-time detection confirmed for both attack scenarios
- All 4 events captured with full context: container name, pod name, namespace, user, command, parent process
- Alert pipeline demonstrated replicating SOC Tier-1 monitoring workflow
- Detection confirmed across different parent process paths (runc and containerd-shim)

### Screenshots
- 14-falco-running.png - Falco service active (systemctl status)
- 15-falco-triggers.png - Live Falco alerts firing in real time
- 16-falco-alerts-cat.png - Saved alerts log showing all 4 events

---

## Phase 6 - Container Image Vulnerability Scanning (Trivy)

### Security Objective
Identify and compare CVEs between an outdated and current nginx base image, demonstrating the impact of image currency on security posture.

### Results

**nginx:1.14 (debian 9.8) - Outdated Image**
| Severity | Count |
|---|---|
| CRITICAL | 31 |
| HIGH | 82 |
| MEDIUM | 54 |
| LOW | 43 |
| UNKNOWN | 7 |
| **Total** | **217** |

**nginx:latest (debian 13.4) - Current Image**
| Severity | Count |
|---|---|
| CRITICAL | 2 |
| HIGH | 18 |
| MEDIUM | 89 |
| LOW | 117 |
| UNKNOWN | 8 |
| **Total** | **234** |

**Comparison - HIGH+CRITICAL Risk Reduction**
| Metric | nginx:1.14 | nginx:latest | Change |
|---|---|---|---|
| CRITICAL | 31 | 2 | 93.5% reduction |
| HIGH | 82 | 18 | 78.0% reduction |
| HIGH+CRITICAL Combined | 113 | 20 | 82.3% reduction |

### Notable Critical CVEs in nginx:1.14 (no longer present in latest)
- CVE-2026-33845: GnuTLS Denial of Service via DTLS zero-length fragment (CRITICAL)
- CVE-2026-7598: libssh2 integer overflow via large username/password (CRITICAL)
- Multiple HIGH CVEs in libssh2, libexpat, libgcrypt, libnghttp2

### Analysis
The increase in MEDIUM and LOW CVEs in nginx:latest is expected behaviour. New CVEs have been publicly disclosed against these libraries since nginx:1.14 was released. The critical finding is the dramatic reduction in HIGH and CRITICAL severity vulnerabilities - from 113 down to 20. Vulnerability management best practice focuses on prioritising HIGH and CRITICAL remediation. Image currency is one of the lowest-cost, highest-impact security controls available.

### Screenshots
- 17-k8s-vulnerability-scans.png - Trivy scan results for nginx:latest
- 18-k8s-vulnerability-scans2.png - Trivy scan results for nginx:1.14
- 19-k8s-old-vs-new-comparison.png - Side by side comparison showing reduction

---

## Phase 7 - CIS Kubernetes Benchmark (kube-bench 0.8.0)

### Security Objective
Audit the cluster configuration against CIS Kubernetes Benchmark controls to identify misconfigurations.

### Results

| Category | PASS | FAIL | WARN |
|---|---|---|---|
| Control Plane + Node + Policies | 7 | 61 | 56 |
| **Total (124 checks)** | **7** | **61** | **56** |

### Notable FAIL Categories

**Authentication Controls**
- Anonymous authentication not disabled on API server
- Insecure port configuration findings
- Recommendation: Set --anonymous-auth=false on kube-apiserver

**Audit Logging**
- Audit logging not enabled by default on K3s
- Recommendation: Configure --audit-log-path and --audit-policy-file

**Pod Security**
- Pod Security Admission controller not configured
- Recommendation: Enable PodSecurity admission plugin with appropriate profiles

**Note on FAIL Count Context**
61 FAIL items is expected on a default K3s single-node lab cluster. K3s is a lightweight distribution that disables many controls by default. In a production environment these would be addressed through a hardened K3s configuration or a full Kubernetes installation with security profiles applied.

### Screenshots
- 20-k8s-bench-mark-v1.png - kube-bench output page 1
- 21-k8s-bench-mark-v2.png - kube-bench output page 2

---

## Overall Security Posture Summary

| Domain | Before Hardening | After Hardening |
|---|---|---|
| Access Control | No RBAC enforced | Least-privilege roles applied and validated |
| Network | All pod traffic allowed | Default deny, selective allow on port 80 |
| Runtime | No monitoring | Falco detecting threats in real time |
| Images | Unscanned outdated image | CVEs identified, 82% HIGH+CRITICAL reduction documented |
| Cluster Config | Default settings | CIS Benchmark audit completed, 124 checks evaluated |
| Secrets | Credentials in plaintext | Kubernetes Secrets with documented encoding risk |

---

## Skills Demonstrated

- Kubernetes cluster deployment and administration (K3s on AWS EC2)
- RBAC design aligned with least-privilege principles (ISO 27001 A.9.2)
- Zero-trust network segmentation using NetworkPolicy
- Runtime threat detection and SOC alert analysis (Falco)
- Container CVE scanning and vulnerability prioritisation (Trivy)
- CIS Kubernetes Benchmark compliance auditing
- Cloud security documentation and findings reporting

---

## References

- CIS Kubernetes Benchmark: https://www.cisecurity.org/benchmark/kubernetes
- Falco Documentation: https://falco.org/docs/
- Trivy Documentation: https://aquasecurity.github.io/trivy/
- kube-bench: https://github.com/aquasecurity/kube-bench
- Kubernetes RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- NIST Container Security Guide: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf
