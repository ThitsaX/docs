# 🏢 Draft - Payment Manager for Mojaloop Production Deployment (On-Premise)

This guide provides step-by-step instructions for deploying **Payment Manager for Mojaloop (PM4ML)** in an **on-premise data center environment** using:

- Ubuntu 24.04 LTS
- MicroK8s (Kubernetes)
- Ansible
- GitOps with ArgoCD
- HAProxy (External & Internal Load Balancer)
- WireGuard VPN (Optional)

---
## 📦 Code Base Reference

This deployment architecture is derived from the official Mojaloop Infrastructure as Code (IaC) modules:

👉 https://github.com/mojaloop/iac-modules

The original AWS-focused implementation has been adapted and extended to support on-premise infrastructure using MicroK8s, HAProxy, and GitOps-based workflows.

---
# 🚀 Platform Applications Deployment Order

The following ArgoCD applications are deployed in a structured sequence to satisfy dependencies and ensure system stability.

---

## 1️⃣ Base Utilities
- `base-utils`

## 2️⃣ Storage Layer
- `storage-app`

## 3️⃣ Certificate Management
- `certmanager-helm`
- `certmanager-app`
- `certmanager-clusterissuers`

## 4️⃣ Service Mesh
- `istio-app`
- `istio-main-app`
- `istio-gateways-app`

## 5️⃣ DNS Management
- `external-dns-app`

## 6️⃣ Vault Storage Backend
- `consul-app`  
  *(Provides the storage backend for HashiCorp Vault)*

## 7️⃣ Vault & Secrets Management
- `vault-app`
- `vault`
- `vault-config-operator`
- `vault-pki-app`

## 8️⃣ Stateful Resources
- `stateful-resources-operators-app`
- `common-stateful-resources-app`

## 9️⃣ Monitoring Stack
- `monitoring-app`
- `monitoring-install`
- `monitoring-post-config`

## 🔐 1️⃣0️⃣ Identity & Access Management
- `keycloak-app`
- `keycloak-install`
- `keycloak-post-config`
- `ory-app`

## 💳 1️⃣1️⃣ PM4ML Core
- `pm4ml`

---


# 📌 1. Prerequisites

## 🖥 Infrastructure Requirements (Production Recommended)

| Component | Count | Purpose |
|------------|--------|----------|
| HAProxy    | 1–2 | Public & Private ingress load balancer |
| MicroK8s Master Nodes | 3 | Kubernetes control plane |
| Worker Nodes | 3+ | Application workloads |
| Storage Server (Ceph) | 3+ | Cold storage |
| Bastion Host(Optional) | 1 | Secure administrative access |
| NAT Gateway (Optional) | 1 | Outbound internet for private subnet |

---

## 🌐 Network Requirements

- Static IP for each VM
- Internal subnet (e.g. `10.10.0.0/16`)
- Public IP mapped to HAProxy
- DNS zone (AWS or Cloudflare)
- Required Ports:

| Port | Purpose |
|------|----------|
| 22 | SSH |
| 80  | HTTP  |
| 443 | HTTPS |
| WireGuard port | VPN (if enabled) |

---

## 💾 Storage Requirements

- Ceph RBD / CephFS
- S3-compatible

---

# 📦 2. Tools & Versions

| Tool | Version |
|------|----------|
| Ubuntu | 24.04 LTS |
| MicroK8s | 1.31/stable |
| Ansible | 2.18+ |
| Helm | v3 |
| kubectl | Matching cluster version |
| Git | Latest |

---

# 🏗 3. VM Provisioning

Provision VMs using:

- VMware
- Proxmox
- OpenStack
- CloudStack
- Bare Metal

### Recommended VM Sizing

| Role | vCPU | RAM | Disk |
|------|------|------|------|
| Master | 4 | 16GB | 200GB |
| Worker | 8 | 16GB | 200GB |
| HAProxy | 2 | 4GB | 50GB |
| Bastion(Optional) | 2 | 4GB | 40GB |
| Storage | 8 | 32GB | Based on workload |

