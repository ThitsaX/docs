# Pre-Deployment Checklist

This checklist is used before deploying Mojaloop platform components or Payment Manager for Mojaloop (PM4ML) into an integration, staging, or production environment.

It does not replace the deployment guide. It confirms that the environment, access, networking, security, and operational basics are ready before the deployment begins.

The main goal is to avoid starting a deployment when a required dependency is still missing or unclear.

---

## 1. Deployment Scope

Confirm the deployment model before preparing the environment.

| Check | Expected result |
| ----- | --------------- |
| Target environment is identified | Integration, staging, production, DR, or local lab |
| Deployment type is selected | Helm, GitOps, on-premise, cloud, hybrid, or local |
| Components are in scope | PM4ML, Mojaloop Hub, Redis, Kafka, MySQL, Tazama, monitoring, logging |
| Out-of-scope items are recorded | Items not being deployed in this phase are explicitly listed |
| Deployment owner is assigned | One person or team owns coordination |
| Technical approvers are known | Infrastructure, security, network, application, and Hub/operator contacts |

Record the following before the deployment starts:

```text
Environment:
Deployment date:
Deployment owner:
Application owner:
Infrastructure owner:
Security owner:
Git repository:
Git branch or commit:
Helm chart versions:
Target namespace:
Base domain:
```

Why this matters:

- Deployment issues become harder to resolve when ownership is unclear.
- Version drift between Git, Helm, and the running cluster can make rollback difficult.
- Explicit scope prevents teams from assuming that monitoring, backups, or security controls are already included.

---

## 2. Infrastructure Readiness

Confirm that compute, storage, and operating system requirements are ready.

| Check | Expected result |
| ----- | --------------- |
| Servers or VMs are provisioned | Required master, worker, load balancer, and storage nodes exist |
| OS version is confirmed | Ubuntu 24.04 LTS or the approved target OS |
| Time synchronization is enabled | NTP or chrony is configured on all nodes |
| Hostnames are stable | Hostnames match the deployment inventory |
| Static IPs are assigned | IPs match the network plan |
| Disk layout is confirmed | OS, application, database, logs, and backup disks are separated where required |
| CPU and memory sizing is approved | Sizing matches the expected environment tier |
| Storage backend is available | Ceph, Rook-Ceph, Longhorn, cloud disk, or local PV is ready |

Basic validation commands:

```bash
hostnamectl
timedatectl
df -h
lsblk
free -h
ip addr
```

For Kubernetes nodes:

```bash
kubectl get nodes -o wide
kubectl describe nodes
```

Why this matters:

- Time drift can break certificates, JWS validation, logs, and audit trails.
- Incorrect disk layout can cause transaction workloads, logs, and backups to compete for the same storage.
- Node sizing issues usually appear later as pod scheduling, Redis, Kafka, database, or ingress instability.

---

## 3. Network and DNS Readiness

Confirm that all required network paths are open before application deployment.

| Check | Expected result |
| ----- | --------------- |
| Internal subnet is confirmed | Cluster nodes can communicate privately |
| Public IP or edge routing is confirmed | External traffic reaches HAProxy, ingress, or the approved edge component |
| Firewall rules are approved | Only required ports are open |
| NodePorts are restricted | NodePort access is allowed only from the load balancer where applicable |
| DNS zone is available | Required records can be created or updated |
| Internal DNS is available | Internal services and private endpoints resolve correctly |
| Partner or Hub endpoints are reachable | Outbound traffic to required external endpoints is allowed |
| VPN access is tested | Administrative or partner VPN paths work if required |

Recommended DNS records to confirm:

```text
argocd.<domain>
pm4ml.<domain>
admin.<domain>
api.<domain>
hub.<domain>
```

Basic validation commands:

```bash
dig argocd.<domain>
dig pm4ml.<domain>
curl -vk https://pm4ml.<domain>
nc -vz <host> 443
nc -vz <host> <nodeport>
```

Why this matters:

- A Kubernetes deployment can be healthy while users and partners still cannot reach it.
- DNS and firewall issues are often mistaken for application failures.
- NodePort exposure mistakes can bypass the intended HAProxy or ingress security boundary.

