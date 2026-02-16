# 🚀 Deploy Mojaloop Locally Using MicroK8s (Ubuntu 24.04 LTS)

---

## 📘 Introduction

This guide explains how to deploy a full Mojaloop environment locally using:

- Ubuntu 24.04 LTS
- MicroK8s
- Helm

The setup includes:

- Mojaloop Core Services
- Testing Toolkit (TTK)
- Two DFSP simulators (payer & payee)
- DFSP onboarding
- P2P transfer simulation

> ⚠️ This environment is for development and testing only.

---

# 🖥️ System Requirements

Minimum:

- Ubuntu 24.04 LTS
- 16GB RAM (32GB recommended)
- 4+ CPU cores
- 50GB disk
- sudo privileges

Verify Ubuntu version:

```bash
lsb_release -a
````

---

# 1️⃣ Install MicroK8s

```bash
sudo snap install microk8s --classic --channel=1.31/stable
```

Add your user:

```bash
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

Verify:

```bash
microk8s status --wait-ready
```

---

# 2️⃣ Enable Required Add-ons

```bash
microk8s enable dns
microk8s enable ingress
microk8s enable storage
microk8s enable helm3
```

Alias kubectl:

```bash
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc
```

---

# 3️⃣ Add Helm Repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add elastic https://helm.elastic.co
helm repo add mojaloop https://mojaloop.io/helm/repo/
helm repo update
```

---

# 4️⃣ Deploy Mojaloop Backend

```bash
helm install backend mojaloop/example-mojaloop-backend \
  --namespace mojaloop \
  --create-namespace
```

Wait for pods:

```bash
kubectl get pods -n mojaloop
```

---

# 5️⃣ Deploy Mojaloop Core

```bash
helm install dev mojaloop/mojaloop \
  --namespace mojaloop
```

Verify:

```bash
kubectl get pods -n mojaloop
```

---

# 6️⃣ Configure Local DNS

Edit:

```bash
sudo nano /etc/hosts
```

Add:

```
127.0.0.1 testing-toolkit.local
127.0.0.1 testing-toolkit.payer
127.0.0.1 testing-toolkit.payee
127.0.0.1 mojaloop.local
```

---

# 7️⃣ Deploy DFSP 1 — Payer

## Create Namespace

```bash
kubectl create namespace payer
```

## Create payer_values.yaml

```bash
touch payer_values.yaml
nano payer_values.yaml
```

Add:

```yaml
global:
  fspId: payer

config:
  mojaloop:
    endpoint: http://dev-ml-api-adapter-service.mojaloop:3000
  callback:
    endpoint: http://payer-ml-testing-toolkit-backend.payer:4040

ingress:
  enabled: true
  hosts:
    - host: testing-toolkit.payer
      paths:
        - /
```

## Deploy

```bash
helm install payer mojaloop/ml-testing-toolkit \
  --namespace payer \
  -f payer_values.yaml
```

Verify:

```bash
kubectl get pods -n payer
```

Access:

```
http://testing-toolkit.payer
```

---

# 8️⃣ Deploy DFSP 2 — Payee

```bash
kubectl create namespace payee
touch payee_values.yaml
nano payee_values.yaml
```

Add:

```yaml
global:
  fspId: payee

config:
  mojaloop:
    endpoint: http://dev-ml-api-adapter-service.mojaloop:3000
  callback:
    endpoint: http://payee-ml-testing-toolkit-backend.payee:4040

ingress:
  enabled: true
  hosts:
    - host: testing-toolkit.payee
      paths:
        - /
```

Deploy:

```bash
helm install payee mojaloop/ml-testing-toolkit \
  --namespace payee \
  -f payee_values.yaml
```

Access:

```
http://testing-toolkit.payee
```

---

# 9️⃣ DFSP Onboarding

1. Open `http://testing-toolkit.local`
2. Go to **Test Runner**
3. Import collections from:

```
https://github.com/mojaloop/ml-testing-toolkit-test-cases
```

Run:

* hub_accounts.json
* new_dfsp.json (for payer)
* new_dfsp.json (for payee)

Update callback URLs per namespace.

---

# 🔟 Simulate P2P Transfer

From payer TTK:

1. Load:

```
p2ptransfer/collections/p2p_happy_path.json
```

2. Load environment:

```
p2ptransfer/envs/p2p_happy_path_env.json
```

3. Click Run

Monitor payee TTK → Monitoring

You should observe:

* GET /parties
* POST /quotes
* POST /transfers

---

# 🧹 Cleanup

```bash
helm uninstall dev -n mojaloop
helm uninstall backend -n mojaloop
kubectl delete namespace mojaloop payer payee pm4ml
```

---

# ✅ Summary

You now have a full local Mojaloop environment including:

* MicroK8s Kubernetes
* Mojaloop core
* Two DFSP simulators
* DFSP onboarding
* P2P transfer simulation

This environment can be used for:

* DFSP development
* Integration testing
* Architecture learning
