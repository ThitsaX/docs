---
title: Hub ↔ PM4ML Security Architecture
---

# 🔐 Hub ↔ PM4ML Security Architecture

This document describes the **actual security architecture** between the **Hub** and **PM4ML**, covering **mTLS (transport security)** and **JWS (application-level signing)**, based on the current PM4ML implementation.

---

## 1. mTLS (Transport-Level Security)

mTLS is used to secure HTTPS communication between Hub and PM4ML.
**Inbound and outbound directions use different certificate lifecycles.**

---

## 2. Inbound mTLS (Hub → PM4ML)

### Flow

* Hub initiates a connection to PM4ML
* PM4ML acts as **TLS server**
* PM4ML presents a **server certificate**

```text
Hub ──mTLS──▶ PM4ML
        (PM4ML server cert)
```

### Certificate lifecycle

* Certificate is:

  * Issued by **cert-manager**
  * Backed by a **Vault CA (ClusterIssuer)**
  * Stored as a Kubernetes `Secret`
* Rotation:

  * Fully managed by **cert-manager**
  * Based on `duration` and `renewBefore`
  * Pods reload or restart to pick up new certs

### Ownership

* PM4ML owns and manages its **server certificate**
* Hub only verifies the certificate chain

✔ **Inbound mTLS is cert-manager–managed**

---

## 3. Outbound mTLS (PM4ML → Hub)

### Flow

* PM4ML initiates a connection to Hub
* PM4ML acts as **TLS client**
* PM4ML presents a **client certificate**

```text
PM4ML ──mTLS──▶ Hub
        (PM4ML client cert)
```

### Certificate lifecycle (DFSP trust workflow)

* PM4ML:

  * Generates a **private key + CSR**
  * Sends CSR to **Hub**
* Hub:

  * Validates DFSP identity
  * **Signs the CSR**
* PM4ML:

  * Stores:

    * private key
    * signed certificate
    * CA chain
  * Storage location: **Vault**

### Rotation

* No cert-manager involvement
* Rotation is triggered by:

  * deleting Vault secrets
  * restarting management-api
* management-api re-runs the CSR → Hub signing flow

✔ **Outbound mTLS is Vault-stored and Hub-signed**
✔ **cert-manager is intentionally NOT used**

---

## 4. Why Two Different mTLS Models Exist

| Direction   | Trust owner          | Reason                     |
| ----------- | -------------------- | -------------------------- |
| Hub → PM4ML | Platform / Infra     | Standard service identity  |
| PM4ML → Hub | Hub (business trust) | DFSP onboarding & approval |

Outbound mTLS is part of **DFSP trust establishment**, which requires explicit Hub approval and therefore cannot be automated by cert-manager.

---

## 5. JWS (Application-Level Message Signing)

JWS is used to ensure **message integrity and authenticity**, independent of TLS.

```text
┌──────────────┐        JWS-signed payloads        ┌──────────────┐
│     HUB      │  ─────────────────────────────▶   │    PM4ML     │
│              │  ◀─────────────────────────────   │              │
└──────────────┘        JWS-signed payloads        └──────────────┘
```

### Key model

* Hub and PM4ML each own:

  * a **JWS private key**
  * a corresponding **public key**
* Public keys are published to **MCM Server**
* Private keys are never shared

### Key properties

* JWS keys:

  * ❌ do not expire
  * ❌ have no TTL
* Rotation is **policy-based**, not time-based
* Old keys remain valid for verification until removed

---

## 6. JWS Public-Key Publishing (Hub example)

```text
cert-manager
   │
   │ rotates switch-jws certificate
   ▼
Kubernetes Secret (switch-jws)
   │
   │ initContainers:
   │  - read tls.crt
   │  - extract public key
   ▼
jws-pubkey-job
   │
   │ POST /api/hub/jwscerts
   ▼
MCM Server
```

* cert-manager rotates the certificate
* Public key is extracted and published
* MCM stores multiple valid public keys
* Enables safe JWS key rotation with overlap

---

## 7. Runtime Verification Flow

### Hub → PM4ML

1. mTLS:

   * PM4ML verifies Hub’s TLS certificate
2. JWS:

   * PM4ML verifies signature using Hub public key from MCM

### PM4ML → Hub

1. mTLS:

   * Hub verifies PM4ML’s client certificate (Hub-signed)
2. JWS:

   * Hub verifies signature using PM4ML public key from MCM

Both layers must succeed for a request to be accepted.

---

## 8. Expiry & Rotation Summary

| Item                                      | Direction            | Expires? | Managed by        | Rotation method       |
|-------------------------------------------|----------------------|----------|-------------------|-----------------------|
| Inbound mTLS cert (PM4ML server)           | Hub → PM4ML          | ✅ Yes   | cert-manager      | Automatic             |
| Outbound mTLS cert (Hub as client)         | Hub → PM4ML          | ✅ Yes   | Hub + Vault       | Manual / workflow     |
| Inbound mTLS cert (Hub server)             | PM4ML → Hub          | ✅ Yes   | cert-manager      | Automatic             |
| Outbound mTLS cert (PM4ML as client)       | PM4ML → Hub          | ✅ Yes   | PM4ML + Vault     | Manual / workflow     |
| JWS public key (Hub)                       | Hub → PM4ML          | ❌ No    | MCM-Server        | Published on rotation |
| JWS public key (PM4ML)                     | PM4ML → Hub          | ❌ No    | MCM-Client        | Published on rotation |
| JWT token (issued by Hub)                  | Hub → PM4ML          | ✅ Yes   | Application       | `exp` claim           |


---

## 9. Key Takeaways

* **Inbound mTLS** → cert-manager–managed
* **Outbound mTLS** → Vault-stored, Hub-signed
* **JWS** → key-based, no expiry, rotated by policy
* **MCM** is the trust registry for JWS public keys

---

## 10. Summary

> **Hub–PM4ML communication uses cert-manager–managed mTLS for inbound connections, Hub-signed Vault-stored certificates for outbound connections, and JWS for message integrity with public keys distributed via MCM.**