---

# 🌍 5. HAProxy Configuration (On-Prem)

## HAProxy

* Two network interfaces required
* Binds to public IP for public subnet
* Binds to private IP for private subnet 
---

# 🔁 6. GitOps Deployment

## Clone GitOps Repository

```bash
git clone <your-onprem-gitops-repo>
cd <repo>
```

## Update Configuration

Modify:

* `ansible/inventory.ini`
```
[cluster]
node1 ansible_host=10.x.x.x role=master
node2 ansible_host=10.x.x.x role=master
node3 ansible_host=10.x.x.x role=master
node4 ansible_host=10.x.x.x role=worker
node5 ansible_host=10.x.x.x role=worker
node6 ansible_host=10.x.x.x role=worker
;if you want both master and worker in one single node remove the role

[haproxy]
haproxy ansible_host=10.x.x.x.x ext_haproxy_ip=13.x.x.x int_haproxy_ip=10.x.x.x 

[ceph]
node1 ansible_host=10.x.x.x
node2 ansible_host=10.x.x.x
node3 ansible_host=10.x.x.x

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/<sshkeyname>
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

```


* `ansible/group_vars/all.yml`
```
microk8s_version : "1.31/stable"
git_url: "https://<your gitops repo>.git"
git_branch: "<user main branch or tag>"
git_user: "<git username>"
git_password: "<git password or toen>"
nic_iface: <eth0>
```

Update:

* DNS configurations
* URLs
* S3 bucket configurations
* PM4ML configurations
---

# 🚀 7. Run Ansible Playbooks

```bash
ansible-playbook -i inventory.ini 1.microk8s_setup.yml
ansible-playbook -i inventory.ini 2.argocd_setup.yml
ansible-playbook -i inventory.ini 3.haproxy_setup.yml
ansible-playbook -i inventory.ini 4.ceph.yml
ansible-playbook -i inventory.ini 5.apps_setup.yml
ansible-playbook -i inventory.ini 6.system-tuning.yml
```

---

# 💾 8. Storage Configuration Changes

## Longhorn Backup (NFS Example)

Replace S3 configuration with:

```yaml
backupTarget: "nfs://10.10.0.50:/longhorn-backup"
```

---

## Loki Configuration (Filesystem)

```yaml
loki:
  storage:
    type: filesystem
    filesystem:
      directory: /var/loki
```

---

## Tempo Configuration

```yaml
tempo:
  storage:
    trace:
      backend: local
      local:
        path: /var/tempo
```

---

# 🔐 9. Certificate Management

## Option A – Public Domain (HTTP01)

Use HTTP challenge if publicly reachable.

## Option B – Internal CA

Use:

* Vault PKI
* Corporate CA

Remove Route53 DNS01 solver configuration.

---

# 🌐 10. DNS Configuration

Create manual A records:

```
argocd.prod-pm4ml.domain.com → External HAProxy IP
wallet1.prod-pm4ml.domain.com → External HAProxy IP
```

---

# 🔐 11. Vault Configuration

* Use Integrated Storage (Raft)
* Disable AWS KMS auto-unseal (if previously used)
* Configure Vault PKI
* Enable Kubernetes Auth

---

# 📊 12. Monitoring Stack Adjustments

Replace:

* S3-backed Loki
* S3-backed Tempo

With:

* NFS
* Ceph
* MinIO (optional)

Update:

* `values-loki.yaml`
* `values-tempo.yaml`

---

# 🔁 13. Manual Secret Sync (Hub → PM4ML)

## Ask hub operator for client secret for authentication to hub

Then create secret in the PM4ML vault
Path:

```
secret/<yourpm4ml_id>
```

Key:

```
mcmdev_client_secret
```
Value:
```
secret_key_from_hub_operator
```
---

# 🔎 15. Verification

## Verify Cluster

```bash
kubectl get nodes
```

## Verify ArgoCD Apps

```bash
kubectl get app -A
```

## Verify HAProxy

```bash
systemctl status haproxy
```

---