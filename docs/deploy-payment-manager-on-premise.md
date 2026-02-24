# Payment Manager for Mojaloop (PM4ML)

## On-Premise Deployment & Environment Architecture Guide

This document provides step-by-step guidance for deploying **Payment Manager for Mojaloop (PM4ML)** in an on-premise data center environment using:

* Ubuntu 24.04 LTS
* MicroK8s (Kubernetes)
* Ansible
* GitOps with Argo CD
* HAProxy (External & Internal Load Balancer)
* WireGuard VPN (Optional)


# Code Base Reference

This deployment architecture is derived from the official Mojaloop Infrastructure as Code (IaC) modules:

- Mojaloop IaC Modules: https://github.com/mojaloop/iac-modules

The original AWS-focused implementation has been adapted and extended to support on-premise infrastructure using MicroK8s, HAProxy, and GitOps-based workflows.

# PM4ML On-Premise Architecture Overview

The following diagram illustrates the logical and network architecture of the PM4ML on-premise deployment.

## Logical Architecture View
![Logical Architecture View](https://github.com/user-attachments/assets/4d244515-a0f3-47ef-b878-663da6c894c9)


## Network Architecture View
![Network Architecture View](https://github.com/user-attachments/assets/d48490fd-e95c-4956-9b8d-293f17af03d0)

# 1. Prerequisites

## 1.1 Infrastructure Requirements (Production Recommended)

| Component               | Count | Purpose                                |
| ----------------------- | ----- | -------------------------------------- |
| HAProxy                 | 1–2   | Public & Private ingress load balancer |
| MicroK8s Master Nodes   | 3     | Kubernetes control plane               |
| Worker Nodes            | 3+    | Application workloads                  |
| Storage Servers (Ceph)  | 3+    | Distributed storage cluster            |


## 1.2 Network Requirements

- Static IP address for each VM  
- Dedicated internal subnet (e.g., `10.10.0.0/16`)  
- Public IP mapped to HAProxy via firewall or edge device  
- DNS zone (AWS Route53, Cloudflare, or enterprise DNS)  


## 1.3 Network Port Requirements

### 1.3.1 Public Network (External Access)

Ports exposed to the internet or external partners.

| Port  | Protocol | Purpose       | Notes                       |
| ----- | -------- | ------------- | --------------------------- |
| 22    | TCP      | SSH           | Restricted by IP whitelist  |
| 80    | TCP      | HTTP          | Redirect to HTTPS           |
| 443   | TCP      | HTTPS         | Public API / Portal         |
| 51820 | UDP      | WireGuard VPN | If WireGuard VPN is enabled |

> Replace the WireGuard port with the actual port configured on your firewall or edge device (e.g., FortiGate, F5, or equivalent).


### 1.3.2 Private Network (Internal Communication)

Ports used for internal cluster, storage, and load balancer communication.  
These ports must not be exposed publicly.

#### MicroK8s (Kubernetes Cluster)

| Port         | Protocol | Purpose                            |
|-------------|----------|------------------------------------|
| 6443        | TCP      | Kubernetes API Server              |
| 10250       | TCP      | Kubelet                            |
| 25000       | TCP      | MicroK8s Cluster Agent             |
| 12379-12380 | TCP      | etcd (MicroK8s datastore)          |


#### Istio Ingress Gateways (NodePort)

HAProxy forwards traffic to Kubernetes nodes via NodePort services.

**External Istio Ingress Gateway**

| Port  | Protocol | Purpose              |
|-------|----------|----------------------|
| 32080 | TCP      | HTTP (NodePort)      |
| 32443 | TCP      | HTTPS (NodePort)     |
| 32081 | TCP      | Status Port (15021)  |

**Internal Istio Ingress Gateway**

| Port  | Protocol | Purpose              |
|-------|----------|----------------------|
| 31080 | TCP      | HTTP (NodePort)      |
| 31443 | TCP      | HTTPS (NodePort)     |
| 31081 | TCP      | Status Port (15021)  |

> These NodePorts must be accessible only from HAProxy source IP addresses.  
> They must not be exposed to public networks.


#### MicroCeph (Ceph Cluster)

| Port       | Protocol | Purpose                     |
|-----------|----------|-----------------------------|
| 6789      | TCP      | Ceph Monitor (v1)           |
| 3300      | TCP      | Ceph Monitor (v2)           |
| 6800-7300 | TCP      | Ceph OSD communication      |


#### HAProxy (Load Balancer)

| Port | Protocol | Purpose                         |
|------|----------|---------------------------------|
| 80   | TCP      | Frontend HTTP                   |
| 443  | TCP      | Frontend HTTPS                  |
| 8404 | TCP      | HAProxy statistics (if enabled) |


### Internal Security Controls

- NodePort ranges must be restricted to HAProxy source IPs only.  
- Kubernetes control-plane ports must be restricted to cluster nodes.  
- Ceph cluster traffic must remain within the private storage network.  
- No NodePort service may be directly exposed to the internet.  
- All ingress traffic must pass through HAProxy before reaching Istio.  
- TLS or mTLS must be enforced where applicable.  


## 1.4 Storage Requirements

- Ceph RBD for Kubernetes persistent volumes  
- Ceph-backed storage for logs, backups, and non-transaction-critical workloads  
- Transaction-critical databases and messaging systems should use local high-performance storage (e.g., NVMe) where required  

# 2. Tools & Versions

| Tool     | Version                  |
| -------- | ------------------------ |
| Ubuntu   | 24.04 LTS                |
| MicroK8s | 1.31/stable              |
| Ansible  | 2.18+                    |
| Helm     | v3                       |
| kubectl  | Matching cluster version |
| Git      | Latest stable            |


# 3. VM Provisioning

VMs may be provisioned using:

* VMware
* Proxmox
* OpenStack
* CloudStack
* Bare Metal

# 4. Infrastructure Sizing by Environment

## 4.1 Production Environment (PROD)

### Recommended Production Infrastructure Sizing

| Component                       | Quantity   | CPU / RAM / Storage (Per Server)                                                                 | Deployment Type | Notes                                   |
|----------------------------------|------------|--------------------------------------------------------------------------------------------------|----------------|------------------------------------------|
| Master Node Cluster              | ×3 Servers | 8 CPU Cores, 32 GB RAM, 2 × 300 GB NVMe                                                         | VM             | Dedicated control plane nodes            |
| Worker Node Cluster              | ×3 Servers | 16 CPU Cores, 64 GB RAM, 2 × 500 GB NVMe                                                        | VM             | Application workloads only               |
| Load Balancers (HAProxy)         | ×2 Servers | 4 CPU Cores, 16 GB RAM, 100 GB SSD                                                              | VM             | Active/Active or Active/Standby          |
| Cold Data Storage Cluster        | ×3 Servers | 16 CPU Cores, 64 GB RAM; OS: 2 × 500 GB SSD (RAID1); Data: 6 × 8 TB HDD; Network: 2 × 10 Gbps | Physical       | Recommended enterprise-grade storage     |
| Cold Data Storage Cluster (Alternative)  | ×3 VMs     | 16 vCPU, 64 GB RAM; OS: 200 GB SSD; Data: 20–40 TB dedicated enterprise block storage                | VM             | Must match physical capacity & IOPS SLA  |
| Bastion Host (Optional)          | ×1 Server  | 2 CPU Cores, 8 GB RAM, 50 GB SSD                                                                | VM             | Secure administrative access             |
| Backup Target                    | ×1         | Enterprise backup system                                                        | External       | Off-site backup retention                |

### Cold Storage Purpose

Cold storage infrastructure is designated exclusively for:

- Long-term regulatory record retention  
- Backup archives  
- Ledger export snapshots  
- Log archival storage  

Cold storage must **not** be used for:

- Latency-sensitive transaction processing  
- Real-time database workloads  
- Messaging systems or performance-critical services  

This separation ensures optimal transaction performance while meeting regulatory compliance and retention requirements.


### VM-Based Cold Storage Considerations

If cold storage is deployed using virtual machines instead of physical servers, the following conditions must be met:

- Underlying storage must be enterprise-grade (SAN, NAS, or dedicated block storage)
- Storage must not be oversubscribed
- IOPS and throughput guarantees must be documented
- Network bandwidth must be sufficient (minimum 10 Gbps recommended)
- Backup and redundancy policies must match physical deployment standards

VM-based cold storage is acceptable only when the virtualization platform provides guaranteed performance and isolation equivalent to dedicated physical storage.

## 4.2 Staging Environment (STG)

Designed for:

* Functional testing
* Integration testing
* Performance validation
* Pre-production validation

### Recommended Staging Infrastructure Sizing

| Component                  | Quantity   | CPU / RAM / Storage (Per Server)                                   | Deployment Type | Notes                                |
|----------------------------|------------|--------------------------------------------------------------------|----------------|--------------------------------------|
| Master + Worker Cluster    | ×2 Servers | 8 CPU Cores, 16-32 GB RAM, 500 GB SSD                                 | VM             | Combined control plane & workloads   |
| Load Balancer (HAProxy)    | ×1 Server  | 2 CPU Cores, 4 GB RAM, 100 GB SSD                                   | VM             | Single instance                      |
| Cold Data Storage          | ×1 Server  | 8 CPU Cores, 32 GB RAM; OS: 200 GB SSD; Data: 2 × 2 TB HDD | VM or Physical | Reduced retention, non-production logs |


## 4.3 Environment Policy

* Production must use dedicated control plane nodes
* Production must maintain high availability
* Staging may operate with reduced redundancy
* Any deviation from Production requirements must be formally approved


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

# 5. HAProxy Configuration (On-Premise)

## 5.1 Overview

HAProxy acts as the ingress load-balancing layer in front of the MicroK8s cluster.

Architecture:

```
Client → Perimeter Firewall → HAProxy → Istio Ingress Gateway → PM4ML Services
```

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

## 5.3 HAProxy Interfaces

| Interface | Purpose                        |
| --------- | ------------------------------ |
| Public    | Receives traffic from firewall |
| Private   | Forwards traffic to Kubernetes |


## 5.4 TLS Termination Strategy

TLS termination occurs at Istio Ingress Gateway.

Benefits:

* Centralized certificate lifecycle management
* Better integration with service mesh policies
* Improved observability
* Reduced HAProxy complexity

## 5.5 High Availability

* Two HAProxy nodes (Production)
* Firewall-level failover or VIP
* Separate physical hosts where possible

## 5.6 Security Controls

* Expose only required ports (typically 443)
* Restrict SSH access
* Apply host-level firewall rules
* Integrate logs into monitoring/SIEM

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
* Bastion Host(Optional)

# 6. GitOps Deployment

The customer must provide and maintain their own Git repository for GitOps-based deployment.

Requirements:

- Repository accessible from the cluster (HTTPS or SSH)
- Proper branch or tag strategy defined
- Secure credential/token management
- Access configured in Argo CD

All platform manifests and Helm values must be stored in the customer's repository.

## 6.1 Platform Applications Deployment Order

Argo CD applications must be deployed in the following sequence to satisfy dependencies and ensure system stability.

- **Base Utilities**
  - `base-utils`

- **Storage Layer**
  - `storage-app`

- **Certificate Management**
  - `certmanager-helm`
  - `certmanager-app`
  - `certmanager-clusterissuers`

- **Service Mesh**
  - `istio-app`
  - `istio-main-app`
  - `istio-gateways-app`

- **DNS Management**
  - `external-dns-app`

- **Vault Storage Backend**
  - `consul-app`

- **Vault & Secrets Management**
  - `vault-app`
  - `vault`
  - `vault-config-operator`
  - `vault-pki-app`

- **Stateful Resources**
  - `stateful-resources-operators-app`
  - `common-stateful-resources-app`

- **Monitoring Stack**
  - `monitoring-app`
  - `monitoring-install`
  - `monitoring-post-config`

- **Identity & Access Management**
  - `keycloak-app`
  - `keycloak-install`
  - `keycloak-post-config`
  - `ory-app`

- **PM4ML Core**
  - `pm4ml`

## 6.2 Clone Repository

```bash
git clone <your-onprem-gitops-repo>
cd <repo>
```

## 6.3 Pre-Deployment Checklist (Applications)

Before running `5.apps_setup.yml`, verify the following:

### Configuration

* [ ] Environment values updated (prod / staging)
* [ ] DNS hostnames updated
* [ ] Istio Gateway hosts updated
* [ ] VirtualService URLs updated
* [ ] PM4ML endpoints configured correctly

### Certificates & TLS

* [ ] cert-manager Issuer / ClusterIssuer configured
* [ ] TLS secrets created or auto-generated
* [ ] DNS challenge credentials configured (if applicable)
* [ ] Certificate CN/SAN matches domain
* [ ] mTLS configuration validated (if required)

### Secrets & Integrations

* [ ] Database credentials updated
* [ ] External service credentials updated
* [ ] Vault / ExternalSecrets configuration validated
* [ ] No placeholder secrets remain

### Storage

* [ ] Correct StorageClass referenced
* [ ] PVC sizes reviewed
* [ ] Stateful workloads reference correct volumes

### Resource Management

* [ ] CPU requests and limits defined
* [ ] Memory requests and limits defined
* [ ] HPA configuration validated (if enabled)

Deployment must not proceed unless all checklist items are verified.

## 6.4 Run Deployment

Execute in order:

```bash
ansible-playbook -i inventory.ini 1.microk8s_setup.yml
ansible-playbook -i inventory.ini 2.argocd_setup.yml
ansible-playbook -i inventory.ini 3.haproxy_setup.yml
ansible-playbook -i inventory.ini 4.ceph.yml
ansible-playbook -i inventory.ini 5.apps_setup.yml
ansible-playbook -i inventory.ini 6.system-tuning.yml
```

## 6.5 Manual Secret (Hub Integration/Onboarding)

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

## 6.6 Verification

```bash
kubectl get nodes
kubectl get app -A
```

Deployment is successful when:

- All nodes are `Ready`
- Argo CD applications are `Synced` and `Healthy`
- HAProxy service is `active`
- `https://argocd.<domain>` is accessible and displays the Argo CD login page
- PM4ML endpoint URLs respond with expected HTTP status codes
- Internal services are reachable from within the cluster
