# Payment Manager for Mojaloop (PM4ML)

## Production Deployment Guide (On-Premise)

This document provides step-by-step guidance for deploying **Payment Manager for Mojaloop (PM4ML)** in an on-premise data center environment using:

* Ubuntu 24.04 LTS
* MicroK8s (Kubernetes)
* Ansible
* GitOps with Argo CD
* HAProxy (External & Internal Load Balancer)
* WireGuard VPN (Optional)

---

# Code Base Reference

This deployment architecture is derived from the official Mojaloop Infrastructure as Code (IaC) modules:

[https://github.com/mojaloop/iac-modules](https://github.com/mojaloop/iac-modules)

The original AWS-focused implementation has been adapted and extended to support on-premise infrastructure using MicroK8s, HAProxy, and GitOps-based workflows.

---
# PM4ML On-Premise Architecture Overview

The following diagram illustrates the logical and network architecture of the PM4ML on-premise deployment.

## Logical Architecture View
<img width="1427" height="906" alt="image" src="https://github.com/user-attachments/assets/4d244515-a0f3-47ef-b878-663da6c894c9" />


## Network Architecture View
<img width="746" height="636" alt="image" src="https://github.com/user-attachments/assets/d48490fd-e95c-4956-9b8d-293f17af03d0" />

---


# 1. Prerequisites

## 1.1 Infrastructure Requirements (Production Recommended)

| Component               | Count | Purpose                                |
| ----------------------- | ----- | -------------------------------------- |
| HAProxy                 | 1–2   | Public & Private ingress load balancer |
| MicroK8s Master Nodes   | 3     | Kubernetes control plane               |
| Worker Nodes            | 3+    | Application workloads                  |
| Storage Server (Ceph)   | 3+    | Cold storage                           |
| Bastion Host (Optional) | 1     | Secure administrative access           |
| NAT Gateway (Optional)  | 1     | Outbound internet for private subnet   |

---

## 1.2 Network Requirements

* Static IP address for each VM
* Internal subnet (e.g., `10.10.0.0/16`)
* Public IP mapped to HAProxy via firewall
* DNS zone (AWS, Cloudflare, or enterprise DNS)

### Required Ports

| Port           | Purpose          |
| -------------- | ---------------- |
| 22             | SSH              |
| 80             | HTTP             |
| 443            | HTTPS            |
| WireGuard Port | VPN (if enabled) |

---

## 1.3 Storage Requirements

* Ceph RBD for persistent storage
* Ceph-backed storage for logs,backups and non-critical workloads

---

# 2. Tools & Versions

| Tool     | Version                  |
| -------- | ------------------------ |
| Ubuntu   | 24.04 LTS                |
| MicroK8s | 1.31/stable              |
| Ansible  | 2.18+                    |
| Helm     | v3                       |
| kubectl  | Matching cluster version |
| Git      | Latest stable            |

---

# 3. VM Provisioning

VMs may be provisioned using:

* VMware
* Proxmox
* OpenStack
* CloudStack
* Bare Metal

---

# 4. Infrastructure Sizing by Environment

## 4.1 Production Environment (PROD)

### Recommended Production Infrastructure Sizing

| Component                 | Quantity   | CPU / RAM / Storage (Per Server)                                                              | Deployment Type | Notes                           |
| ------------------------- | ---------- | --------------------------------------------------------------------------------------------- | --------------- | ------------------------------- |
| Master Node Cluster       | ×3 Servers | 8 CPU Cores, 32 GB RAM, 2 × 300 GB NVMe                                                       | VM              | Dedicated control plane nodes   |
| Worker Node Cluster       | ×3 Servers | 16 CPU Cores, 64 GB RAM, 2 × 500 GB NVMe                                                      | VM              | Application workloads only      |
| Load Balancers (HAProxy)  | ×2 Servers | 4 CPU Cores, 16 GB RAM, 100 GB SSD                                                            | VM              | Active/Active or Active/Standby |
| Cold Data Storage Cluster | ×3 Servers | 16 CPU Cores, 64 GB RAM; OS: 2 × 500 GB SSD (RAID1); Data: 6 × 8 TB HDD; Network: 2 × 10 Gbps | Physical        | Enterprise storage              |

### Cold Storage Purpose

Cold storage devices are intended exclusively for:

* Long-term regulatory record retention
* Backup archives
* Ledger export snapshots
* Log archival storage

Cold storage must not be used for latency-sensitive transaction processing workloads.

---

## 4.2 Staging Environment (STG)

Designed for:

* Functional testing
* Integration testing
* Performance validation
* Pre-production validation

### Recommended Staging Infrastructure Sizing

| Component               | Quantity   | CPU / RAM / Storage (Per Server)   | Deployment Type      | Notes                 |
| ----------------------- | ---------- | ---------------------------------- | -------------------- | --------------------- |
| Master Node Cluster     | ×3 Servers | 4 CPU Cores, 16 GB RAM, 200 GB SSD | VM                   | Control plane         |
| Worker Node Cluster     | ×2 Servers | 8 CPU Cores, 32 GB RAM, 300 GB SSD | VM                   | Application workloads |
| Load Balancer (HAProxy) | ×1 Server  | 2 CPU Cores, 8 GB RAM, 50 GB SSD   | VM                   | Single instance       |
| Cold Storage            | Optional   | Reduced capacity                   | VM or Shared Storage | Optional              |

---

## 4.3 Environment Policy

* Production must use dedicated control plane nodes
* Production must maintain high availability
* Staging may operate with reduced redundancy
* Any deviation from Production requirements must be formally approved

---

## 4.4 Master and Worker on Same Node (Risk Acceptance)

Combining control plane and worker workloads is technically possible but not recommended for production.

Risks:

* Resource contention
* Reduced stability under load
* Increased blast radius
* Difficult capacity planning
* Potential Kubernetes API impact

Production recommendation:

* 3 dedicated master nodes
* Workloads only on worker nodes
* No scheduling on control plane nodes

If combined, formal risk acceptance is required.

---

# 5. HAProxy Configuration (On-Premise)

## 5.1 Overview

HAProxy acts as the ingress load-balancing layer in front of the MicroK8s cluster.

Architecture:

```
Client → Perimeter Firewall → HAProxy → Istio Ingress Gateway → PM4ML Services
```

---

## 5.2 Architecture Components

### Perimeter Firewall

* First security boundary
* Filters inbound traffic
* May perform NAT
* Centralized logging

### HAProxy

* Layer 4 (TCP) load balancing
* Forwards traffic to Kubernetes nodes
* Does not terminate TLS

### Istio Ingress Gateway

* Terminates TLS
* Applies routing policies
* Enforces service mesh security
* Routes traffic to PM4ML

---

## 5.3 HAProxy Interfaces

| Interface | Purpose                        |
| --------- | ------------------------------ |
| Public    | Receives traffic from firewall |
| Private   | Forwards traffic to Kubernetes |

---

## 5.4 TLS Termination Strategy

TLS termination occurs at Istio Ingress Gateway.

Benefits:

* Centralized certificate lifecycle management
* Better integration with service mesh policies
* Improved observability
* Reduced HAProxy complexity

---

## 5.5 High Availability

* Two HAProxy nodes (Production)
* Firewall-level failover or VIP
* Separate physical hosts where possible

---

## 5.6 Security Controls

* Expose only required ports (typically 443)
* Restrict SSH access
* Apply host-level firewall rules
* Integrate logs into monitoring/SIEM

---

## 5.7 Public and Private IP Requirements

### Public IP

* Assigned at firewall layer
* NAT to HAProxy private IP
* Minimum: 1 Public IP
* Recommended: 1 Public VIP

Example:

```
Internet → Firewall → NAT → HAProxy → Istio → Services
```

### Private IP

Required for:

* HAProxy
* Kubernetes Masters
* Kubernetes Workers
* Storage Cluster
* Bastion Host

---

# 6. GitOps Deployment

The customer must provide and maintain their own Git repository for GitOps-based deployment.

Requirements:

- Repository accessible from the cluster (HTTPS or SSH)
- Proper branch or tag strategy defined
- Secure credential/token management
- Access configured in Argo CD

All platform manifests and Helm values must be stored in the customer's repository.

---

## 6.1 Platform Applications Deployment Order

Argo CD applications must be deployed in the following sequence to satisfy dependencies and ensure system stability.

1. Base Utilities

   * `base-utils`

2. Storage Layer

   * `storage-app`

3. Certificate Management

   * `certmanager-helm`
   * `certmanager-app`
   * `certmanager-clusterissuers`

4. Service Mesh

   * `istio-app`
   * `istio-main-app`
   * `istio-gateways-app`

5. DNS Management

   * `external-dns-app`

6. Vault Storage Backend

   * `consul-app`

7. Vault & Secrets Management

   * `vault-app`
   * `vault`
   * `vault-config-operator`
   * `vault-pki-app`

8. Stateful Resources

   * `stateful-resources-operators-app`
   * `common-stateful-resources-app`

9. Monitoring Stack

   * `monitoring-app`
   * `monitoring-install`
   * `monitoring-post-config`

10. Identity & Access Management

* `keycloak-app`
* `keycloak-install`
* `keycloak-post-config`
* `ory-app`

11. PM4ML Core

* `pm4ml`

---

## 6.2 Clone Repository

```bash
git clone <your-onprem-gitops-repo>
cd <repo>
```

---

# PM4ML Deployment – Quick Guide

## 1. Configure Inventory

Edit:

```
ansible/inventory.ini
```

Update:

* Cluster node IPs (master/worker)
* HAProxy internal and external IPs
* Ceph node IPs
* SSH key path

---

## 2. Configure Global Variables

Edit:

```
ansible/group_vars/all.yml
```

Update:

* `microk8s_version`
* `git_url`
* `git_branch`
* `git_user`
* `git_password`
* `nic_iface`
* DNS and storage settings

---

## 3. Run Deployment

Execute in order:

```bash
ansible-playbook -i inventory.ini 1.microk8s_setup.yml
ansible-playbook -i inventory.ini 2.argocd_setup.yml
ansible-playbook -i inventory.ini 3.haproxy_setup.yml
ansible-playbook -i inventory.ini 4.ceph.yml
ansible-playbook -i inventory.ini 5.apps_setup.yml
ansible-playbook -i inventory.ini 6.system-tuning.yml
```

---

## 4. Storage Backend

Use **Ceph** for:

- Loki log storage
- Non-critical persistent volume workloads

Ceph is **not** used for latency-sensitive or transaction-critical workloads.

Ensure:

- The Ceph cluster is healthy
- A proper StorageClass is configured
- Relevant applications reference the Ceph StorageClass in their Helm values
---

## 5. Certificate Configuration

Choose one:

* Public domain → HTTP01 challenge
* Internal environment → Vault PKI or Corporate CA

---

## 6. DNS Records

Create A records:

```
argocd.<domain> → <external_haproxy_ip>
walletX.<domain> → <external_haproxy_ip>
```

---

## 7. Vault Setup

* Use Raft storage
* Enable PKI
* Enable Kubernetes Auth

---

## 8. Manual Secret (Hub Integration)

To enable PM4ML to connect with the Mojaloop Hub, create the following secret in Vault.
The client secret must be obtained securely from the Mojaloop Hub operator.

Create in Vault (KV path):

```
secret/<pm4ml_id>/
```

Key:

```
mcmdev_client_secret
```

---

## 9. Verification

```bash
kubectl get nodes
kubectl get app -A
systemctl status haproxy
```

Deployment is successful when:

* All nodes are `Ready`
* Argo CD applications are `Synced` and `Healthy`
* HAProxy service is `active`

---
