# Kubernetes Security Hardening & Runtime Threat Detection Lab

![AWS](https://img.shields.io/badge/AWS-EC2-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue)
![Falco](https://img.shields.io/badge/Runtime_Security-Falco_0.43.1-green)
![Trivy](https://img.shields.io/badge/Image_Scanning-Trivy-red)
![CIS](https://img.shields.io/badge/Benchmark-CIS_Kubernetes-yellow)
![Date](https://img.shields.io/badge/Completed-May_2026-lightgrey)

## Project Overview

A hands-on cloud security lab demonstrating Kubernetes security hardening, runtime threat detection, and container vulnerability management on AWS EC2. This project replicates real-world cloud security workflows used by Cloud Security Engineers and SOC Analysts protecting containerised infrastructure.

## Key Results

| Security Control | Finding |
|---|---|
| Falco Runtime Detection | 4 real alerts captured - 2 shell spawns in container + 2 sensitive file reads on /etc/shadow |
| Container Image CVE Reduction | HIGH+CRITICAL CVEs dropped from 113 to 20 (82% reduction) updating nginx:1.14 to nginx:latest |
| CRITICAL CVEs Specifically | Reduced from 31 to 2 (93.5% reduction) between image versions |
| CIS Benchmark Audit | 7 PASS / 61 FAIL / 56 WARN across 124 checks - all FAIL items documented |
| RBAC Enforcement | developer-user blocked from delete/create, read-only access confirmed |
| Network Policy | Zero-trust default-deny-all applied - selective allow for port 80 only |

## Security Controls Implemented

| Control | Tool | Purpose |
|---|---|---|
| Access Control | Kubernetes RBAC | Least-privilege role enforcement |
| Network Segmentation | NetworkPolicy | Zero-trust pod isolation |
| Runtime Threat Detection | Falco 0.43.1 | Real-time container activity monitoring |
| Image Vulnerability Scanning | Trivy | CVE detection before deployment |
| CIS Benchmark Audit | kube-bench 0.8.0 | Cluster compliance validation |
| Secret Management | Kubernetes Secrets | Credential isolation from application code |

## Architecture

```
AWS EC2 (t2.medium - Ubuntu 22.04 LTS - us-east-1)
└── K3s Kubernetes Cluster
    ├── Namespace: production
    │   ├── nginx-app Deployment (nginx:latest)
    │   ├── RBAC: developer-readonly Role + RoleBinding
    │   ├── NetworkPolicy: default-deny-all (zero-trust baseline)
    │   ├── NetworkPolicy: allow-nginx-ingress (port 80 only)
    │   └── Secret: db-credentials
    ├── Namespace: monitoring
    └── Namespace: security-testing
```

## Falco Runtime Security Alerts (Real Output - May 11, 2026)

Falco detected the following security events during simulated attack scenarios:

```
07:32:01 NOTICE  Shell spawned in container with attached terminal
                 container=nginx | pod=nginx-app-5d665df68-qzlwr | ns=production
                 user=root | command=bash | parent=runc

07:33:03 WARNING Sensitive file opened for reading by non-trusted program
                 file=/etc/shadow | container=nginx | ns=production
                 user=root | command=cat /etc/shadow

07:36:59 WARNING Sensitive file opened for reading by non-trusted program
                 file=/etc/shadow | container=nginx | ns=production
                 user=root | command=cat /etc/shadow

07:37:10 NOTICE  Shell spawned in container with attached terminal
                 container=nginx | pod=nginx-app-5d665df68-qzlwr | ns=production
                 user=root | command=bash | parent=containerd-shim
```

**SOC Relevance:** Shell spawns inside running containers are a critical IOC indicating potential container escape or attacker-controlled code execution. /etc/shadow access indicates credential harvesting. Both are Tier-1 SOC escalation triggers in real environments.

## Container Image Vulnerability Comparison (Trivy)

| Severity | nginx:1.14 (outdated) | nginx:latest | Change |
|---|---|---|---|
| CRITICAL | 31 | 2 | 93.5% reduction |
| HIGH | 82 | 18 | 78.0% reduction |
| MEDIUM | 54 | 89 | Increased (new CVEs disclosed) |
| LOW | 43 | 117 | Increased (new CVEs disclosed) |
| UNKNOWN | 7 | 8 | - |
| **HIGH+CRITICAL Total** | **113** | **20** | **82.3% reduction** |

**Finding:** Updating from nginx:1.14 to nginx:latest reduces HIGH and CRITICAL CVE exposure by 82%. The increase in MEDIUM/LOW is expected - new CVEs have been disclosed since 1.14 was released. Prioritising HIGH+CRITICAL remediation is standard vulnerability management practice.

## CIS Kubernetes Benchmark (kube-bench 0.8.0)

| Result | Count |
|---|---|
| PASS | 7 |
| FAIL | 61 |
| WARN | 56 |
| **Total Checks** | **124** |

Notable FAIL categories: anonymous authentication controls, audit logging configuration, pod security admission settings. Full remediation notes in LAB_REPORT.md.

## RBAC Validation

| Test | User | Expected | Result |
|---|---|---|---|
| delete pods | developer-user | Denied | no |
| create deployments | developer-user | Denied | no |
| list pods | developer-user | Allowed | yes |
| get services | developer-user | Allowed | yes |

## Tools Used

- **Platform:** AWS EC2 t2.medium, Ubuntu 22.04 LTS
- **Kubernetes:** K3s (lightweight certified Kubernetes)
- **Runtime Security:** Falco 0.43.1 with modern eBPF probe
- **Image Scanning:** Trivy (Aqua Security)
- **Benchmark:** kube-bench 0.8.0
- **Frameworks:** CIS Kubernetes Benchmark, Kubernetes RBAC, NetworkPolicy, ISO 27001

## Repository Structure

```
k8s-security-lab/
├── README.md
├── LAB_REPORT.md
├── TROUBLESHOOTING.md
├── manifests/
│   ├── rbac/
│   │   ├── readonly-role.yaml
│   │   └── readonly-rolebinding.yaml
│   ├── network-policies/
│   │   ├── default-deny-all.yaml
│   │   └── allow-nginx-ingress.yaml
│   └── secrets/
│       └── secret-example.yaml
├── scan-results/
│   ├── nginx-scan.txt
│   ├── nginx-old-scan.txt
│   └── kube-bench-summary.txt
└── screenshots/
```

## Author

**Muhammad Rumman Aqeel**
- GitHub: [github.com/rummanaqeel](https://github.com/rummanaqeel)
- LinkedIn: [linkedin.com/in/rumman-aqeel-336758376](https://linkedin.com/in/rumman-aqeel-336758376)
- Email: rummanaqeel8@gmail.com
