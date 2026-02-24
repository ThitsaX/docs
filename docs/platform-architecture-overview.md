---
title: Platform Architecture Overview (Hub, PM4ML, Tazama)
---

# 🏗️ Platform Architecture Overview  
**Hub, PM4ML, and Tazama (Cloud & On-Premise)**

This document describes the **reference architecture** for deploying the **Mojaloop Hub**, **PM4ML (Payment Manager for Mojaloop)**, and **Tazama** platforms across **cloud and on-premise datacenter environments**.

The architecture focuses on **deployment topology, platform responsibilities, security boundaries, and operational separation**, without exposing environment-specific or sensitive implementation details.

> **Note**  
> This document presents a reference architecture based on commonly adopted deployment patterns.  
> Specific implementations may vary depending on regulatory, organizational, and operational requirements.

---

## 1. Architectural Goals

The architecture is designed to:

- Support **secure financial transactions**
- Enable **cloud, on-premise, and hybrid deployments**
- Provide **operational resilience** using a Primary–Standby model
- Maintain **clear trust and responsibility boundaries** between platforms
- Scale predictably based on transaction throughput

---

## 2. Deployment Topology (Primary–Standby)

The platform is deployed across **two independent sites**:

### Primary Site
- Active production environment
- Handles live transaction traffic
- Deployed in:
  - Public cloud **or**
  - On-premise datacenter

### Standby Site
- Secondary environment with reduced capacity
- Maintained for service continuity
- Deployed in a **separate cloud region** or **physical datacenter**

```text
┌───────────────┐        Secure Connectivity      ┌───────────────┐
│  Primary Site │  ◀────────────────────────────▶ │  Standby Site │
│               │                                 │               │
│ Hub           │                                 │ Hub           │
│ PM4ML         │                                 │ PM4ML         │
│ Tazama        │                                 │ Tazama        │
└───────────────┘                                 └───────────────┘
```

Both sites share the same logical architecture. Differences are limited to **capacity sizing**, not functionality.

---

## 3. Platform Components

### 3.1 Mojaloop Hub

* Central transaction switching platform
* Enables interoperability between participants
* Deployed as containerized services
* Acts as the **trust authority** for DFSP onboarding

---

### 3.2 PM4ML (Payment Manager for Mojaloop)

* DFSP-facing payment management platform
* Can be deployed:

  * Per DFSP, or
  * As a shared service
* Supports both cloud-hosted and on-premise environments
* Establishes business trust with the Hub

---

### 3.3 Tazama

* Transaction monitoring and fraud detection platform
* Consumes transaction events for analysis
* Can be deployed centrally or regionally
* Operates independently of transaction processing paths

---

## 4. Infrastructure Architecture

### Runtime Platform

* Kubernetes used as the common orchestration layer
* Consistent deployment patterns across environments

### Networking

* Private, encrypted communication between platforms
* Hybrid connectivity supported for cloud ↔ on-premise scenarios

### Traffic Management

* Controlled ingress and egress for platform services
* Clear separation between internal and external traffic paths

### Data Services

* Databases and messaging systems sized per expected throughput
* Logical isolation maintained between platforms

---

## 5. Security Architecture (High Level)

```text
                     ┌─────────┐
                     │ Tazama  │
                     └─────────┘
                      ▲      ▲
                      │      │
        Transaction   │      │  Transaction
        Events        │      │  Events
                      │      │
┌────────┐   HTTPS + Security Stack   ┌─────────┐
│  Hub   │  ◀──────────────────────▶  │  PM4ML  │
└────────┘  mTLS + OAuth2 + JWS + ILP └─────────┘
```

### Security Principles

* Encrypted communication for all inter-platform traffic
* Strong platform identity and authentication
* Separation of **infrastructure trust** and **business trust**
* Centralized secrets and key management

> Detailed mTLS and JWS mechanisms are documented separately.

---

## 6. Availability and Resilience

* High availability within each site
* No active-active transaction processing across sites
* Standby site maintained in a ready state
* Recovery and failover procedures defined outside this document

---

## 7. Operational Model

* Infrastructure provisioned using automation
* Platform deployments follow GitOps practices
* Centralized monitoring, logging, and alerting
* Consistent operations across cloud and on-premise environments

---

## 8. Scope Clarification

This document covers:

* Platform roles and responsibilities
* Deployment topology
* High-level security and connectivity patterns

This document does **not** include:

* Environment-specific configurations
* Backup and restore procedures
* Incident response or failover runbooks
* Regulatory or compliance controls

---

## 9. Key Takeaways

* A single architecture supports **cloud, on-premise, and hybrid** deployments
* Primary–Standby model balances resilience and cost
* Hub, PM4ML, and Tazama have **clearly separated responsibilities**
* Security is enforced at both transport and application layers

---

## 10. Summary

The Mojaloop Hub, PM4ML, and Tazama platforms are deployed using a consistent, secure, and flexible architecture that supports cloud and on-premise environments while maintaining clear trust boundaries and operational resilience.**