---

## 4. Kubernetes Platform Readiness

Confirm that the Kubernetes platform is ready for application workloads.

| Check | Expected result |
| ----- | --------------- |
| Cluster access works | `kubectl get nodes` returns all nodes |
| Nodes are ready | All required nodes are `Ready` |
| CNI is healthy | Pods can communicate across nodes |
| CoreDNS is healthy | Cluster DNS resolves service names |
| Ingress controller or Istio is installed | Required gateways and services exist |
| StorageClass is available | Redis and other persistent workloads can provision volumes |
| Metrics are available | CPU, memory, and pod metrics can be collected |
| Namespace plan is agreed | Namespaces and labels match the deployment model |
| Resource quota policy is known | Quotas and limits will not block deployment |

Validation commands:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
kubectl get svc -A
kubectl get ingress -A
kubectl top nodes
kubectl top pods -A
```

If Istio is used:

```bash
kubectl get pods -n istio-system
kubectl get gateway -A
kubectl get virtualservice -A
```

Why this matters:

- Application deployment should not begin until the platform control plane, networking, ingress, and storage are already healthy.
- Missing StorageClass or broken DNS usually causes application failures that are expensive to debug after Helm or GitOps has started.

---

## 5. GitOps and Release Inputs

Confirm that the deployment source is ready and reviewed.

| Check | Expected result |
| ----- | --------------- |
| Deployment repository is accessible | Deployment team can clone and review the repo |
| Target branch is confirmed | Branch name or commit SHA is recorded |
| Environment values are reviewed | No placeholder values remain |
| Helm chart versions are pinned | Versions are explicit and reproducible |
| Image tags are pinned | Production does not use floating tags such as `latest` |
| Argo CD access is available | Operators can view sync and health status |
| Sync order is understood | Platform dependencies deploy before applications |
| Rollback source is known | Previous working commit or chart version is documented |

Values to review before deployment:

```text
Domain names
Ingress hosts
TLS and mTLS settings
StorageClass names
Resource requests and limits
Redis configuration
Hub endpoint URLs
DFSP identifiers
OAuth or client settings
Vault secret paths
Log level
Replica counts
```

Useful validation commands:

```bash
git status
git branch --show-current
git rev-parse HEAD
helm lint <chart-path>
helm template <release-name> <chart-path> -f <values-file>
```

Why this matters:

- GitOps makes deployment repeatable only when the repository state is clean, reviewed, and pinned.
- Placeholder values can deploy successfully but fail at runtime when traffic begins.
- Rollback is slower when the previous known-good version is not recorded.

---

## 6. Secrets, Certificates, and Trust Material

Confirm that secret and certificate ownership is clear before deployment.

| Check | Expected result |
| ----- | --------------- |
| Secret manager is ready | Vault, Kubernetes secrets, or approved secret store is available |
| Secret paths are defined | Expected secret locations are documented |
| No secrets are stored in Git | Repositories contain references only |
| TLS certificate source is known | cert-manager, enterprise CA, public CA, or manual import |
| mTLS model is agreed | Inbound and outbound certificate ownership is clear |
| JWS key process is agreed | Signing keys and public key publishing flow are defined |
| Certificate expiry is known | Expiry dates and rotation owners are recorded |
| Hub/operator trust workflow is ready | CSR, signing, exchange, or approval process is agreed |

Minimum certificate and key inventory:

```text
PM4ML server TLS certificate:
PM4ML outbound client certificate:
Hub server CA or trust bundle:
Hub client certificate or public key:
JWS private key owner:
JWS public key publishing location:
Vault path:
Rotation owner:
Expiry date:
```

Useful validation commands:

```bash
kubectl get secret -A
kubectl get certificate -A
kubectl get certificaterequest -A
openssl x509 -in <certificate-file> -noout -subject -issuer -dates
```

Why this matters:

- TLS, mTLS, OAuth, and JWS failures can look like application bugs when the real issue is trust material.
- Manual certificate workflows must be planned before the deployment window, especially when Hub approval is required.
- Unknown expiry dates create operational risk after go-live.

---

## 7. Security and Access Controls

Confirm that access is ready and restricted to the right teams.

| Check | Expected result |
| ----- | --------------- |
| Admin access is approved | Only required operators have privileged access |
| Break-glass access exists | Emergency access is documented and controlled |
| SSH access is restricted | Bastion, VPN, or IP allowlist is enforced |
| Kubernetes RBAC is reviewed | Users and service accounts have required permissions only |
| Argo CD access is restricted | Sync and admin rights are limited |
| Vault access is restricted | Secret read/write access is limited by role |
| Audit logging is enabled | Administrative actions can be traced |
| Public endpoints are approved | Exposed endpoints match the security design |

Why this matters:

- Deployment access often becomes permanent access if it is not reviewed before go-live.
- Weak RBAC or broad secret access increases the impact of mistakes and compromise.
- Security controls are easier to validate before production traffic is enabled.

---

## 8. Observability Readiness

Confirm that the team can see the system after it is deployed.

| Check | Expected result |
| ----- | --------------- |
| Logging stack is available | Application and platform logs can be searched |
| Metrics stack is available | CPU, memory, network, pod, and service metrics are collected |
| Dashboards are prepared | Platform and application dashboards exist or are planned |
| Alerts are configured | Critical deployment and runtime failures page the right team |
| Log retention is agreed | Retention period matches operational and compliance needs |
| Alert ownership is assigned | Each alert has an owning team |

Minimum alerts to prepare:

```text
Node NotReady
Pod CrashLoopBackOff
Pod Pending
PersistentVolumeClaim Pending
Ingress or gateway unavailable
Certificate near expiry
Argo CD application OutOfSync
Argo CD application Degraded
Redis unavailable
Database unavailable
High error rate on public endpoints
```

Why this matters:

- A deployment is not operationally ready if the team cannot see failures.
- Alert ownership prevents incidents from bouncing between application, infrastructure, and security teams.
- Certificate and storage alerts are especially important for PM4ML and Mojaloop-style deployments.

---

## 9. Backup and Recovery Readiness

Confirm that recovery requirements are understood before production use.

| Check | Expected result |
| ----- | --------------- |
| Backup scope is defined | Databases, Vault, GitOps repo, certificates, configs, and persistent data are covered |
| Backup target is available | Local, remote, or off-site backup target is ready |
| Retention is agreed | Retention matches business and compliance requirements |
| Restore process is documented | Operators know how to restore critical data |
| Restore test is planned | Backups will be validated, not only created |
| RPO and RTO are known | Recovery expectations are realistic and agreed |

Minimum backup inventory:

```text
Application database:
Redis data:
Vault data:
TLS and mTLS certificates:
JWS keys:
GitOps repository:
Helm values:
Ingress configuration:
DNS records:
```

Why this matters:

- A backup is not useful until restore has been tested.
- Certificate, key, and Vault recovery are as important as database recovery for secure payment integrations.
- RPO and RTO expectations should be agreed before production incidents happen.

---

## 10. Go / No-Go Review

Use this final review before starting the deployment.

| Area | Status |
| ---- | ------ |
| Deployment scope confirmed | Pass / Fail |
| Owners and approvers confirmed | Pass / Fail |
| Infrastructure ready | Pass / Fail |
| Network and DNS ready | Pass / Fail |
| Kubernetes platform healthy | Pass / Fail |
| GitOps or Helm inputs reviewed | Pass / Fail |
| Secrets and certificates ready | Pass / Fail |
| Security access reviewed | Pass / Fail |
| Observability ready | Pass / Fail |
| Backup and recovery plan agreed | Pass / Fail |
| Rollback point identified | Pass / Fail |
| Deployment window approved | Pass / Fail |

Deployment should not begin if any required area is marked `Fail`.

Record the decision:

```text
Go / No-Go decision:
Decision time:
Approved by:
Known risks:
Follow-up actions:
```

---

## Summary

A deployment is ready to start when the platform is healthy, the deployment source is reviewed, access is controlled, trust material is prepared, observability is available, and recovery expectations are agreed.

This checklist should be completed before following the environment-specific deployment guide.
